# E2E Testing: Work Creation & Rights Management

This example demonstrates how we implement end-to-end testing for a
real-world user flow that combines **metadata validation** and **rights
(ownership) management**.

The test is written using a Page Object pattern and focuses on
readability, maintainability, and clear mapping to business logic.

It showcases how we verify both:

-   **form validation behavior**
-   **complex domain workflows** such as creator splits and ownership
    shares.

------------------------------------------------------------------------

## Overview

The scenario covers a full lifecycle of a musical work:

-   Creating a new work with validation checks
-   Verifying ISWC validation rules
-   Submitting valid work metadata
-   Adding a creator split
-   Verifying ownership distribution
-   Replacing an existing creator with a new (unregistered) one
-   Handling additional profile data (first name / last name)
-   Verifying split state transitions (**Accepted → Pending**)

------------------------------------------------------------------------

## Test Scenario

1.  User logs into the platform\
2.  Navigates to the **Works** section\
3.  Opens the **Create Work** form\
4.  Enters invalid ISWC values and verifies validation errors:
    -   too short
    -   incorrect format
5.  Submits valid work data\
6.  Verifies that the work is created correctly\
7.  Adds a **creator split** for an existing user\
8.  Verifies:
    -   ownership share
    -   role (function)
    -   acceptance status
    -   associated email
9.  Edits the split and replaces the creator with a **new (unregistered)
    user**
10. Fills in additional profile data (first name, last name, territory)
11. Verifies that:

-   the split is created
-   status becomes **Pending**
-   ownership data is correct

------------------------------------------------------------------------

## E2E Test Implementation

``` python
def test_user_can_create_work_and_manage_creator_splits(self):
    app.auth.login(self, TEST_EMAIL, TEST_PASSWORD)
    app.role.go_to_link(self, "Works")

    # Open work creation form
    app.work.open_create_work_form(self)

    # Validate ISWC: too short
    app.work.fill_iswc(self, "123")
    app.work.submit_create_work(self)
    self.assertEqual(
        app.work.get_iswc_error_message(self),
        "The ISWC number should contain 11 characters",
    )

    # Validate ISWC: wrong prefix
    app.work.fill_iswc(self, "12345678901")
    app.work.submit_create_work(self)
    self.assertEqual(
        app.work.get_iswc_error_message(self),
        'The ISWC number should start with a "T"',
    )

    # Submit valid work data
    app.work.fill_work_form(
        self,
        title="Ukrainian song",
        iswc="T0707705212",
        territory="UA",
        lyrics="some lyrics",
        language="eng",
        explicit="not_required",
    )
    app.work.submit_create_work(self)

    # Verify created work
    self.assertEqual(app.work.get_work_title(self), "Ukrainian song")
    self.assertEqual(app.work.get_work_iswc(self), "ISWC: T0707705212")
    self.assertEqual(app.work.get_work_territory(self), "Territory: Ukraine")
    self.assertEqual(app.work.get_work_language(self), "Lyrics (English)")

    # Add existing creator split
    app.work.add_split(self, "Creator")
    app.work.fill_existing_creator_split(
        self,
        name="Anton",
        function="Lyricist",
        percent="25",
    )
    app.work.submit_split(self)

    # Verify existing creator split
    self.assertEqual(app.work.get_split_status(self, "Anton"), "Accepted")
    self.assertEqual(app.work.get_split_amount(self, "Anton"), "25.00%")
    self.assertEqual(app.work.get_split_name(self, "Anton"), "Anton SecondName")
    self.assertEqual(app.work.get_split_function(self, "Anton"), "Lyricist")
    self.assertEqual(app.work.get_split_email(self, "Anton"), "test@example.com")

    # Replace creator with new (unregistered) user
    app.work.edit_split(self, "SecondName")
    app.work.clear_selected_creator(self)

    app.work.fill_new_creator_split(
        self,
        email="random_email@example.com",
        function="Songwriter",
        percent="25",
    )

    # Verify additional profile form appears
    self.assert_elements_present(".//div[@id='additional-info']")

    # Fill new creator details
    app.work.fill_new_creator_details(
        self,
        first_name="Creator Name",
        last_name="Lastname",
        territory="Canada",
    )
    app.work.submit_split(self)

    # Verify new creator split
    self.assertEqual(app.work.get_split_status(self, "Creator Name"), "Pending")
    self.assertEqual(app.work.get_split_amount(self, "Creator Name"), "25.00%")
    self.assertEqual(
        app.work.get_split_email(self, "Creator Name"),
        "random_email@example.com",
    )
    self.assertEqual(
        app.work.get_split_name(self, "Creator Name"),
        "Creator Name Lastname",
    )
```

------------------------------------------------------------------------

## Page Object Model

``` python
class WorkPage:
    def open_create_work_form(self, sb):
        sb.click(".//a[contains(@href, 'work_new')]")
        sb.wait_for_element_visible("#div_id_title input")

    def fill_iswc(self, sb, value):
        sb.clear("#div_id_iswc input")
        sb.send_keys("#div_id_iswc input", value)

    def fill_work_form(self, sb, title, iswc, territory, lyrics, language, explicit="not_required"):
        sb.clear("#div_id_title input")
        sb.send_keys("#div_id_title input", title)
        sb.clear("#div_id_iswc input")
        sb.send_keys("#div_id_iswc input", iswc)
        sb.click(f"#div_id_territory option[value='{territory}']")
        sb.send_keys("#div_id_lyrics textarea", lyrics)
        sb.click(f"#div_id_language option[value='{language}']")

    def submit_create_work(self, sb):
        sb.click(".//button[@id='add_button']")
        sb.wait_for_element_not_visible(".//span[contains(@class, 'spinner')]", timeout=15)

    def add_split(self, sb, split_type):
        role = "add_creator_split" if split_type == "Creator" else "add_work_contribution"
        sb.click(".//button[contains(text(),'Participant')]")
        sb.click(f".//a[contains(@href, '{role}')]")

    def fill_existing_creator_split(self, sb, name, function, percent):
        sb.click(f".//span[contains(text(), '{name}')]")
        sb.click(f".//*[@id='div_id_creator_function']//option[contains(text(), '{function}')]")
        sb.clear(".//div[@id='div_id_split']//input")
        sb.send_keys(".//div[@id='div_id_split']//input", percent)

    def fill_new_creator_split(self, sb, email, function, percent):
        sb.send_keys(".//span[@class='selection']//input", email)
        sb.click(f".//span[contains(text(), '{email}')]")
        sb.click(f".//*[@id='div_id_creator_function']//option[contains(text(), '{function}')]")
        sb.clear(".//div[@id='div_id_split']//input")
        sb.send_keys(".//div[@id='div_id_split']//input", percent)

    def fill_new_creator_details(self, sb, first_name, last_name, territory):
        sb.send_keys("#id_first_name", first_name)
        sb.send_keys("#id_last_name", last_name)
        sb.click(f"#id_territory option[contains(text(), '{territory}')]")

    def submit_split(self, sb):
        sb.click("input[type='submit']")
        sb.wait_for_element_visible(".//h2[@class='work_name']")

    def edit_split(self, sb, username):
        sb.click(f".//span[contains(text(), '{username}')]//a[contains(@href, 'edit')]")

    def clear_selected_creator(self, sb):
        sb.click(".//span[contains(@class, 'remove')]")

    def get_split_status(self, sb, username):
        return sb.get_text(f".//span[contains(text(), '{username}')]/..//strong[@class='status']")

    def get_split_email(self, sb, username):
        return sb.get_text(f".//span[contains(text(), '{username}')]/..//strong[@class='email']")

    def get_split_amount(self, sb, username):
        return sb.get_text(f".//span[contains(text(), '{username}')]/..//strong[@class='share']")

    def get_split_name(self, sb, username):
        return sb.get_text(f".//span[contains(text(), '{username}')]")

    def get_split_function(self, sb, username):
        return sb.get_text(f".//span[contains(text(), '{username}')]/..//span[@class='function']")
```

------------------------------------------------------------------------

## Key Takeaways

-   Business-readable tests
-   Strong validation coverage
-   Page Object abstraction
-   Real-world ownership workflows
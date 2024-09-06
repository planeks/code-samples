# Custom LinkedIn Connector for Django App

This document provides a detailed guide on integrating LinkedIn login functionality into your Django application using a custom connector.

## Overview

Due to issues with existing libraries like `allauth` and `social-auth`, which may not be fully compatible with the latest LinkedIn API updates, a custom LinkedIn connector was implemented. This connector handles the entire OAuth2 process, including obtaining authorization, exchanging authorization codes for access tokens, and fetching user information.

## Setup and Configuration

### Prerequisites

Ensure you have the following in your Django settings:

- `LINKEDIN_CLIENT_ID`: Your LinkedIn application's client ID.
- `LINKEDIN_CLIENT_SECRET`: Your LinkedIn application's client secret.
- `LINKEDIN_REDIRECT_URL`: The URL LinkedIn will redirect to after authentication.

### Installation

1. Place the `connectors.py` file in your Django app.
2. Create views for Linkedin auth.
3. Update your `urls.py` to include the LinkedIn login views.

### Settings

Add the LinkedIn configuration to your Django settings:

```python
# settings.py
LINKEDIN_CLIENT_ID = 'your-client-id'
LINKEDIN_CLIENT_SECRET = 'your-client-secret'
LINKEDIN_REDIRECT_URL = 'your-callback-view-url'
```

### Views

The views handle the interaction between the user and the LinkedIn OAuth process. You need to create two views: one to initiate the login process and another to handle the callback from LinkedIn.

1. LinkedIn Login View 

This view redirects the user to LinkedIn's authorization page.
```python
def linkedin_login_view(request):
    return LinkedInConnector.login_to_provider()
```
2. LinkedIn Callback View

This view handles the callback from LinkedIn after the user has authorized the application. It processes the authorization code, exchanges it for an access token, fetches user information, and logs the user into your Django application.
```python
def linkedin_login_callback(request):
    if request.method == 'GET':
        try:
            LinkedInConnector.login(request)
            return redirect(reverse('index'))
        except BadRequest as e:
            logger.error(f'LinkedIn login error: {e}')
            return render(request, "error_register_login_failed.html")
    return redirect(reverse("login_register"))
```

### LinkedIn Connector
This class manages the entire authentication process with LinkedIn.

```python
class LinkedInConnector:
    authorize_url = "https://www.linkedin.com/oauth/v2/authorization/?"
    access_token_url = "https://www.linkedin.com/oauth/v2/accessToken"
    profile_url = "https://api.linkedin.com/v2/userinfo"

    @classmethod
    def _bad_request_check(cls, response):
        if response.status_code != 200:
            raise BadRequest

    @classmethod
    def login_to_provider(cls):

        params = {
            "response_type": "code",
            "client_id": settings.LINKEDIN_CLIENT_ID,
            "redirect_uri": settings.LINKEDIN_REDIRECT_URL,
            "state": "ASDFASDFASDF",
            "scope": "profile,email,openid"
        }

        return redirect(cls.authorize_url + urlencode(params))

    @classmethod
    def get_authorization_code(self, request):
        return request.GET.get("code")

    @classmethod
    def get_access_token(cls, authorization_code):

        data = {
            "code": authorization_code,
            "client_id": settings.LINKEDIN_CLIENT_ID,
            "client_secret": settings.LINKEDIN_CLIENT_SECRET,
            "redirect_uri": settings.LINKEDIN_REDIRECT_URL,
            "grant_type": "authorization_code",
        }

        response = requests.post(cls.access_token_url, data=data)
        cls._bad_request_check(response)
        return response.json().get("access_token")

    @classmethod
    def get_userinfo(cls, access_token):

        headers = {"Authorization": f"Bearer {access_token}"}
        userinfo_response = requests.get(cls.profile_url, headers=headers)
        cls._bad_request_check(userinfo_response)
        return userinfo_response.json()

    @classmethod
    def populate_user(cls, userinfo):
        user_email = userinfo.get("email")
        user_name = userinfo.get("name")

        if get_user_model().objects.filter(email=user_email).exists():
            linkedin_user = get_user_model().objects.filter(
                email=user_email
            ).first()
        else:
            linkedin_user = get_user_model().objects.create_user(
                email=user_email,
                name=user_name,
                is_linkedin_user=True
            )

        return linkedin_user

    @classmethod
    def login(cls, request):
        authorization_code = cls.get_authorization_code(request)
        access_token = cls.get_access_token(authorization_code)
        userinfo = cls.get_userinfo(access_token)

        linkedin_user = cls.populate_user(userinfo=userinfo)
        linkedin_user.backend = 'django.contrib.auth.backends.ModelBackend'
        login(request, linkedin_user)
        return linkedin_user
```

### Linkedin developer app

To integrate LinkedIn login functionality into your Django application, you first need to create a LinkedIn Developer Application. This application will provide you with the necessary credentials (Client ID and Client Secret) and configure your application's settings.
1. Sign In to LinkedIn Developer Portal:

   * Go to the LinkedIn Developer Portal and sign in with your LinkedIn account.


2. Create a New Application:

   * Click on "Create App" and fill out the required information such as the app name, company page (if applicable), and app logo.
   * Provide a valid email address and agree to the LinkedIn API terms of use.

   
3. Configure Application Settings:

   * Once the application is created, navigate to the "Auth" tab in your application settings.
   * Set the "OAuth 2.0 Redirect URLs" to the URL where LinkedIn will redirect users after they authenticate. This should match the LINKEDIN_REDIRECT_URL in your Django settings (e.g., https://yourapp.com/linkedin/callback).

   
4. Get Your Credentials:

   * Under the "Auth" tab, you will find your "Client ID" and "Client Secret". Copy these credentials and add them to your Django settings as LINKEDIN_CLIENT_ID and LINKEDIN_CLIENT_SECRET.

   
5. Set Permissions:

   * In the "Products" tab, add the necessary products for your application. You should add "Sign In with LinkedIn using OpenID Connect" product for correct work of the connector.
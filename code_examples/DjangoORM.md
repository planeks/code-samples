# Django ORM

An example of the relatively complex queries for the SaaS dashboard with annotations and `Case` function.

```python
@login_required
def dashboard_view(request, *args, **kwargs):
    """
    Generate view for dashboard.
    """
    onboarding_process = get_onboarding_process(request.user)
    random_content_box = ContentBox.objects.get_random_item()

    invitations = Invitation.objects.filter(
        Q(Q(target__email=request.user.email) & Q(status=PENDING_STATUS)) | Q(initiator=request.user)
    ).annotate(
        relevancy=Case(
            When(status=ACCEPTED_STATUS, then=1),
            When(status=DECLINED_STATUS, then=1),
            When(status=PENDING_STATUS, then=2),
            output_field=IntegerField()
        )
    ).select_related('target').order_by('relevancy')

    requests_sent = []
    pending_requests = set()

    group_sent_requests = {}
    ids_for_remove = set()
    for invitation in invitations:
        if not invitation.parent_content_obj:
            ids_for_remove.add(invitation.id)
        elif invitation.status == PENDING_STATUS and invitation.target.email == request.user.email:
            pending_requests.add(invitation)
        elif invitation.hidden:
            continue
        elif invitation.type == Invitation.InvitationTypes.GROUP_MEMBERSHIP:
            try:
                if invitation.parent_content_obj.id not in group_sent_requests:
                    group_sent_requests[invitation.parent_content_obj.id] = {
                        "group_name": invitation.parent_content_obj.displayartistname.name,
                        "profile_id": invitation.parent_content_obj.displayartistname.artist.id,
                        "requested_members": [],
                        "accepted_members": [],
                        "status": PENDING_STATUS,
                        "ids": [invitation.id]
                    }
                else:
                    group_sent_requests[invitation.parent_content_obj.id]["ids"].append(invitation.id)
            except Group.displayartistname.RelatedObjectDoesNotExist:
                ids_for_remove.add(invitation.id)
                continue

            group_sent_requests[invitation.parent_content_obj.id]['requested_members'].append({
                "avatar": invitation.initiator_content_obj.avatar,
                "full_name": invitation.initiator_content_obj.full_name,
                "email": invitation.initiator_content_obj.unregistered_user.email
            })

            if invitation.target_content_obj:
                group_sent_requests[invitation.parent_content_obj.id]['accepted_members'].append({
                    "avatar": invitation.target_content_obj.avatar,
                    "full_name": invitation.target_content_obj.full_name,
                    "email": invitation.target_content_obj.corresponding_email
                })
                group_sent_requests[invitation.parent_content_obj.id]['status'] = ACCEPTED_STATUS
        else:
            requests_sent.append(invitation)

    # remove broken invitations
    Invitation.objects.filter(id__in=ids_for_remove).delete()

    profile = request.user.profiles.first()
    dans = list(DisplayArtistName.objects.filter(
        Q(artist=profile) | Q(group__artists=profile), status=ACCEPTED_STATUS
    ).values('id', 'name')) if profile else []
    publishers = list(request.user.publishers.values('id', 'name', 'corresponding_email', 'picture'))
    collaborators = list(request.user.collaborators.values('id', 'name', 'corresponding_email', 'picture'))

    sorted_requests = {
        "profiles": [],
        "profiles_agreements": [],
        "dans": [],
        "groups": [],
        "publishers_agreements": [],
        "collaborators": [],
        "profile_transfers": [],
    }
    if request.GET.get('as_table'):
        change_avatar_url(publishers, request.user, static('img/default_avatar_2.png'), False)
        change_avatar_url(collaborators, request.user, static('img/default_avatar_2.png'), False)

        for dan in dans:
            dan['picture'] = profile.avatar
            dan['corresponding_email'] = profile.corresponding_email

        for r in pending_requests:
            if r.type in [
                Invitation.InvitationTypes.CREATOR_SPLIT, Invitation.InvitationTypes.WORK_CONTRIBUTION,
                Invitation.InvitationTypes.OTHER_ARTIST_SPLIT, Invitation.InvitationTypes.REC_CONTRIBUTION
            ]:
                sorted_requests['profiles'].append(r)

            elif r.type == Invitation.InvitationTypes.PUBLISHING_AGREEMENT:
                if isinstance(r.initiator_content_obj, Profile):
                    sorted_requests['profiles_agreements'].append(r)
                else:
                    sorted_requests['publishers_agreements'].append(r)

            elif r.type == Invitation.InvitationTypes.COLLABORATOR_SPLIT:
                sorted_requests['collaborators'].append(r)

            elif r.type == Invitation.InvitationTypes.PROFILE_REQUEST:
                sorted_requests['profile_transfers'].append(r)

            elif r.type == Invitation.InvitationTypes.GROUP_MEMBERSHIP:
                sorted_requests['groups'].append(r)

            else:
                sorted_requests['dans'].append(r)

    return render(request, 'pages/dashboard.html', {
        'onboardingProcess': onboarding_process,
        'randomContentBox': random_content_box,
        'invitations': pending_requests,
        'sent_invitations': requests_sent,
        'group_sent_requests': group_sent_requests,
        'profile': profile,
        'dans': json.dumps(dans),
        'publishers': json.dumps(publishers),
        'collaborators': json.dumps(collaborators),
        'sorted_requests': sorted_requests,
    })

```

An example of the query with annotations, attributes renaming, etc.

```python
def get_s2_artist_splits(recording):
    dans = DisplayArtistName.objects.filter(artistsplits__recording=recording).distinct().annotate(
        split=Coalesce(Sum('artistsplits__split'), 0),
        email=Coalesce('artist__unregistered_user__email', 'artist__corresponding_email'),
        performer_function=F('artistsplits__performer_function'),
        user=F('artist__user'),
    ).values(
        "id", "name", "split", "email", "performer_function", "user", "status"
    )

    formatted_featured_artist_splits = []
    formatted_main_artist_splits = []
    main_artist_ids = set()
    featured_artist_ids = set()
    for dan in dans:
        is_registered = False if not dan['user'] or dan['status'] == PENDING_STATUS else True

        value = dan['id'] if is_registered else dan['email']
        formatted_dan = {
            "name": dan['name'],
            "value": value,
            "locked": True if dan['split'] > 0 else False,
            "title": dan['email']
        }

        if dan['performer_function'] == ArtistSplits.MAIN_ARTIST:
            formatted_main_artist_splits.append(formatted_dan)
            main_artist_ids.add(value)
        else:
            formatted_featured_artist_splits.append(formatted_dan)
            featured_artist_ids.add(value)

    return formatted_main_artist_splits, formatted_featured_artist_splits, main_artist_ids, featured_artist_ids
```

An example of the queries with annotations, `Case` and `When` clauses, and `prefetch_related`.


```python
@login_required
@permission_required('roles.can_view_role', raise_exception=True, fn=objectgetter(Profile, 'profile_id'))
def profile_details_view(request, profile_id):
    from notifications.utils import is_paypal_requested_profile
    profile = get_object_or_404(Profile.objects.prefetch_related(
        'creator_works', 'contributor_works', 'contributor_recordings', 'artist_recordings'
    ), id=profile_id)
    is_paypal_requested = is_paypal_requested_profile(profile)

    creator_works = profile.creator_works.distinct().all()
    contributor_works = profile.contributor_works.distinct().all()
    contributor_recordings = profile.contributor_recordings.distinct().all()
    artist_recordings = profile.artist_recordings.exclude(
        artistsplits__display_artist_name__status__in=[PENDING_STATUS, DECLINED_STATUS]
    ).distinct().all()

    publishing_agreements = PublishingAgreement.objects.filter(profile=profile, is_archived_by_profile=False).annotate(
        work_ids=Case(
            When(
                entire_catalogue=False,
                then=ArrayAgg('works__id', distinct=True, default=Value([]), filter=~Q(works__isnull=True))
            ),
            When(entire_catalogue=True, then=ArrayAgg(
                'profile__creator_splits__work__id', distinct=True, default=Value([]),
                filter=~Q(profile__creator_splits__work__isnull=True)
            )),
            default=[]
        )
    ).all()

    archived_publishing_agreements_count = PublishingAgreement.objects.filter(
        profile=profile, is_archived_by_profile=True
    ).count()

    display_artist_names = DisplayArtistName.objects.filter(
        Q(artist=profile) | Q(group__artists=profile), status=ACCEPTED_STATUS
    ).distinct().annotate(
        rec_ids=ArrayAgg('artistsplits__recording__id', distinct=True, default=Value([]), filter=~Q(artistsplits__recording__id=None))
    ).all()

    return render(request, 'roles/profile_details.html', {
        "profile": profile,
        "creator_works": creator_works,
        "contributor_works": contributor_works,
        "contributor_recordings": contributor_recordings,
        "artist_recordings": artist_recordings,
        "display_artist_names": display_artist_names,
        'is_paypal_requested': is_paypal_requested,
        'publishing_agreements': publishing_agreements,
        'archived_publishing_agreements_count': archived_publishing_agreements_count
    })
```


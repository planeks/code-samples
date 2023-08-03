# Auth0 support

An example of the Auth0 support in Django with custom backend for `social_django`.

_settings.py_

```python
...

INSTALLED_APPS = [
    ...
    'social_django',
    ...
]


AUTHENTICATION_BACKENDS = {
    'accounts.backends.Auth0',
    'django.contrib.auth.backends.ModelBackend'
}

SOCIAL_AUTH_REDIRECT_IS_HTTPS = True
SOCIAL_AUTH_TRAILING_SLASH = True
SOCIAL_AUTH_AUTH0_DOMAIN = config('SOCIAL_AUTH_AUTH0_DOMAIN', default='')
SOCIAL_AUTH_AUTH0_KEY = config('SOCIAL_AUTH_AUTH0_KEY', default='')
SOCIAL_AUTH_AUTH0_SECRET = config('SOCIAL_AUTH_AUTH0_SECRET', default='')
SOCIAL_AUTH_AUTH0_SCOPE = [
    'openid',
    'profile',
    'email',
]

...
```

_backends.py_

```python
from urllib import request
from jose import jwt
from social_core.backends.oauth import BaseOAuth2


class Auth0(BaseOAuth2):
    """Auth0 OAuth authentication backend"""
    name = 'auth0'
    SCOPE_SEPARATOR = ' '
    ACCESS_TOKEN_METHOD = 'POST'
    REDIRECT_STATE = False
    EXTRA_DATA = [
        ('picture', 'picture'),
        ('email', 'email')
    ]

    def authorization_url(self):
        return 'https://' + self.setting('DOMAIN') + '/authorize/'

    def access_token_url(self):
        return 'https://' + self.setting('DOMAIN') + '/oauth/token/'

    def get_user_id(self, details, response):
        """Return current user id."""
        return details['user_id']

    def get_user_details(self, response):
        # Obtain JWT and the keys to validate the signature
        id_token = response.get('id_token')
        jwks = request.urlopen('https://' + self.setting('DOMAIN') + '/.well-known/jwks.json')
        issuer = 'https://' + self.setting('DOMAIN') + '/'
        audience = self.setting('KEY')  # CLIENT_ID
        payload = jwt.decode(id_token, jwks.read(), algorithms=['RS256'], audience=audience, issuer=issuer)

        return {'username': payload['nickname'],
                'first_name': payload['name'],
                'picture': payload['picture'],
                'user_id': payload['sub'],
                'email': payload['email']}
```

_views.py_

```python
from django.shortcuts import render, redirect
from django.conf import settings
from django.contrib.auth import logout


def login_view(request):
    return redirect('/login/auth0/')  # Redirect to the standard social_django endpoint


def logout_view(request):
    _next = request.GET.get('next')
    _user = request.user
    logout(request)
    return_to = urlencode({'returnTo': settings.SITE_URL})
    logout_url = 'https://%s/v2/logout?client_id=%s&%s' % (
        settings.SOCIAL_AUTH_AUTH0_DOMAIN,
        settings.SOCIAL_AUTH_AUTH0_KEY,
        return_to)
    return redirect(logout_url)
```

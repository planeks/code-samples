# Django channels

## asgi.py

```python
import os
from django.urls import path
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from turbo.consumers import TurboStreamsConsumer


os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

django_asgi_app = get_asgi_application()
application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AuthMiddlewareStack(
        URLRouter(
            [
                path('ws/', TurboStreamsConsumer.as_asgi()),
            ]
        )
    ),
})
```

## Custom consumer

> In that project the library `djangochannelsrestframework` has been used for implementing API endpoint with websockets support.


`asgi.py`:

```python
import os

from channels.routing import ProtocolTypeRouter, URLRouter
from django.core.asgi import get_asgi_application

from api.middleware import TokenAuthMiddleware, SttProviderMiddleware, LanguageCodeMiddleware
from api.routing import ws_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

django_asgi_app = get_asgi_application()
application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': LanguageCodeMiddleware(
        SttProviderMiddleware(
            TokenAuthMiddleware(
                URLRouter(ws_urlpatterns)
            )
        )
    ),
})
```

`middleware.py`:

```python
from channels.db import database_sync_to_async
from channels.middleware import BaseMiddleware
from django.db import close_old_connections
from rest_framework.authtoken.models import Token


@database_sync_to_async
def get_user(token):
    try:
        user = Token.objects.get(key=token).user
    except Token.DoesNotExist:
        user = None
    return user


class TokenAuthMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        close_old_connections()
        try:
            token_key = (dict((x.split('=') for x in scope['query_string'].decode().split("&")))).get('token', None)
        except ValueError:
            token_key = None
        scope['user'] = await get_user(token_key)
        return await super().__call__(scope, receive, send)


class SttProviderMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        try:
            provider = (dict((x.split('=') for x in scope['query_string'].decode().split("&")))).get('stt_provider', None)
        except ValueError:
            provider = None
        scope['stt_provider'] = provider
        return await super().__call__(scope, receive, send)


class LanguageCodeMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        try:
            language = (dict((x.split('=') for x in scope['query_string'].decode().split("&")))).get('language', None)
        except ValueError:
            language = None
        scope['language'] = language
        return await super().__call__(scope, receive, send)
```

`routing.py`:

```python
from django.urls import re_path

from api.consumers import TranscriptionConsumer

ws_urlpatterns = [
    re_path(r'ws/transcription/', TranscriptionConsumer.as_asgi()),
]
```

`consumers.py`:

```python
import base64
import json

from djangochannelsrestframework.consumers import AsyncAPIConsumer
from djangochannelsrestframework.decorators import action
from djangochannelsrestframework.permissions import IsAuthenticated
from rest_framework import status

from api.stt_providers import AVAILABLE_PROVIDERS
from api.stt_providers.google.google_stt import GoogleSttProvider


class TranscriptionConsumer(AsyncAPIConsumer):
    permission_classes = [IsAuthenticated]

    async def websocket_connect(self, message):
        stt_provider = self.scope.get('stt_provider')
        language = self.scope.get('language', 'en-US')
        self.provider = AVAILABLE_PROVIDERS.get(stt_provider, GoogleSttProvider)(language)
        await super().websocket_connect(message)

    @action()
    async def transcribe(self, voice_stream, **kwargs):
        bytes_stream = base64.b64decode(voice_stream)
        await self.provider.transcribe_stream(bytes_stream)
        transcript = self.provider.transcript
        return json.dumps({'transcript': transcript}), status.HTTP_200_OK
```

The piece of the client code:

```javascript
const ws = new WebSocket(`ws://127.0.0.1:8000/ws/transcription/?token=${token}&stt_provider=google&language=en-US`);

ws.onopen = () => {
    mediaRecorder.addEventListener('dataavailable', async (event) => {
        if (event.data.size > 0 && ws.readyState === 1) {
            const reader = new FileReader();
            reader.readAsDataURL(event.data);

            reader.onloadend = function () {
                let base64string = reader.result;
                base64string = base64string.replace('data:audio/webm;codecs=opus;base64,', '')

                ws.send(JSON.stringify({
                    action: "transcribe",
                    request_id: new Date().getTime(),
                    voice_stream: base64string
                }));
            }
        }
    })
    mediaRecorder.start(250);
}

ws.onmessage = (message) => {
    const {data} = JSON.parse(message.data);
    if (data) {
        const response = JSON.parse(data).transcript;
        console.log(`Received response ${response}`)
        document.querySelector('#response').innerText = response;
    }
}
```

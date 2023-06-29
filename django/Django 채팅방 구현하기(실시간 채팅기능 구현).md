# Django 채팅방 구현하기
##### mac 환경에서 구현되었습니다.
## 1. 실시간 채팅기능 구현하기
> 공식문서를 참고하였습니다. https://channels.readthedocs.io/en/latest/index.html

실시간 채팅기능을 구현하기 위해서 Daphne와 Channels를 설치해줍니다.
```bash
    pip install -U channels
    pip install daphne
```
``` python
    INSTALLED_APPS = [
    'daphne',
    'channels',
    ...
    ]
```
<details>
<summary>왜 daphne를 쓰는가?</summary>

Django channels는 Http요청을 uWSGI 프로토콜로, WebSocket요청을 ASGI프로토콜로 받는데, Daphne는 이를 위해 개발된 HTTP, HTTP2, WS 프로토콜 서버로서, HTTP와 WS 요청을 받아들여서 자동으로 어떤 프로토콜로 처리해야 할지 스스로 결정합니다.
> 참조 : https://victorydntmd.tistory.com/265
</details>

settings.py에도 `ASGI_APPLICATION = 'config.asgi.application'`를 추가해줍니다.\
\
이제 채팅을 담당할 django-app을 만들어 줍니다.
```bash
python manage.py startapp chatapp
```

채팅은 여러 사람들이 사용할 수 있어야 하기 때문에 channel layel을 이용해서 여러 개의 consumer들이 서로 간에 커뮤니케이션을 하게 해야 합니다.\
이를 위해  redius를 백업 저장소로 하는 채널 계층을 생성해줍니다.

``` bash
pip install channels_redis
```
```python
# settings.py
# Channels
ASGI_APPLICATION = "mysite.asgi.application"
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

### consumer이란?
consumer(소비자)는 WebSocket 연결을 처리하고 관련된 이벤트를 처리하는 클래스로 다음과 같은 처리를 담당합니다.
1. WebSocket 연결 처리
    - 클라이언트와 서버 간의 실시간 양방향 통신을 제공하며 WebSocket 프로토콜을 이해하고, 클라이언트로부터의 메시지를 받아들이고 처리하는 역할을 수행
2. 이벤트 처리
   - 클라이언트와의 연결 이벤트(접속, 접속 해제 등) 및 사용자 정의 이벤트를 처리
3. 비동기 처리

코드 설명 : https://dbza.tistory.com/entry/django-channels-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC-consumer-%EB%8D%B0%EC%9D%B4%ED%84%B0-%ED%9D%90%EB%A6%84

consumers.py 파일을 통해 counsumer(사용자)를 정의해줍니다.
``` python
# chatapp/consumers.py
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        # Join room group
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)

        await self.accept()

    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    # Receive message from WebSocket
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        # Send message to room group
        await self.channel_layer.group_send(
            self.room_group_name, {"type": "chat.message", "message": message}
        )

    # Receive message from room group
    async def chat_message(self, event):
        message = event["message"]

        # Send message to WebSocket
        await self.send(text_data=json.dumps({"message": message}))
```
이제 Channels에게서 HTTP요청을 받았을 때 실행할 코드를 Channels에 알려주도록 라우팅 구성을 해줍니다.\
Channels는 웹소켓 연결을 처리하기 위해 URL과 연결된 소비자(Consumer)를 매핑해야 하며, 이 작업은 routing.py에서 담당합니다.

```python
# chatapp/routing.py
from django.urls import re_path

from . import consumers

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", consumers.ChatConsumer.as_asgi()),
]
```

`(project명)/asgi.py`다음과 같은 위치에 asgi.py파일을 만들고 다음과 같이 작성합니다.

``` python
import os

from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
from django.core.asgi import get_asgi_application

from chatapp.routing import websocket_urlpatterns

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")
django_asgi_app = get_asgi_application()

import chatapp.routing

application = ProtocolTypeRouter(
    {
        "http": django_asgi_app,
        "websocket": AllowedHostsOriginValidator(
            AuthMiddlewareStack(URLRouter(websocket_urlpatterns))
        ),
    }
)
```

우선 채팅이 성공적으로 작동하는지 확인하기 위해 임의로 index.html과 room.html을 생성합니다.
<details>
<summary>index.html</summary>

``` html
<!-- chatapp/templates/chat/index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Chat Rooms</title>
</head>
<body>
    What chat room would you like to enter?<br>
    <input id="room-name-input" type="text" size="100"><br>
    <input id="room-name-submit" type="button" value="Enter">

    <script>
        document.querySelector('#room-name-input').focus();
        document.querySelector('#room-name-input').onkeyup = function(e) {
            if (e.key === 'Enter') {  // enter, return
                document.querySelector('#room-name-submit').click();
            }
        };

        document.querySelector('#room-name-submit').onclick = function(e) {
            var roomName = document.querySelector('#room-name-input').value;
            window.location.pathname = '/chat/' + roomName + '/';
        };
    </script>
</body>
</html>
```

</details>

<details>
<summary>room.html</summary>

``` html
<!-- chatapp/templates/chat/room.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Chat Room</title>
</head>
<body>
    <textarea id="chat-log" cols="100" rows="20"></textarea><br>
    <input id="chat-message-input" type="text" size="100"><br>
    <input id="chat-message-submit" type="button" value="Send">
    {{ room_name|json_script:"room-name" }}
    <script>
        const roomName = JSON.parse(document.getElementById('room-name').textContent);

        const chatSocket = new WebSocket(
            'ws://'
            + window.location.host
            + '/ws/chat/'
            + roomName
            + '/'
        );

        chatSocket.onmessage = function(e) {
            const data = JSON.parse(e.data);
            document.querySelector('#chat-log').value += (data.message + '\n');
        };

        chatSocket.onclose = function(e) {
            console.error('Chat socket closed unexpectedly');
        };

        document.querySelector('#chat-message-input').focus();
        document.querySelector('#chat-message-input').onkeyup = function(e) {
            if (e.key === 'Enter') {  // enter, return
                document.querySelector('#chat-message-submit').click();
            }
        };

        document.querySelector('#chat-message-submit').onclick = function(e) {
            const messageInputDom = document.querySelector('#chat-message-input');
            const message = messageInputDom.value;
            chatSocket.send(JSON.stringify({
                'message': message
            }));
            messageInputDom.value = '';
        };
    </script>
</body>
</html>
```

</details>

``` python
# chatapp/views.py
def index(request):
    return render(request, "chat/index.html")
def room(request, room_name):
    return render(request, "chat/room.html", {"room_name": room_name})
```

``` python
# chatapp/urls.py
urlpatterns = [
    path("", views.index, name="index"),
    path("<str:room_name>/", views.room, name="room"),
]
```
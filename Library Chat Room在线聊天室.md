# **Library Chat Room在线聊天室**

# 1 django+ Channels项目配置

## 1.1 Pycharm创建Django项目

```
python manage startapp chat
```

## 1.2 安装Channels

```
pip install channels
```

## 1.3 配置Channels、websocket

在setting.py注册app

```python
#chatroom/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'channels',
    'chat.apps.ChatConfig',
]
```

在setting中添加asgi_application

```python
#chatroom/settings.py
ASGI_APPLICATION = "chatroom.asgi.application"
```

修改asgi文件

```python
#chatroom/asgi.py
import os

from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from . import routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'chatroom.settings')

#application = get_asgi_application()

application = ProtocolTypeRouter(
    {
        "http": get_asgi_application(),
        "websocket": URLRouter(routing.websocket_urlpatterns),
    }
)
```

在setting同级目录创建routing.py文件

```python
#chatroom/routing.py
from django.urls import re_path
from chat import consumers


websocket_urlpatterns = [
    re_path(r'ws/(?P<group>\w+)/$', consumers.ChatConsumer.as_asgi())
]
```

在chat目录下创建consumers.py文件

```python
#chat/consumers.py
from channels.generic.websocket import WebsocketConsumer
from channels.exceptions import StopConsumer


class ChatConsumer(WebsocketConsumer):
  	#有客户端请求时
    def websocket_connect(self, message):
      	#握手
        self.accept()
    
    #收到客户端发送的消息时
    def websocket_receive(self, message):
        self.send("收到消息")
        #self.close
        #服务端主动断开连接
    
		#当客户端主动断开连接时
    def websocket_disconnect(self, message):
        raise StopConsumer()
```

至少要有以上三个方法

------

- http：wsgi

  ```python
  urls.py
  views.py
  ```

- websocket：asgi｜routings-->consumers.py

  ```python
  routings.py
  consumers.py
  ```

# 2 基本收发功能

## 2.1 用户访问界面

路由

```python
#chatroom/urls.py
from django.contrib import admin
from django.urls import path

from chat import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('index/', views.index),
]

```

视图

```python
#chat/views.py
from django.shortcuts import render


def index(request):
    return render(request, 'index.html')

```

模版

```html
//chat/templates/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <style>
        .message {
            height: 300px ;
            border: 1px solid lightslategray;
            width: 100%;
            text-align: center;
            background: lightslategray;
        }
    </style>
</head>
<body>
<div class="message" id="message" ></div>
<div>
    <input type="text" placeholder="请输入" id="txt">
    <input type="button" value="查询" onclick="sendMessage()">
</div>
</body>
</html>
```

## 2.2 websocket连接和收发

客户端向服务端发消息

```html
//chat/templates/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <style>
        .message {
            height: 300px ;
            border: 1px solid lightslategray;
            width: 100%;
            text-align: center;
            background: lightslategray;
        }
    </style>
</head>
<body>
<div class="message" id="message" ></div>
<div>
    <input type="text" placeholder="请输入" id="txt">
    <input type="button" value="查询" onclick="sendMessage()">
    <input type="button" value="关闭" onclick="closeConn()">

</div>
<script>
    //发起websocket连接
    socket = new WebSocket("ws://127.0.0.1:8000/ws/123/");

    // 当websocket接收到消息时触发
    socket.onmessage = function (event){
        let tag = document.createElement("div");
        tag.innerText = event.data;
        document.getElementById("message").appendChild(tag);
    }
  
  	//当创建连接成功时触发
  	socket.onopen = function (event){
        let tag = document.createElement("div");
        tag.innerText = "[连接成功啦]";
        document.getElementById("message").appendChild(tag);

    }
  
  	//服务端主动断开连接时触发
  	socket.onclose = function (event){
        let tag = document.createElement("div");
        tag.innerText = "[连接断开啦]";
        document.getElementById("message").appendChild(tag);
    }

    //客户端发送
  	function sendMessage() {
        let tag = document.getElementById("txt");
        socket.send(tag.value);
    }
  	
  	//客户端主动断开
  	function closeConn(){
        socket.close();
    }
</script>
</body>
</html>
```

服务端接收数据

```python
#chat/consumers.py
from channels.generic.websocket import WebsocketConsumer
from channels.exceptions import StopConsumer


class ChatConsumer(WebsocketConsumer):
  	#有客户端请求时
    def websocket_connect(self, message):
      	print("建立连接")
      	#握手
        self.accept()
    
    #收到客户端发送的消息时
    def websocket_receive(self, message):
        #self.close()
        #服务端主动断开连接
        text = message["text"]
        res = "{}，真好".format(text)
        self.send(res)
         
		#当客户端主动断开连接时
    def websocket_disconnect(self, message):
      	print("断开连接")
        raise StopConsumer()
```

------

基于django实现websocket

但只能实现单点处理

# 3 群聊功能实现（channel layers内存）

列表实现思路：全局变量列表存储self

## 3.1 配置

```python
#chatroom/settings.py
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer",
    }
}
```

## 3.2 同步channel layers

同时实现“房间号”

```python
#chat/consumers.py
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer
from channels.exceptions import StopConsumer


class ChatConsumer(WebsocketConsumer):
  	#有客户端请求时
    def websocket_connect(self, message):
      	#获取群号
        group = self.scope['url_route']['kwargs'].get("group")
        #将客户端建立的连接加入到内存当中
        async_to_sync(self.channel_layer.group_add)(group, self.channel_name)
        # 接受客户度端连接
        self.accept()
        print("建立连接")
        # 给客户端主动发送消息

    #收到客户端发送的消息时
    def websocket_receive(self, message):
      	#获取群号
        group = self.scope['url_route']['kwargs'].get("group")
        #给组里每一个客户端，执行f1方法
        async_to_sync(self.channel_layer.group_send)(group, {
            'type': 'f1',
            'message': message
        })

    #f1方法中执行想执行的功能
    def f1(self, event):
        text = event['message']['text']
        if text == "llm" or text == "梁洛铭":
            res = "{}是帅b".format(text)
        else:
            res = "{}是傻b".format(text)
        self.send(res)

    #当客户端主动断开连接时
    def websocket_disconnect(self, message):
      	#获取群号
        group = self.scope['url_route']['kwargs'].get("group")
        #将当前客户端移除当前组
        async_to_sync(self.channel_layer.group_discard)(group, self.channel_name)
        print("客户端断开连接")
        raise StopConsumer()

```

## 3.3 房间号

```javascript
//chat/templates/index.html 
socket = new WebSocket("ws://127.0.0.1:8000/ws/{{ room_num }}/");

```

```python
#chat/views.py
from django.shortcuts import render


def index(request):
    room_num = request.GET.get("num")
    return render(request, 'index.html', {"room_num": room_num})

```


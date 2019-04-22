# Django Channels Tutorial

</br>

[참고: Django Channels 공식문서](https://channels.readthedocs.io/en/latest/index.html#django-channels)

간단한 채팅 구현 튜토리얼로 두 개의 페이지가 구성됩니다.

- index 페이지 - 채팅방 이름 입력하고, 입장할 수 있습니다.
- room 페이지 - WebSocket을 통해 Django 서버와 통신하여 채팅 목록 확인

본 튜토리얼은 아래의 환경으로 구성됩니다.

- Python 3.7.3
- Django 2.2
- Channels 2.2

</br>

**Channels 2.0 이상 부터는 Python 3.5 버전 이상, Django 1.11 버전 이상을 지원합니다.**

</br>

---

</br>

## Part 1: 기본 설정하기

### 프로젝트 생성

```shell
$ django-admin startproject mysite
```

위 명령어를 입력하면 아래와 같은 구조의 Django 프로젝트가 생성됩니다.

```
mysite/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        wsgi.py
```

편의를 위해 settings.py가 위치하는 `mysite/mysite`는 `mysite/config`로 변경합니다. 

`mysite/mysite`를 `mysite/config`로 변경할 경우 반드시 settings.py, wsgi.py에서 참조하는 경로를 바꿔주셔야 합니다.

- settigns.py

```python
# 기존
ROOT_URLCONF = 'mysite/urls'
# 변경
ROOT_URLCONF = 'config/urls'
```

- wsgi.py

```python
# 기존
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')

# 변경
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
```

- `mysite/mysite` 변경 후

```
mysite/
    manage.py
    config/
        __init__.py
        settings.py
        urls.py
        wsgi.py
```

</br>

### Channels 설치 및 설정

##### PyPI를 이용하여 channels 설치

```python
pip install -U channels
```

##### INSTALLED_APPS에 적용

```python
INSTALLED_APPS = (
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.sites',
    ...
    'channels',
)
```

##### routing.py 생성

```python
# 프로젝트(source_root) 디렉터리/routing.py

application = ProtocolTypeRouter({
    # 초기 세팅 시 비워두며 http 요청이 들어오면 Django가 추가합니다.
})
```

##### ASGI 설정

```python
ASGI_APPLICATION = "{프로젝트(source_root) 폴더명}.routing.application"
```

##### Channels 정상 설치 확인

터미널에 아래 명령어를 입력하여 Channels가 정상적으로 설치되어 있는지 확인합니다.

`python3 -c 'import channels; print(channels.__version__)'`

- 정상적으로 설치되지 않은 경우 아래와 같이 channles 모듈을 찾을 수 없다는 오류가 표시됩니다.

```shell
>>> python3 -c 'import channels; print(channels.__version__)'

Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'channels'
```

- 정상적으로 설치된 경우 아래와 같이 설치된 버전이 표시됩니다.

```shell
>>> python3 -c 'import channels; print(channels.__version__)'

2.2.0
```

</br>

### 채팅 앱 생성

채팅 모델 및 뷰 등을 구성할 채팅 앱을 생성합니다.

`python3 manange.py chat`

채팅 앱을 생성하면 `mysite` 디렉터리 안에 아래와 같은 `chat` 디렉터리가 생성됩니다.

```
chat/
    __init__.py
    admin.py
    apps.py
    migrations/
        __init__.py
    models.py
    tests.py
    views.py
```

이 튜토리얼에서는 `views.py`와 `__init__.py`만 사용되므로 나머지 파일들은 아래 디렉터리 구조와 같이 삭제하셔도 좋습니다.

```
chat/
    __init__.py
    views.py
```

앱을 추가했으므로, `settings.py`에 부분에 추가된 앱을 등록합니다.

```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # add
    'channels',
    'chat',
]
```

</br>

### index 페이지 구성

본 튜토리얼 처음에 나열했던 두 가지 페이지 중 첫 번째로 index 페이지를 구성합니다.</br>해당 페이지에서는 참여할 채팅방의 이름을 입력고, 입장할 수 있습니다.

#### 템플릿 파일 작성

먼저, 템플릿 파일을 보관할 디렉터리를 생성하고 그 안에 index 페이지의 HTML 파일을 생성합니다.

```
chat/
    __init__.py
    templates/
        chat/
            index.html
    views.py
```

생성한 HTML 파일을 수정합니다.

```html
<!-- chat/templates/chat/index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Chat Rooms</title>
</head>
<body>
    입장하실 채팅방 이름을 입력해주세요.<br/>
    <input id="room-name-input" type="text" size="100"/><br/>
    <input id="room-name-submit" type="button" value="입장하기"/>

    <script>
        document.querySelector('#room-name-input').focus();
        document.querySelector('#room-name-input').onkeyup = function(e) {
            if (e.keyCode === 13) {  // enter, return
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

#### 뷰 파일 작성

`chat/templates/index.html`을 렌더링 할 `views.py`에 아래 코드를 추가합니다.

```python
# chat/views.py

from django.shortcuts import render

# index 함수가 호출될 때 render() 함수를 이용해서 request 정보와 html파일, context({})를 반환한다.
def index(request):
    return render(request, 'chat/index.html', {})
```

#### URL 설정

해당 뷰를 호출하기 위해 있도록 아래 위치에 `urls.py`를 추가하고 코드를 추가합니다.

```
chat/
    __init__.py
    templates/
        chat/
            index.html
    urls.py
    views.py
```

`urls.py`에 아래 코드를 추가합니다.

```python
# chat/urls.py

from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

마지막으로, django 기본 `urls.py`에 `chat.urls.py`를 연결합니다.

```python
# config/urls.py

from django.urls import include, path
from django.contrib import admin

urlpatterns = [
    path('chat/', include('chat.urls')),
    path('admin/', admin.site.urls),
]
```

</br>

---

</br>

## Part 2: 채팅서버 구현


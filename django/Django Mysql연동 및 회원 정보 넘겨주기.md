# Django Mysql연동 및 회원 정보 넘겨주기

<aside>
💡 mac 환경에서 진행되었으며, 개인 정리용이라서 내용이 난잡할 수 있습니다.

</aside>

# Mysql 연동

장고와 mysql이 연결되기 위해서 필요한 mysqlclient를 설치한다.

```bash
pip install mysqlclient
```

setting.py를 다음과 같이 수정한 뒤, migrate 해준다.

```python
DATABASES={
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '[DB명]',
        'USER': '[사용자명]',
        'PASSWORD': '[암호]',
        'HOST': '[주소]',
        'PORT': '3306',
    }
}
```

성공적으로 진행된다면 DB에 다음과 같은 테이블이 생긴다.

![중간의 device는 따로 생성한 테이블이다.](https://blog.kakaocdn.net/dn/dL1RUN/btr5qs5aFUQ/pcGZnaVYzMYVO6yqHWcRW1/img.png)

중간의 device는 따로 생성한 테이블이다.

## models.py 위치 옮기기

원하는 django app에 넣어줌으로써 models.py를 쓰기 편하게 한다.

![Untitled](https://blog.kakaocdn.net/dn/ehLrVR/btr5vsXyz17/ZKoryIoe6eN5Qn1KcXvKX0/img.png)

database라는 폴더를 만들어 준 뒤 models.py를 옮겨주었다.

database라는 폴더를 인식 할 수 있도록 setting.py를 수정해준다.

![Untitled](https://blog.kakaocdn.net/dn/cV7bHZ/btr5pzcnNyX/W0Z7bKdOQx4QB5uZXI3oL0/img.png)

# 회원 정보 넘기기

회원가입이나 로그인은 다음 강좌를 통해 django에서 제공하는 form을 이용해서 만들었습니다.

[작정하고 장고! Django로 Pinterest 따라만들기 : 바닥부터 배포까지 - 인프런 | 강의](https://www.inflearn.com/course/장고-핀터레스트/dashboard)

- 대충 요약
    
    ```python
    from django.views.generic import CreateView
    from django.contrib.auth.models import User
    from django.contrib.auth.forms import UserCreationForm
    
    class AccountCreateView(CreateView):
        model = User
        form_class = UserCreationForm
        # reverse / reverse_lazy 차이 함수와 클래스가 파이썬에서 불러와지는 방식이 달라서
        # reverse : 함수 / reverse_lazy : 클래스
        success_url = reverse_lazy('accountapp:login')
        template_name = 'accountapp/create.html'
    ```
    
    django에서 기본적으로 제공하는 UserCreationForm을 이용했다.
    
    우리가 쓰고자 하는 외부 테이블로 바꾸기 위해서는 수정이 필요하다.
    

## models.py의 내용을 DB의 내용으로 동기화 하기

- 참고한 글들
    
    [Django](https://docs.djangoproject.com/en/4.1/topics/auth/customizing/)
    
    [Django AttributeError: 'User' object has no attribute 'set_password' but user is not override](https://stackoverflow.com/questions/41333026/django-attributeerror-user-object-has-no-attribute-set-password-but-user-is)
    
    [7.2 DRF 회원가입, 로그인, 로그아웃 복습 TIL](https://velog.io/@kti0940/7.2-TIL)
    
    [django - username에 verbose_name 적용하기](https://kimdoky.github.io/django/2018/11/26/django-username-verbose/)
    
    기타 따로 저장못한 글들…
    

db로 회원정보를 넘기기위해서는 우선 django의 데이터 모델인 models.py를 db에 있는 데이터로 바꿔줘야 할 필요가 있다.

![Untitled](https://blog.kakaocdn.net/dn/yMuHZ/btr5oL5vIv5/1AfKaoScwpKKUYs68K9w6K/img.png)

```bash
python manage.py inspectdb
```

다음 명령어를 입력해주면 현재 db내에 있는 테이블 정보를 터미널에 보여줍니다.

가저온 테이블 정보가 정확하다면 다음 명령어를 통해 models.py를 수정합니다.

```bash
python manage.py inspectdb > models.py
```

![성공적으로 반영된 모습](https://blog.kakaocdn.net/dn/p5c7b/btr5pbbO3Wz/VwX8vnqkXTxNnrWoGZxgt0/img.png)

성공적으로 반영된 모습

이 모델을 그대로 쓰면 좋지만 유저 테이블(모델)과 관련해 이런저런 에러가 발생하기 때문에 수정이 필요하다.

<aside>
💡 다음 명령어들의 정확한 작동원리는 아직 아해하지 못했습니다…

</aside>

기본적으로 유저 모델이 다음과 유사하게 생성되어 있을 것 이다.

```python
class UserData(models.Model):
	id = models.CharField(db_column='ID', primary_key=True, max_length=20)
	password = models.CharField(max_length=100)
	name = models.CharField(max_length=20)
	phone_number = models.CharField(db_column='phone number', max_length=13)

	REQUIRED_FIELDS = []

	class Meta:
	        managed = False
	        db_table = 'user_data'
```

하지만 지금 상태로 회원가입을 시도하자고 하면 다음과 같은 오류가 발생했다.

- ~~ table이 없습니다.
- 'Manager' object has no attribute 'get_by_natural_key’

이를 해결하기 위해서 다음과 같이 코드를 수정해주었다.

```python
from django.contrib.auth.models import UserManager, AbstractUser

class CustomUserManager(UserManager):
    pass

class UserData(AbstractUser):
    objects = CustomUserManager()

    id = models.CharField(db_column='ID', primary_key=True, max_length=20)
    password = models.CharField(max_length=100)
    name = models.CharField(max_length=20)
    phone_number = models.CharField(db_column='phone number', max_length=13)

    first_name = None
    last_name = None
    is_superuser = None
    is_staff = None
    last_login = None
    date_joined = None
    email = None
    username = None

    USERNAME_FIELD = 'id'
    REQUIRED_FIELDS = ['password', 'name', 'phone_number']

    class Meta:
        managed = False
        db_table = 'user_data'
```

왼쪽은 기본 장고 유저 테이블, 우측은 개인적으로 만든 유저 테이블이다.

![Untitled](https://blog.kakaocdn.net/dn/eM5Cxk/btr5oMi2L9S/jbYJVT7lK6FRddB0fIWdKk/img.png)

정보를 회원가입 form을 수정없이 user_data테이블에 보내게된다면 다음과 같은 데이터들 때문에 오류가 난다.

last_login, username, first_name, …..

이를 방지하기 위해 기존에 존재하던 User를 수정해줘야 한다. 이때 AbstactUser를 사용한다.

그런 다음 사용하지 않는 값들을 전부 None으로, USERNAME_FIELD와 REQUIRED_FIELDS를 작성한다.

```python
class UserData(AbstractUser):

...

    first_name = None
    last_name = None
    is_superuser = None
    is_staff = None
    last_login = None
    date_joined = None
    email = None
    username = None

    USERNAME_FIELD = 'id'
    REQUIRED_FIELDS = ['password', 'name', 'phone_number']

....
```

```python
class CustomUserManager(UserManager):
    pass

class UserData(AbstractUser):
    objects = CustomUserManager()
```

다음 UserManger를 상속받는 CustomUserManger를 만들어

model.py가 get_by_natural_key를 가지도록 해주었다.

UserData가 사용할 수 있도록 해준다.

그런 다음 settings.py에 들어가 아무 곳이나 작성해준다.

```python
AUTH_USER_MODEL = '[].UserData'
```

이제 views.py에 들어가서 model을 변경해주면 된다.

```python
class AccountCreateView(CreateView):
    model = [아까 만든 모델]
    form_class = CustomUserCreationForm
    success_url = reverse_lazy('accountapp:login')
    template_name = 'create.html'
```

성공적으로 데이터가 넘어가는 것을 확인할 수 있다.

![Untitled](https://blog.kakaocdn.net/dn/c5XaE9/btr5orzun1B/JkS7bEzjuI0Pv2pj95cXuk/img.png)

![Untitled](https://blog.kakaocdn.net/dn/b5oF6s/btr5oMDmxJL/bMxP5aRH576uKJI9YJmxf1/img.png)

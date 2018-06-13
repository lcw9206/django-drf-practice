# REST Framework Tutorial - 4 



* 튜토리얼 사이트
* 인증 & 권한




## 우리가 만든 API에서는 누구나 데이터를 편집하거나, 삭제하는데 있어 아무 제한이 없다.

* 다음과 같은 사항을 확실히 하기 위해 더 나은 기능들을 추가해보자.
```
* 데이터는 항상 생성자와 연관되어 있다.
* 인증받은 사용자만이 데이터를 생성할 수 있다.
* 작성자만이 수정하거나 삭제할 수 있다.
* 인증받지 않은 사용자는 읽기 전용만 가능하다.
```


## Adding information to our model
* Snippet 모델을 수정하자. 먼저 두 개의 필드를 추가한다.
* 이 필드 중 하나(owner)는 데이터를 만든 사용자를 만든 사람을 나타내는데 표현한다.
* 다른 필드는 HTML의 하이라이트 부분을 저장하는데 사용한다.
```
# snippets/ models.py

owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```
* 또한 모델 저장 시, 하이라이트 부분의 코드를 pygments 라이브러리를 이용해 하이라이트 필드에 저장되도록 해야한다.
* 다음 라이브러리들을 Import하자.
```
# snippets/ models.py

from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```
* 이제는 모델 클래스에 save메서드를 추가할 수 있다.
```
# snippets/ models.py

def save(self, *args, **kwargs):
    """
    Use the `pygments` library to create a highlighted HTML
    representation of the code snippet.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = 'table' if self.linenos else False
    options = {'title': self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

* 위의 작업이 완료되면, 데이터베이스를 업데이트 하자.
* 일반적으로 업데이트 시, migration을 이용하지만, 이번 튜토리얼에서는 데이터베이스를 삭제하고 다시 시작해보자.
```
# terminal

>>> rm -f db.sqlite3
>>> rm -r snippets/migrations
>>> python manage.py makemigrations snippets
>>> python manage.py migrate
```
* API 테스르를 위해 몇 명의 사용자를 생성할 수 있다. 가장 빠른 방법은 createsuperuser를 이용하는 것이다.
```
# terminal

>>> python manage.py createsuperuser
```


## Adding endpoints for our User models
* 이제 사용자를 추가했으니 사용자를 보여주는 API를 추가하는 것이 좋다.
* 새로운 serializer.py에 serializer를 추가하자.
```
# snippets/ serializer.py

from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```
* snippets는 User 모델과 역방항 관계이기 때문에, Modelserializer 클래스를 이용할 때 기본적으로 포함되어 있지 않다.
* 그래서 명시적으로 필드를 지정했다.
* 우리는 사용자와 관련된 읽기 전용 view만 사용하면 되므로, 제네릭 클래스 기반 뷰 중, ListAPIView 및 RetrieveAPIView를 사용한다.
```
# snippets/ views.py

from django.contrib.auth.models import User
from snippets.serializer import UserSerializer

class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
* 마지막으로 이러한 view들을 URL conf에서 참조해 API에 추가해야한다.
```
# snippets/ urls.py

url(r'^users/$', views.UserList.as_view()),
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),
```


## Associating Snippets with Users
* 지금까지 데이터를 생성했지만, 데이터를 생성한 생성자와 연결할 방법이 없었고, 이를 해결하기 위해 perform_create 메서드를 오버라이딩한다.
* 이 메서드는 인스턴스 저장 방법을 관리하고, 들어오는 요청이나 요청된 URL에서 들어온 정보를 처리한다.
* views.py의 SnippetList 클래스에 다음 내용을 추가한다.
```
# snippets/ views.py

def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```
* 이제 serializer의 create메서드는 데이터와 함께 owner 필드가 함께 전달된다.


## Updating our serializer
* 이제 데이터와 이를 만든 사용자가 연결되어 있으므로 SnippetsSerializer를 업데이트해보자.
```
# snippets/ serializer.py

owner = serializers.ReadOnlyField(source='owner.username')
```
* ReadOnlyField는 source 인자를 이용해 특정 필드를 지정할 수 있다.
* 또한, 마침표 표기 방식을 이용해 속성을 탐색할 수도 있다. 마치 Django의 템플릿 언어와 비슷하다.
* 이 필드는 타입이 CharField, BooleanField와 달리 타입이 없으며, 항상 읽기 전용이므로 모델의 인스턴스를 업데이트할 때는 사용 할 수 없다.
* CharField(read_only=True)도 동일한 기능을 수행한다.


## Adding required permissions to views
* 이제 사용자와 데이터가 연결 되었으므로, 인증받은 사용자만이 데이터를 생성/ 업데이트/ 삭제할 수 있다.
* REST 프레임워크에는 주어진 view에 접근할 수 있는 사용자를 제한하는데 사용할 수 있는 많은 클래스를 제공한다.
* IsAuthenticatedOrReadOnly는 인증 된 요청에 읽기 - 쓰기 권한이 부여되고 인증되지 않은 요청에는 읽기 전용 권한이 부여됩니다.
* views.py에 다음 내용을 추가한다.
```
# snippets/ views.py

from rest_framework import permissions
```
* 그리고 SnippetList 클래스와 SnippetDetail 클래스에 다음 속성을 추가한다.
```
# snippets/ views.py

permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```


## Adding login to the Browsable API
* 이 시점에 브라우저에서 API에 접속해본다면 데이터가 생성되지 않는 것을 알 수 있다.
* 이 문제를 해결하기 위해 사용자 로그인 기능이 필요하다.
* 프로젝트 레벨의 urls.py를 수정해 API에 로그인 view를 추가해보자. 
```
# snippets/ urls.py

from django.conf.urls import include

urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls')),
]
```
* 위의 '^api-auth/'은 우리가 사용하고자 하는 URL이다.
* 이제 브라우저로 다시 API에 접근해보면 오른쪽 상단에 Login 링크가 보일 것이다. 앞에서 만든 사용자로 로그인하면 데이터를 생성할 수 있을 것이다.
* 몇 가지 데이터를 생성하고, '/users/'로 이동해보면 해당 사용자가 만든 데이터들이 snippets 필드에 포함된 것을 볼 수 있다.


## Object level permissions
* 데이터들은 누구나 볼 수 있어야 하지만, 만든 사용자만이 업데이트와 삭제를 할 수 있어야 한다.
* 이를 위해 permissions.py 파일을 만들어 사용자 지정 권한을 만든다.
```
# snippets/ permissions.py

from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the snippet.
        return obj.owner == request.user
```
* 이제 SnippetDetail view를 편집해 해당 사용자 지정 권한을 추가한다.
```
# snippets/ views.py

permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly,)
IsOwnerOrReadOnly 클래스도 Import 한다.
from snippets.permissions import IsOwnerOrReadOnly
```
* 이제 브라우저를 열면, 데이터를 만든 사용자에게만 'DELETE', 'PUT' 기능이 나타나는 것을 알 수 있다.


## Authenticating with the API
* API에 대한 권한 설정을 했으므로, 데이터를 편집하려면 인증 절차가 필요하다.
* 우리는 어떤 인증 클래스도 만들지 않았기때문에, 기본값인 SessionAuthentication과 BasicAuthentication이 적용된다.
* 웹 브라우저로 API를 사용할 때, 로그인할 수 있으며, 브라우저 세션은 인증 정보를 제공한다.
* 프로그램 상에서 API를 사용할 때, 인증 정보를 명시적으로 제공해야 한다.
* 인증 없이 데이터를 생성하려고 하면, 다음과 같은 에러를 보인다.
```
# terminal

http POST http://127.0.0.1:8000/snippets/ code="print 123"

{
    "detail": "Authentication credentials were not provided."
}
```
* 사용자 계정과 비밀번호를 포함해 요청하면 성공한다.
```
# terminal

http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print 789"

{
    "id": 1,
    "owner": "admin",
    "title": "foo",
    "code": "print 789",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

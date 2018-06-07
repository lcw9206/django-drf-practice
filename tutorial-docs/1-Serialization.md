# REST Framework Tutorial - 1 : Serialization


* [튜토리얼 사이트](http://www.django-rest-framework.org/)
* REST Framework를 다뤄봅니다.


## 가상환경 생성
```
virtualenv env
source env/bin/activate
```


## 환경세팅
* pygments : 파이썬 기반 문법 하이라이터
```
pip install django
pip install djangorestframework
pip install pygments  
```


## 프로젝트 생성
`django-admin.py startproject tutorial`


## 앱 생성
```
cd tutorial
python manage.py startapp snippets
```


## 모델 생성 및 migrate
```
snippets/ models.py

from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles


LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```


## migration & migrate
```
python manage.py makemigrations snippets
python manage.py migrate
```


## Serializer.py 생성
* snippets 인스턴스를 json과 같은 포맷으로 만들기 위해 Serializer 클래스를 이용한다.
* Serializer 클래스는 Django의 Form과 유사하며 required, max_length 등의 유효성 검사 플래그를 포함한다.
```
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    # code의 style={'base_template': 'textarea.html'} 플래그는 Form 클래스의 widgets.Textarea와 동일하다.
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```


## 직접 만든 Serializer 이용해보기
* 먼저 shell에 접속한다.
`python manage.py shell`

* 필요한 모듈을 import 하고, 인스턴스를 생성한다.
```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print "hello, world"\n')
snippet.save()
```

* 생성된 인스턴스를 파이썬 데이터 타입으로 변환한다.
```
serializer = SnippetSerializer(snippet)
serializer.data
{'id': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}
```

* 변환된 파이썬 데이터 타입의 인스턴스를 json 타입으로 변환하는 것으로 직렬화를 마무리한다.
```
content = JSONRenderer().render(serializer.data)
content
'{"id": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'
```

* 모델 인스턴스가 아닌 쿼리셋을 직렬화하는 방법은 serializer 인수에 many = True 플래그를 추가하면 된다.
```
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
[OrderedDict([('id', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), 
('style', 'friendly')]), OrderedDict([('id', 2), ('title', u''), ('code', u'print "hello, world"\n'), 
('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', u''), 
('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```


## ModelSerializers 사용하기
* 장고가 Form과 ModelForm 클래스를 제공하는 것처럼 REST Framework도 Serializer와 ModelSerializer를 제공한다.
* 앞에서 만든 serializer를 ModelSerializer 클래스를 이용해 리팩토링 해보자.
```
# snippets/ serializers.py

class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```
* serializer의 장점은 serializer 인스턴스의 모든 필드를 표현할 수 있다는 것이다.
* shell에 접속해 아래의 문구를 실행하면 각 필드의 모든 값을 읽어오는 모습을 볼 수 있다.
`>>> python manage.py shell`


```
shell

>>> from snippets.serializers import SnippetSerializer
>>> serializer = SnippetSerializer()
>>> print(repr(serializer))
```
* But! ModelSerializer는 Serializer 클래스의 축소 버전임을 기억하자.
1. 선언된 필드를 자동으로 읽어온다.
2. create, update 메서드가 구현되어 있다.


## Serializer 클래스를 이용하는 view 만들기
* view를 수정하고, urls에 sinnpets를 등록하자.
* snippet_list 메서드는 snippets의 모든 데이터를 보여주거나, 새로 생성한다.
```
# snippets/ views.py

from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

* snippet_detail 메서드는 특정 데이터를 보여주거나, 데이터를 수정 혹은 삭제하는 기능을 갖고있다.
```
@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

* urls에 등록한다.
```
# snippets/ urls.py

from django.conf.urls import url
from snippets import views


urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```
```
# tutorial/ urls.py

from django.conf.urls import url, include


urlpatterns = [
    url(r'^', include('snippets.urls')),
]
```
* Serializer를 이용하는 view를 만들고, snippets 앱을 url에 연결했다.
* json의 내용이 잘못된 경우 500 서버 오류를 보게 될 것이다.


## 웹 API 테스트하기
* 서버를 구동해 데이터를 보도록 하자.
```
>>> python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

* 이제 서버를 가동한 터미널이 아닌 다른 터미널 창에서 테스트를 해보자. curl 혹은 httpie를 이용할 수 있다.
* 지금은 파이썬으로 작성된 httpie를 이용하기 위해 설치하자.
`pip install httpie`

* 아래의 명령어를 이용해 전체 데이터를 조회할 수 있다.
```
http http://127.0.0.1:8000/snippets/
HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```
*혹은 아래의 명령어를 이용해 개별 데이터를 조회할 수 있다.
```
http http://127.0.0.1:8000/snippets/2/
HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

## 이 것으로 첫 번째 튜토리얼이 끝난다.

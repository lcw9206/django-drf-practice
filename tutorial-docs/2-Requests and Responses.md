# REST Framework Tutorial - 2 


* [튜토리얼 사이트](http://www.django-rest-framework.org/)
* 이제 REST Framework의 핵심부를 다뤄보자.


## Request 객체
* REST 프레임워크의 Request 객체는 HttpRequest를 확장하여 조금 더 유연하게 요청을 파싱한다.
* Request 객체의 핵심은 Request.data 속성으로, Request.Post와 비슷하지만 웹 API 작업에 더 적합하다.
```
request.POST  # Form 데이터만 다룬다. 오로지 POST 메서드에서만 사용이 가능하다.
request.data  # 아무 데이터나 다룰 수 있다. POST뿐만 아니라 PUT, PATCH 메서드에서도 사용이 가능하다.
```


## Response 객체
* REST 프레임워크에는 Response 객체도 존재한다.
* 이 객체는 TemplateResponse 타입이며, 렌더링되지 않은 컨텐츠를 불러와 클라이언트가 원하는 컨텐츠 형태로 변환한다.
```
return Response(data)  # 클라이언트가 원하는 형태로 컨텐츠를 렌더링한다.
```


## Status 코드
* 앞에서 만든 View에서 HTTP Status 코드를 사용한다해서 오류를 쉽게 읽을 수 있는 것은 아니다.
* REST 프레임워크에서는 오류에 대해 조금 더 명확한 Status 코드를 제공한다.
* 따라서 숫자로 된 식별자 보다는 문자 형태의 식별자를 사용하는 것이 효율적이다.


## API 뷰 감싸기
* REST 프레임워크는 API view를 쓰는데 2가지 wrapper를 제공한다.
```
CBV에서 사용할 수 있는 APIView 클래스
FBV에서 사용할 수 있는 @api_rest 데코레이터
```
* 이 wrapper들은 Request에 기능을 더하거나, context를 추가해 컨텐츠가 잘 변환되도록 한다.
* 또한 405 Method Not Allowed를 반환하거나, request.data가 깨졌을 때, ParseError를 던지기도 한다.


## Pulling it all together - 한 군데에 모으기
* 위에서 배운 클래스와 데코레이터를 이용해 view를 만들어보자.
```
# snippets / views.py

from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
* view가 간단해지며 Form API와 유사해졌다. 이제 개별 데이터를 담당하는 view를 수정해보자.
```
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```
* request.data는 json 및 다양한 형식을 다룰 수 있다. 또한 응답 객체 또한 우리가 원하는 형태로 렌더링이 가능하다.
* Response의 형태가 명시되어있지 않기 때문이다.


## Adding optional format suffixes to our URLs - URL을 이용해 다른 포맷 제공하기

* 여러 형태를 제공할 수 있는 Response 객체의 장점을 이용하기 위해, API에서도 여러 형태를 제공해야 한다.
* URL을 이용해 다양한 형태를 전달받으려면 다음과 같은 URL을 다룰 수 있어야한다.
```
http://example.com/api/items/4.json
```
* 우선 형태를 다루기 위해 format 키워드를 view에 추가하자.
```
def snippet_list(request, format=None):
def snippet_detail(request, pk, format=None):
```
* 그리고 urls.py에 format_suffix_patterns라는 패턴을 추가로 등록하자.
```
# snippets/ urls.py

from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views


urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```


## How's it looking? - 결과 살펴보기
* 앞에서 했던 것과 동일하게 터미널을 이용해 API를 테스트해보자.
* 잘못된 요청에도 잘 대응하는 것을 볼 수 있으며, 전체 데이터 목록도 받아볼 수 있다.
```
>>> http http://127.0.0.1:8000/snippets/

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
* Accept 헤더를 이용해 Response 객체의 형태도 지정할 수 있다.
```
>>> http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
>>> http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```
*아니면 아래와 같이 인자로 추가한 format을 이용할 수도 있다.
```
>>> http http://127.0.0.1:8000/snippets.json  # JSON suffix
>>> http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
```
* Content-Type 헤더를 이용해 형태를 지정할 수도 있다.
```
# POST using form data

>>> http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```
```
# POST using JSON

>>> http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

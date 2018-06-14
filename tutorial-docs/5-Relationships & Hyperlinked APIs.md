## REST Framework Tutorial - 5 



* [튜토리얼 사이트](http://www.django-rest-framework.org/)
* 관계 및 하이퍼링크된 API


현재 우리의 API 내의 관계는 기본 키(Primary Key)를 이용해 나타내고 있다.
이번에는 하이퍼링크를 이용해 API의 cohesion과 discoverability를 향상 시킬 것이다.


## Creating an endpoint for the root of our API
* 우리는 지금까지 데이터와 사용자 간의 엔드 포인드를 만들었지만, API의 단일 진입점이 없었다.
* 이를 만들기 위해 앞서 소개한 Function-based view와 @api_view 데코레이터를 이용한다.
```
# snippets/ views.py

from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```
* 여기서 두 가지를 알아야 한다. 먼저 정규화된 URL을 반환하기 위해 REST 프레임워크의 reverse 메서드를 사용한다.
* 두번 째, URL 패턴은 나중에 snippets/ urls.py에서 선언될 이름으로 식별된다.


## Creating an endpoint for the highlighted snippets
* 여전히 API에서 간과하고 있는 부분은 데이터의 하이라이트 부분을 볼 수 없다는 것이다.
* 이번에는 다른 API의 엔드 포인트와 달리 JSON 대신 HTML 형태로 나타내겠다. REST 프레임워크는 두 가지 스타일의 HTML 렌더링을 제공한다. 하나는 템플릿을 사용하는 것, 두번 째는 미리 렌더링된 HTML을 사용하는 것이다.
* 하이라이트 view를 생성할 때 고려해야 할 것은 우리가 사용할 수 있는 generic view가 없다는 것이다.
* 우리는 객체(object instance)가 아니라 객체의 속성(a property of obejct instance)을 반환할 것이기 때문이다.
* generic view 대신 기본 클래스를 사용해 .get() 메서드를 구현하자.
```
# snippets/ views.py

from rest_framework import renderers
from rest_framework.response import Response


class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = (renderers.StaticHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```
* 평소처럼 우리가 만든 view를 URL conf에 추가하자.
```
# snippets/ urls.py

url(r'^$', views.api_root),
```
* 그리고 하이라이트된 데이터에 대한 url을 추가한다. 
```
# snippets/ urls.py

url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),
```


## Hyperlinking our API
* 요소들 간의 관계를 다루는 것은 웹 API 설계에서 어려운 측면 중 하나다. 관계를 나타내기 위한 방법은 다양하다.
```
* 기본 키(Primary key) 사용
* 요소 간에 하이퍼링크 사용
* 관계 요소에 unique한 식별 slug 플드 사용
* 관계 요소의 기본 문자열 표현 사용
* 포함된 관계 요소에 대한 표현
* custom 정의 표현
```
* REST 프레임워크는 위의 모든 방법을 지원한다. 정,역방향 관계나, 외래키처럼 사용자화된 관리자에 적용할 수 있다.
* 이번에는 하이퍼 링크 방식을 사용하겠다. 이를 위해 기존 ModelSerializer 대신 HyperlinkedModelSerializer를 사용한다.
* HyperlinkedModelSerializer와 ModelSerializer는 다음과 같은 차이가 있다.
```
* id 필드는 기본적으로 포함되지 않는다.
* HyperlinkedIdentityField를 사용해 URL 필드를 포함한다.
* 관계는 PrimaryKeyRelatedField 대신 HyperlinkedRelatedField를 사용한다.
```
* 하이퍼링크를 사용하기 위해 serializer를 수정한다.
```
# snippets/ serializer.py

class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ('url', 'id', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style')


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ('url', 'id', 'username', 'snippets')
```
* 새롭게 highlight 필드를 추가했다. 이 필드는 url 필드와 같은 타입이며, snippet-detail url 패턴 대신 snippet-highlight url 패턴을 가리킨다.
* 앞에서 URL의 format 접미어로 '.json'을 붙인 것처럼, highlight 필드에는 '.html'을 붙였다.


## Making sure our URL patterns are named
* 하이퍼링크된 API를 사용하려면 URL 패턴의 이름을 지정해야 한다. URL 패턴들을 살펴보자.
```
* API의 최상단은 user-list와 snippet-list를 참조한다.
* 데이터 serializer에는 snippet-highlight를 가리키는 필드가 존재한다.
* 우리의 데이터와 사용자 serializer에는 기본적으로  '{model_name} -detail'을 참조하는 'url'필드가 포함되며, 이 경우 'snippet-detail'과 'user-detail'을 가리킨다.
```
* 모든 이름을 URL conf에 넣었다면 snippets/ urls.py는 다음과 같아야 한다.
```
# snippets/ urls.py

from django.conf.urls import url, include
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views


# API endpoints
urlpatterns = format_suffix_patterns([
    url(r'^$', views.api_root),
    url(r'^snippets/$',
        views.SnippetList.as_view(),
        name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    url(r'^users/$',
        views.UserList.as_view(),
        name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$',
        views.UserDetail.as_view(),
        name='user-detail')
])
```


## Adding pagination
* 사용자 및 데이터는 많은 인스턴스를 반환할 수 있으므로, pagination 후, API가 각 페이지를 단계별로 실행할 수 있도록 해야 한다.
* tutorial/ settings.py를 수정해 pagination의 기본 스타일을 변경할 수 있다.
```
# tutorial/ settings.py

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```
* REST 프레임워크의 모든 설정은 REST_FRAMEWORK라는 딕셔너리에 넣어야 한다. 이 딕셔너리는 사전에 네임 스페이스로 지정되어있다.
* 필요하다면 pagination 스타일을 변경할 수 있지만, 우리는 기본 스타일을 사용하겠다.


## Browsing the API
* 탐색 가능한 API를 열어 링크를 눌러보면, API를 둘러 볼 수 있습니다. 또한 데이터의 highlight 부분을 HTML 형태로 볼 수 있다.

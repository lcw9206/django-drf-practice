## REST Framework Tutorial - 6



* [튜토리얼 사이트](http://www.django-rest-framework.org/)
* Viewsets and routers

REST 프레임워크는 ViewSets란 추상화 클래스를 포함한다. 이는 개발자가 API의 상태 및 상호 작용을 모델링하는 데 집중하고 일반적인 규칙에 따라 URL 구성을 자동으로 처리 할 수 있도록 해준다.
ViewSet 클래스는 read, update 메서드를 제공하지만 get, put 메소드는 제공하지 않는다는 점을 제외하고는 View 클래스와 거의 동일합니다.


## Refactoring to use ViewSets
* 우리가 만든 view를 가져와 view sets으로 리팩토링해보자.
* 첫번 째로 우리가 만든 view인 Userlist와 UserDetail을 하나로 모아 UserViewSet로 리팩토링 해보자.
```
# snippets/ views.py

from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    This viewset automatically provides `list` and `detail` actions.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
* 위에서 사용한 ReadOnlyModelViewSet 클래스는 기본적으로 읽기 전용 옵션을 자동 제공한다.
* 우리는 여전히 기본 view를 사용할 때와 같이 queryset과 serializer_class를 설정하지만, 2개의 클래스에 동일한 정보를 설정할 필요는 없어졌다.
* 다음으로는 SnippetList, SnippetDetail, SnippetHighlight view 클래스들을 대체할 것이다. 세 개의 view를 삭제하고, 한 클래스로 대체할 것이다.
```
# snippet/ views.py

from rest_framework.decorators import action
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    This viewset automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```
* 이번에는 기본으로 read와 write 메서드를 제공하는 ModelViewSet 클래스를 이용했다. 
* 알아둬야할 것은 @action 데코레이터를 이용해 highlight 메서드를 생성한 것이다. 이 데코레이터는 create/ update/ delete 기능에 해당하지 않는 곳에 사용할 수 있다.
* @action 데코레이터를 사용한 기능은 기본적으로 GET 요청에 답한다. 만일 POST 요청에 답하길 원한다면, 메소드 인자를 사용하면 된다.
* 추가 기능의 URL은 메서드 이름에 따라 다르다. 만일 구성을 변경하고 싶다면, 데코레이터에 url_path를 설정하자.


## Binding ViewSets to URLs explicitly
* 핸들러 메서드는 단지 URL 설정과 연결하는 기능만 담당한다. 여기서는 먼저 뷰셋의 뷰들을 명시적으로 살펴보자
* snippets/ urls.py에서 ViewSet 클래스를 concrete view와 연결한다.
```
# snippets/ urls.py

from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```
* ViewSet클래스의 뷰들을HTTP 메서드에 따라 어떻게 실제 뷰와 연결했는지 살펴보세요.
* 이제 실제 뷰와 URL을 연결합니다.
```
# snippets/ urls.py

urlpatterns = format_suffix_patterns([
    url(r'^$', api_root),
    url(r'^snippets/$', snippet_list, name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
    url(r'^users/$', user_list, name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
])
```

## Using Routers
* View 클래스 대신 ViewSet 클래스를 사용했기 때문에, 우리가 직접 URL을 설정할 필요가 없다.
* Router 클래스를 사용하면 규칙에 따라 자동으로 view와 url들이 연결된다. 단지 view를 Router에 등록하면 된다.
* snippets/ urls.py를 수정하자.
```
# snippets/ urls.py

from django.conf.urls import url, include
from rest_framework.routers import DefaultRouter
from snippets import views

# Create a router and register our viewsets with it.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# The API URLs are now determined automatically by the router.
urlpatterns = [
    url(r'^', include(router.urls))
]
```
* ViewSet을 Router에 등록하는 것은 url 패턴을 설정하는 것과 비슷하다. 우리는 URL prefix와 ViewSet을 인자로 사용한다.
* DefaultRouter 클래스는 API의 최상단 뷰를 자동으로 생성해주므로, views 모듈에 있는 api-root 메서드를 삭제했다.

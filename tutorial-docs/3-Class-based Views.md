# REST Framework Tutorial - 3 



* [튜토리얼 사이트](http://www.django-rest-framework.org/)
* CBV를 이용해 API view를 작성한다.




## Rewriting our API using class-based views

* CBV를 기반으로 최상단 View를 재작성해보자.
```
# snippets/ views.py

from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

* 이전 코드와 비슷해 보이지만 HTTP 메서드를 더 잘 구분할 수 있다.
* 위와 마찬가지로 데이터(인스턴스)를 담당하는 view도 수정하자.
```
# snippets/ views.py

class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

* 우리는 CBV를 사용하므로 snippets/ urls.py를 CBV에 맞게 수정한다.
```
# snippets/ urls.py

from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.SnippetList.as_view()),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```



## Using mixins

* CBV는 기능을 손쉽게 조합할 수 있다는 큰 이점을 갖고 있다.
* 지금까지 생성한 create/ retrieve/ update / delete 등의 명령은 모델이 지원하는 API view와 유사하다.
* 그리고 이런 공통된 동작은 REST 프레임워크에서 믹스인 클래스로 구현되어있다.
* 이제 이 믹스인 클래스를 이용해 views.py를 구성해보자.
```
# snippets/ views.py

from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

* GenericAPIView, ListModelMixin. CreateModelMixin를 이용해 view를 구성했다.
* GenericAPIView는 핵심기능을 제공하며, 믹스인 클래스들은 .list(), .create() 기능을 제공한다.
* 위에서 이 기능들을 get과 post 메서드에 적절히 매칭했다. 이제 인스턴스를 담당하는 view도 수정해보자.
```
# snippets/ views.py

class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```
* 위에서도 GenericAPIView를 이용해 핵심기능을 제공하며, 나머지 믹스인들이 .retrieve(), .update(), .destroy() 기능을 제공한다.


## Using generic CBV

* 이미 믹스인 클래스를 이용해 view의 코드를 많이 줄였지만, 더 줄일 수 있다.
* REST 프레임워크는 믹스인 클래스와 연결된 Generic view를 제공하며, 이것을 사용하면 코드를 줄일 수 있다.
```
# snippets/ views.py

from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

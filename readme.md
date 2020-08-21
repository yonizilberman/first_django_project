
# **Ptyhon environment**
```shell
$ python3 -m venv env
$ source env/bin/activate
```
Now that we're create & inside a virtual environment.
we can install our package requirements.
```shell
$ pip install django
$ pip install djangorestframework
$ pip install pygments  # We'll be using this for the code highlighting
```

### Creating a new project to work with.
```shell
$ cd ~ # the path thaa you what the project to be
$ django-admin startproject tutorial
$ cd tutorial
```

### Creating a new app in the project.
```shell
$ python manage.py startapp snippets
```
Now we need to add our new app `snippets` & `rest_framework` to the `INSTALLED_APPS` in the `settings.py` file:
```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
]
```

### Creating a model.
```python
from django.db import models

class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    ...

    class Meta:
        ordering = ['created']


```

### Creating an initial migration for our model.
```bash
$ python manage.py makemigrations snippets
$ python manage.py migrate

// migrate back 
manage.py migrate <app_name> <migrations_file_name>
```

# Django Rest Framework
By installing the `rest_framework` to the `INSTALLED_APPS` in the `settings.py` file we add the DRF to the project.

## Creating a serialization
The first thing we need to get started on our Web API is a way of serializing & deserializing the app instances into `json` representations. Create a file in the `snippet` directory named `serializers.py`.

```
snippet
  |_  migrations
  |_  __init__.py
  |_  models.py
  |_  serializers.py     <-
  |_  apps.py
  |_  admin.py
  |_  tests.py
  |_  views.py
```
We will declaring serializers that work very similar to Django's forms and add the following to the file.
```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True,max_length=100)
    ...

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
        ...
        instance.save()
        return instance
```
### Drop into the Django shell:
```shell
$ python ./manage.py shell
```
For us to use the `model` & `serializers` we will need to import them and see how to use them.
```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```
## Using ModelSerializers:
Our SnippetSerializer class is replicating a lot of information that's also contained in the Snippet model. It would be nice if we could keep our code a bit more concise.
In order to keep our code bit more concise, the DRF includes both Serializer classes & ModelSerializer classes.

Let's refactoring our serializer using the ModelSerializer class. Open the file `snippet/serializers.py` again, and replace the SnippetSerializer class with:
```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']

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
        ...
        instance.save()
        return instance
```
One nice property that serializers have is that you can inspect all the fields in a serializer instance, by printing its representation. Open the Django shell with `python manage.py shell`, then try the following:
```python
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
```
`ModelSerializer` classes don't do anything particularly magical. Simply a shortcut for creating serializer classes:
- An automatically determined set of fields.
- Simple default implementations for the `create()` and `update()` methods.

## Writing regular Django views using our Serializer
Let's see how we can write some API views using our new Serializer class. For the moment we'll just write the views as regular Django views.

Edit the `snippet/views.py` file, and add the following.
```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```
The root of our API is going to be a view that supports listing all the existing snippets, or creating a new snippet.
```python
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
We add the `@csrf_exempt` for a clients that won't have a CSRF token.

Now we just need to wire these views up. Create the `snippets/urls.py` file:
```
snippet
  |_  migrations
  |_  __init__.py
  |_  models.py
  |_  serializers.py
  |_  apps.py
  |_  admin.py
  |_  tests.py
  |_  urls.py           <-
  |_  views.py
```
Wire up the root urlconf, in the `tutorial/urls.py` file, to include our snippet app's URLs.
```python
from django.urls import path, include

urlpatterns = [
    ...
    path('', include('snippets.urls')),
]
```
This code up until now is not caver couple of edge cases:
- Not dealing with properly at the moment.
- Sending malformed `json`, or if a request is made with a method that the view doesn't handle.

Then we'll end up with a 500 "server error" response. Still, this'll do for now.

## Testing our first attempt at a Web API
We can test our API using `curl` (bash command) or `httpie`. Httpie is a user friendly http client that's written in Python.
```bash
$ pip install httpie
```
To run just
```bash
$ http URL
# excemple
$ http://127.0.0.1:8000/snippets/
```
Or you can visit the URLs in a web browser.

## Requests and Responses
The core of REST framework.

### **Requests object:**
REST framework introduces a `Request` object that extends the regular `HttpRequest`, and provides more flexible request parsing. The core functionality of the Request object is the `request.data` for working with Web APIs.

### **Responses objects:**
REST framework also introduces a `Response` object, which is a type of `TemplateResponse` that takes unrendered content and uses content negotiation to determine the correct content type to return to the client.

### **Status codes:**
REST framework provides more explicit identifiers for each status code, such as `HTTP_400_BAD_REQUEST` in the `status` module. It's a good idea to use these throughout rather than using numeric identifiers.

### **Wrapping API views:**
REST framework provides two wrappers you can use to write API views.
- The `@api_view` decorator for working with __function based views__.
- The `APIView` class for working with __class-based views__.

These wrappers provide a few bits of functionality like receive Request instances in your view, and adding context to Response objects so that content negotiation can be performed.

The wrappers also provide behaviour such as returning `405 Method Not Allowed` responses when appropriate, and handling any `ParseError` exceptions that occur when accessing `request.data` with malformed input.

## Adding optional format suffixes to our URLs
Using format suffixes gives us URLs that explicitly refer to a given format, and means our API will be able to handle URLs such as http://example.com/api/items/4.json.

Start by adding a format keyword argument to both of the views, like so.
```python
def snippet_list(request, format=None):
# AND to
def snippet_detail(request, pk, format=None):
```
Now update the `snippets/urls.py` file slightly, to append a set of `format_suffix_patterns` in addition to the existing URLs.
```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```
We don't necessarily need to add these extra url patterns in, but it gives us a simple, clean way of referring to a specific format.

## Rewriting our API using class-based views
We'll start by rewriting the root view as a class-based view. All this involves is a little bit of refactoring of views.py.
```python
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
Nowwe need to refactor our `snippets/urls.py` slightly now that we're using class-based views.
```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```
Now run the development server everything should be working just as before.

## Using mixins
One of the big wins of using class-based views is that it allows us to easily compose reusable bits of behaviour.

The create/retrieve/update/delete operations those bits of common behaviour are implemented in REST framework's mixin classes.

Let's compose the views by using the mixin classes. go to our `views.py` module again.
```python
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
We're building our view using `GenericAPIView` in the class `SnippetList` and `SnippetDetail` with:
- `ListModelMixin` - use `self.list` GET function, get the list for the object.
- `CreateModelMixin` - use `self.create` POST function, create a new object.
- `RetrieveModelMixin` - use `self.retrieve` GET function, get the single object from the list.
- `UpdateModelMixin` - use `self.update` PUT function, put new values in an existing object.
- `DestroyModelMixin` - use `self.destroy` DELETE function, delete an existing object.

## Using generic class-based views
Insted of `from rest_framework import mixins` and then:
```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
```
We can get the same result with just use for the `from rest_framework import generics` the `ListCreateAPIView`:
```python
class SnippetList(generics.ListCreateAPIView):
```
That's pretty concise. We've gotten a huge amount for free, and our code looks like good, clean, idiomatic Django.

## Adding information (restrictions) to our model eho can edit or delete:



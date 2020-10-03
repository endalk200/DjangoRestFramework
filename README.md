
![Tux, the Linux mascot](https://www.django-rest-framework.org/img/logo.png)

# DjangoRestFramework

Django REST framework is a powerful and flexible toolkit for building Web APIs.


Some reasons you might want to use REST framework:

* The `Web browsable` API is a huge usability win for your developers.
* Authentication policies including packages for `OAuth1a` and `OAuth2`.
* Serialization that supports both ORM and non-ORM data sources.
* **Customizable** all the way down - just use regular function-based views if you don't need the more  powerful features.
* Extensive documentation, and great community support.
* Used and trusted by internationally recognized companies including [**Mozilla**](https://mozilla.org), [**Red Hat**](https://redhat.com), [**Heroku**](https://heroku.com), and [**Eventbrite**](https://eventbrite.co.uk/about).

Lets see quick overview of the framework and then we will go deeper.

## Quick Start

To demonstrate lets create a project.

### Requirements

The requirements for this project are:

* Python (3.5, 3.6, 3.7, 3.8, 3.9)
* Django (2.2, 3.0, 3.1)

### Installation

	To follow along you can either create a project from scratch and follow along (recommended) 
	or you can just clone the project from my repo
	
		git clone https://github.com/endalk200/rest_framework_tutorial.git
		cd rest_framework_tutorial
				
		python3 virtualenv env # create a virtual environment (recommended)
		source env/bin/activate # activate your environment

		python3 pip install -r requirements.txt # install python dependencies from requirements.txt
		
		python manage.py migrate # run your migration
		python manage.py runserver
	
	Your good to go. skip the installation guide bellow

Rather than installing your dependencies globally  create a virtual environment  first.

	mkdir rest_framework_tutorial # create a directory for your project
	python3 virtualenv env # If not installed use 'python3 pip install virtualenv'
	source env/bin/activate

	pip install django
	pip install djangorestframework
	pip install pygments # for source syntax source code syntax highlighting
	pip install markdown
	
	
Then create you django project

	django-admin startproject rest_framework_tutorial
	python manage.py startapp snippets
	
We'll need to add our new snippets app and the rest_framework app to `INSTALLED_APPS`. 
Let's edit the tutorial/settings.py file:

	INSTALLED_APPS = [
	 
	    'rest_framework',
	    'snippets.apps.SnippetsConfig',
	]
	
### Create Our Model

Create new django model in your `snippets/models.py` file.

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ['created']
```

After creating the model `migrate`

	python manage.py makemigrations
	python migrate
	
### Creating Serializers For The Model

First we need to provide a way of serializing and deserializing the snippet instances into 
representations such as json. We can do this by declaring serializers that work very similar to Django's forms. Create a folder `API` in your `snippets` folder and then create a new file
`__init__.py` so that django treats the folder as python module. Then create a new file 
called `serializers.py` to store all our serializers in. Add the following to `serializers.py` file

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
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

The first part of the serializer class defines the fields that get serialized/deserialized. 
The `create()` and `update()` methods define how fully fledged instances are created or 
modified when calling `serializer.save()`

A serializer class is very similar to a Django Form class, and includes similar validation flags on the various fields, such as `required`, `max_length` and `default`

### How To Use Serializers

Before we proced lets see how we can use the serializer we just created in python shell

	python manage.py shell
	
execute the following

```python
from snippets.models import Snippet
from snippets.API.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```
we just created two instances of snippets. Lets look at them. 

```python
serializer = SnippetSerializer(snippet)
serializer.data
``` 

At this point we've translated the model instance into Python native datatypes. 
To finalize the serialization process we render the data into json.

```python
content = JSONRenderer().render(serializer.data)
content
# Outputs JSON serialized data
```

Know that we have serialized the data into JSON, we can do the reverse verry easily. In your
python shell add:

```python
import io

stream = io.BytesIO(content) # load the content from above which is JSON
data = JSONParser().parse(stream) # parse the JSON stream to python datatypes
```

Then we can restore the JSON data to new fully populated object

```python
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

### Using Model Serializers

The `SnippetSerializer` we just created is replicating a lot of information we have in our
`Snippet` model. Since rednundant code is not verry concise and maintainable we have to tweak it a little.

Luckly `DjangoRestFramework` provides `ModelSerializer` just like `ModelForm` in django and can be used to serialize a django model.

Lets refactor `snippets/API/serializers.py` file and add the following.

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
```
The serializer we just created don't do anything by themselves we have to create API views to access the data represented by the serializer.

Create `views.py` file in your `snippets/API/` directory and added the following imports to get
started.

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

The root of our API is going to be a view that supports listing all the existing snippets, or creating a new snippet.

```
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

Note that because we want to be able to POST to this view from clients that won't have a CSRF token we need to mark the view as csrf_exempt. 
This isn't something that you'd normally want to do, and REST framework views actually use more sensible behavior than this, but it'll do for our purposes right now.

The above view only handles retrieving the snippets list and create a new one. we need new view
for individual snippet actions like fetching the detail, updating and deleting the snippet.

```python
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

Finally we need these views we just created to be discoverable, so create `snippets/API/urls.py` 
file and add the following routes.

```python
from django.urls import path
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
]
```

and then we need register the new `snippets/API/urls.py` file to the main `rest_frameworks_tutorial/urls.py`.

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('snippets.API.urls')),
]
```

Now we can view our API from the browser go to `http://127.0.0.1:8000/snippets/` and that route 
will list all snippets stored in the database in serialized JSON format.

So far the views we have created can technically perform CRUD operation but we could polish them 
a little to add context to responses.

REST framework provides two wrappers you can use to write API views.

* The `@api_view` decorator for working with function based views.
* The `APIView` class for working with class-based views.

These wrappers provide a few bits of functionality such as making sure you receive Request instances in your view, and adding context to Response objects so that content negotiation can be performed.

The wrappers also provide behaviour such as returning `405_Method_Not_Allowed` responses when appropriate, and handling any `ParseError`exceptions that occur when accessing `request.data` with malformed input.

So lest use these new components to refactor our views a little bit.

```python
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

Here is the view for an individual snippet, in the views.py module.

```python
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

This should all feel very familiar - it is not a lot different from working with regular Django views.

Notice that we're no longer explicitly tying our requests or responses to a given content type. `request.data` can handle incoming json requests, but it can also handle other formats. Similarly we're returning response objects with data, but allowing REST framework to render the response into the correct content type for us.

Now the API are browseablefrom the browser to test your API endpoints. 

We have concluded the Quick Start Guide part of this documentation. If you want to go in depth use the content table bellow.

#### Table Of Contents

1. [Quick Start](https://github.com/endalk200/DjangoRestFramework/blob/master/README.md).
2. [Class Based Views](https://github.com/endalk200/DjangoRestFramework/blob/master/ClassBasedViews.md).
	1. Class Based API Views
	2. Using Mixins
	3. Using Generic Class Based Views
3. Authentication and Permission
4. Relationships and Hyper Linked APIs
5. Viewsets andd Routers
6. Caching
7. Throtling
8. Filters
9. Pagination
10. Content Negotiation
11. Metadata and Schemas
12. Exceptions
13. Testing 
14. Settings
15. AJAX, CSRF, CORS

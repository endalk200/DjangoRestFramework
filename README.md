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

Before we proced lets se how we can use the serializer we just created in python shell

	python manage.py shell
	
execute the following

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



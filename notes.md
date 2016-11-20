Table of Contents:

* [Part 1](#part1): Intro and Overview
    * [creating a project](#create)
    * [manage.py](#manage)
    * [views.py](#views)
    * [urls.py](#urls)

* [Part 2](#part2): Databases
    * [settings.py and db setup](#settings)
    * [models.py and migrations](#models)
    * [database queries](#queries)
    * [admin panel and db management](#dbadmin)



## <a name="part1"></a> [part 1: creating a project, views, url redirects, http request/response](https://docs.djangoproject.com/en/1.10/intro/tutorial01/)

### <a name="create"></a> creating a project

    django-admin startproject mysite

(never store in `root`)

creates a directory tree for the **project**:

    mysite/
        manage.py
        mysite/
            __init__.py
            settings.py
            urls.py
            wsgi.py


### <a name="manage"></a>  [`manage.py`](https://docs.djangoproject.com/en/1.10/ref/django-admin/)

use `manage.py` to manage your project (and its database):

* **run the server** locally:

        python manage.py runserver

* **create an app** (in this case a polls app):

        python manage.py startapp polls

### <a name="views"></a> `views.py`

Views for each app are stored **in the app's** `views.py` file. To allow views to respond to a request:

    from django.http import HttpResponse

then **pass the request object** to the view and return a response:

    def index(request):
        return HttpResponse("Hello, world. You're at the polls index.")

### <a name="urls"></a> `urls.py`: the URLconf module

URL redirects live in the **project** directory's `urls.py`:

    from django.conf.urls import include, url
    from django.contrib import admin

    urlpatterns = [
        url(r'^polls/', include('polls.urls')),
        url(r'^admin/', admin.site.urls),
    ]

The first **regex** specifies to look for `<www.somedomain.com>/polls/`


The second argument's `include()` function says: 'look at some other URLconf'. In this case, check `mysite/polls/urls.py`. It should also import url:

    from django.conf.urls import url

and should import the views file so that it can generate the correct views:


    from . import views

    urlpatterns = [
        url(r'^$', views.index, name='index'),
    ]
The **regex** here specifies the ending of the slug. The second argument passes an httpRequest object to the index view. The name kwarg makes index available as a shorter variable.

Back in the project's `urls.py`, `admin.site.urls` is used to access the admin console app.


## <a name="part2"></a> [part 2: database setup, intro to models, intro to admin panel](https://docs.djangoproject.com/en/1.10/intro/tutorial02/)

### <a name="settings"></a> `settings.py` and db setup

`settings.py` stores all the config info.

`INSTALLED_APPS` near the top shows the apps installed in the project. [admin](https://docs.djangoproject.com/en/1.10/ref/contrib/admin/#module-django.contrib.admin), [auth](https://docs.djangoproject.com/en/1.10/topics/auth/#module-django.contrib.auth), [contenttypes](https://docs.djangoproject.com/en/1.10/ref/contrib/contenttypes/#module-django.contrib.contenttypes), [sessions](https://docs.djangoproject.com/en/1.10/topics/http/sessions/#module-django.contrib.sessions), [messages](https://docs.djangoproject.com/en/1.10/ref/contrib/messages/#module-django.contrib.messages) and [staticfiles](https://docs.djangoproject.com/en/1.10/ref/contrib/staticfiles/#module-django.contrib.staticfiles) are included by default. They can be commented out or removed to make the project smaller.

Specify `TIME_ZONE` at the bottom.

In the `DATABASES` dictionary, if using **sqlite3** don't change anything. Otherwise, change `ENGINE` to specify the db type.  `NAME` should have the absolute path + filename. For other db's also specify `USER`, `PASSWORD`, and `HOST` as per [database info](https://docs.djangoproject.com/en/1.10/ref/settings/#std:setting-DATABASES). **NoSQL** would require different setup.

Create tables in the database for the existing apps with `python manage.py migrate`. It reads from `settings.py -> INSTALLED_APPS`.

### <a name="models"></a> `models.py` and migrations


#### 1. **make a model**

Similar syntax to peewee for django-ORM eg. in `polls/models.py`:


    import datetime

    from django.db import models
    from django.utils.encoding import python_2_unicode_compatible
    from django.utils import timezone

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Question(models.Model):
        question_text = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')

        def __str__(self):
            return self.question_text

            def was_published_recently(self): return self.pub_date >= timezone.now() - datetime.timedelta(days=1)

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Choice(models.Model):
        question = models.ForeignKey(Question, on_delete=models.CASCADE)
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)

        def __str__(self):
            return self.choice_text

Optional first arg for human readable name e.g. `'date published'` passed to `pub_date`.

Similarly, use the `__str__` magic method to provide a human readable representation of the object when running queries. This will also change the display in the admin panel.

[`models.CASCADE`](https://docs.djangoproject.com/en/1.10/ref/models/fields/#django.db.models.CASCADE) is used to delete the foreign key row and its representative object when the parent row is deleted.

Need to add the new models to `mysite/settings.py` like so:

    INSTALLED_APPS = [
        'polls.apps.PollsConfig', # This guy.
        'django.contrib.admin',
        ...
        'django.contrib.staticfiles',
    ]

#### 2. **create a new migration**

Create a new migration with `makemigrations`: `python manage.py makemigrations polls`

    "You should think of [migrations](https://docs.djangoproject.com/en/1.10/topics/migrations/) as a version control system for your database schema. makemigrations is responsible for packaging up your model changes into individual migration files - analogous to commits - and migrate is responsible for applying those to your database."

This is like adding files to staging in git with `git add [stuff]`.

See the sql from the migration with `sqlmigrate`: `python manage.py sqlmigrate polls 0001`

**Note**: an autoincrementing primary key is added by default (like peewee). Foreign key field names have `_id` appended to them.

Check for problems before migrating with `check`: `python manage.py check`

#### 3. **add the migration to the project**

Create the tables with `migrate`:  `python manage.py migrate`

This is like committing the files from staging in git.

### <a name="queries"></a> [Database Queries](https://docs.djangoproject.com/en/1.10/topics/db/queries/)

Open the interactive shell with `python manage.py shell`.

import the models:

`from polls.models import Question, Choice`

query the questions:

`Question.objects.all()`

import timezone for datetime + tz support

`from django.utils import timezone`

create and save a row to the `Question` table:

`q = Question(question_text="What's new?", pub_date=timezone.now())`

`q.save()`

Check fields pythonically e.g. `q.id`, `q.question_text`, `q.pub_date`.

`Question.objects.all()` gives the `__str__` representation of the existing question.

You can also search by filter e.g. `Question.objects.filter(id=1)`, `Question.objects.filter(question_text__startswith='What')`,

import timezone to run time-based queries:

    from Django.utils import timezone
    current_year = timezone.now().year
    Question.objects.get(pub_date__year=current_year)

retrieve an object by primary key with `Question.objects.get(pk=1)`

Run a custom method:

    q = Question.objects.get(pk=1)
    q.was_published_recently()

Create new child objects:

    q.choice_set.create(choice_text='Not much', votes=0)
    q.choice_set.create(choice_text='The sky', votes=0)
    c = q.choice_set.create(choice_text='Just hacking again', votes=0)

Get child objects (rows with the object's primary key as foreign key):

    q.choice_set.all()

Get number of children:

    q.choice_set.count()

Get the parent object from the child: `c.question`

`__`, dunder, separates relationships to build longer queries eg:

    Choice.objects.filter(question__pub_date__year=current_year)


delete a set of objects:

    c = q.choice_set.filter(choice_text__startswith='Just hacking')

    c.delete()

### <a name="dbadmin"></a> admin panel: intro and database mgmt

Create a super user: `python manage.py createsuperuser`

You'll be prompted for a username, email address, password (prompted twice).

Start the server with `python manage.py runserver`

Go to `127.0.0.1:8000/admin/` and log into the admin panel as the new superuser.

The question objects are unavailable. Enable them by adding the following to `polls/admin.py`:

    from .models import Question

    admin.site.register(Question)

You can drill down and modify any existing db rows through the admin panel.

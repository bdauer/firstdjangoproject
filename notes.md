Table of Contents:

* [Part 1](#part1): Intro and Overview
    * [creating a project](#create)
    * [manage.py / creating an app](#manage)
    * [views.py](#views)
    * [urls.py](#urls)
* [Part 2](#part2): Databases
    * [settings.py and db setup](#settings)
    * [models.py and migrations](#models)
    * [database queries](#queries)
    * [admin panel and db management](#dbadmin)
* [Part 3](#part3) Templates
    * [views with add'l args](#moreviews)
    * [templates](#templates)
    * [404 errors](#404)
    * [the template engine and working with URLs](#tempeng)
* [Part 4](#part4) Forms and Generic Views
    * [forms](#forms)
    * [generic views](#genviews)


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


### <a name="manage"></a> [`manage.py`](https://docs.djangoproject.com/en/1.10/ref/django-admin/)/creating an app

use `manage.py` to manage your project (and its database):

* **run the server** locally:

        python manage.py runserver

* **create an app** (in this case a polls app):

        python manage.py startapp polls

### <a name="views"></a> `views.py`

**note** This tut goes through writing out views. Consider [generic views](https://docs.djangoproject.com/en/1.10/topics/class-based-views/generic-display/) for more concise, DRY code. See also in these notes: [genviews](#genviews).

Views for each app are stored **in the app's** `views.py` file. To allow views to respond to a request:

    from django.http import HttpResponse

then **pass the request object** to the view and return a response:

    def index(request):
        return HttpResponse("Hello, world. You're at the polls index.")

### <a name="urls"></a> [`urls.py](https://docs.djangoproject.com/en/1.10/ref/urlresolvers/#module-django.urls)`: the URLconf module`

URL redirects live in the **project** directory's `urls.py`:

    from django.conf.urls import include, url
    from django.contrib import admin

    urlpatterns = [
        url(r'^polls/', include('polls.urls')),
        url(r'^admin/', admin.site.urls),
    ]

The first **regex** specifies to look for `<www.somedomain.com>/polls/`. Once the main URLconf file is pointed at the app's location, any additional views in the app's URLconf will be seen by the project.


The second argument's `include()` function says: 'look at some other URLconf'. In this case, check `mysite/polls/urls.py`. It should also import url:

    from django.conf.urls import url

and should import the views file so that it can generate the correct views:

    from . import views

    urlpatterns = [
        url(r'^$', views.index, name='index'),
    ]
The **regex** here specifies the ending of the slug. The second argument passes an httpRequest object to the index view. The name kwarg makes index available as a shorter variable. This will be useful when referencing URLs within templates.

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


(to have the database run queries without involving python, see: [`F()`](https://docs.djangoproject.com/en/1.10/ref/models/expressions/#f-expressions))

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

Check fields Pythonically e.g. `q.id`, `q.question_text`, `q.pub_date`.

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

## <a name="part3"></a>  Part 3: Templates

### <a name="moreviews"></a> views with additional arguments

Update `polls/views.py`:

    def detail(request, question_id):
        return HttpResponse("You're looking at question %s." % question_id)

    def results(request, question_id):
        response = "You're looking at the results of question %s."
        return HttpResponse(response % question_id)

    def vote(request, question_id):
        return HttpResponse("You're voting on question %s." % question_id)

Then connect the new views to the app in the URLconf file, `polls/urls.py`

    from django.conf.urls import url

    from . import views

    urlpatterns = [
        # ex: /polls/
        url(r'^$', views.index, name='index'),
        # ex: /polls/5/
        url(r'^(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
        # ex: /polls/5/results/
        url(r'^(?P<question_id>[0-9]+)/results/$', views.results, name='results'),
        # ex: /polls/5/vote/
        url(r'^(?P<question_id>[0-9]+)/vote/$', views.vote, name='vote'),
    ]

Start the site locally and navigate to `/polls/34` as well as `/polls/34/results` and `/polls/34/vote` to see how the pages are constructed out of the views.

1. When seeking a url's view, Django first checks the project's URLconf file to find the app, passing the remainder of the URL along.
2. That remainder goes to the app's URLconf file to continue to parse the URL.
3. When a match is found, Django looks for a view with the same name as the `urlpattern`.
4. The HttpRequest and any additional arguments are passed to the view.

### <a name="templates"></a> [templates](https://docs.djangoproject.com/en/1.10/topics/templates/)

Each app should have its own templates folder e.g. `mysite/polls/templates/polls`. The inner `polls` folder is used for namespacing. Otherwise, if there are templates with the same name in two different apps, Django may grab the wrong template.

Add an `index.html` file at the very bottom of the new directory hierarchy containing:

    {% if latest_question_list %}
        <ul>
        {% for question in latest_question_list %}
            <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
        {% endfor %}
        </ul>
    {% else %}
        <p>No polls are available.</p>
    {% endif %}

Change `polls/views.py`. First import the template loader and the Question model:

    from django.template import loader

    from .models import Question


Beneath that pass suitable args along to the `index.html` template:

    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        template = loader.get_template('polls/index.html')
        context = {
            'latest_question_list': latest_question_list,
        }
        return HttpResponse(template.render(context, request))

Visit `/polls` to see the result.

#### the `render()` shortcut

Import the function with `from django.shortcuts import render`.

Rewrite the index view as follows:

    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        context = {'latest_question_list': latest_question_list}
        return render(request, 'polls/index.html', context)

With `render()`, you don't need to import `HttpResponse` or the template loader. Primarily it makes the return statement more concise and Pythonic.

### <a name="404"></a> 404 errors

Update `views.py`:

    from django.http import Http404
    from django.shortcuts import render

    from .models import Question
    # ...
    def detail(request, question_id):
        try:
            question = Question.objects.get(pk=question_id)
        except Question.DoesNotExist:
            raise Http404("Question does not exist")
        return render(request, 'polls/detail.html', {'question': question})

The `try-except` block will ensure that if the queried object doesn't exist, we raise a 404 error.

**note**: At this point there's no template, so this won't render.

#### the `get_object_or_404()` shortcut

Update `views.py`:

    from django.shortcuts import get_object_or_404, render

    from .models import Question
    # ...
    def detail(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/detail.html', {'question': question})

This syntax replaces the `try-except` block with one concise function.

`get_list_or_404()` is used for querying multiple objects with `filter()`,

### <a name="tempeng"></a> template engine basics and working with urls

Create `detail.html` in `/polls/templates/polls`:

    <h1>{{ question.question_text }}</h1>
    <ul>
    {% for choice in question.choice_set.all %}
        <li>{{ choice.choice_text }}</li>
    {% endfor %}
    </ul>

The basics are pretty much like jinja2.


#### removing hardcoded urls

Hardcoding URLs means that if you want to make changes to URLs later you'll have to change them in multiple places.

In `index.html` change from this:

    <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>

to this:

    <li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>

This uses the url name assigned in `/polls/urls.py` instead of an explicit `href`.

Now if the url changes, only `urls.py` needs to change

#### namespacing urls

Add `app_name = 'polls'` above `urlpatterns` in `polls/urls.py`.

Change the `<li>` entry in `polls/templates/polls/index.html` to `<li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>`

Note the change to `polls:detail`. By creating a namespace for the app URLs we prevent conflicts with variable names for other apps in the project, much like we did for templates.

## <a name="part4"></a> Part 4: Forms and Generic Views

### <a name="forms"></a> forms

Update `polls/detail.html`:

    <h1>{{ question.question_text }}</h1>

    {% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

    <form action="{% url 'polls:vote' question.id %}" method="post">
    {% csrf_token %}
    {% for choice in question.choice_set.all %}
        <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}" />
        <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br />
    {% endfor %}
    <input type="submit" value="Vote" />
    </form>


* choice
    * Added a radio button for each choice.
    * Added a counter to track loops of the for-loop.
    * `value` is the ID of the choice.
    * `name` is the variable name sent with the form.
    * `ID` is the number of the choice, the value associated with `name`, generated by the counter.
* The form action sends a post request to `vote` with the question id.
* In order to prevent cross site request forgeries, **always** include `{% csrf_token %}` in POST requests.

`polls/urls.py` already handles the post request. Update `vote()` in `polls/views.py`:

    from django.shortcuts import get_object_or_404, render
    from django.http import HttpResponseRedirect, HttpResponse
    from django.urls import reverse

    from .models import Choice, Question
    # ...
    def vote(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        try:
            selected_choice = question.choice_set.get(pk=request.POST['choice'])
        except (KeyError, Choice.DoesNotExist):
            # Redisplay the question voting form.
            return render(request, 'polls/detail.html', {
                'question': question,
                'error_message': "You didn't select a choice.",
            })
        else:
            selected_choice.votes += 1
            selected_choice.save()
            # Always return an HttpResponseRedirect after successfully dealing
            # with POST data. This prevents data from being posted twice if a
            # user hits the Back button.
            return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))

* use request.POST[`name`] to access the variables from the POST request.
* a `KeyError` is raised if the `name` variable doesn't exist.
* `HttpResponseRedirect` prevents multiple form submits on successful submission.
* [`reverse`](https://docs.djangoproject.com/en/1.10/ref/urlresolvers/#django.urls.reverse) redirects to the view given as its first arg.

Update `results()` in `polls/views.py`:

    def results(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/results.html', {'question': question})

Add a results template as `polls/templates/polls/results.html`

    <h1>{{ question.question_text }}</h1>

    <ul>
    {% for choice in question.choice_set.all %}
        <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
    {% endfor %}
    </ul>

    <a href="{% url 'polls:detail' question.id %}">Vote again?</a>

See the result at `http://127.0.0.1:8000/polls/1/`.

**Caution**: If two users vote at the same time that will cause a race condition. See [race conditions](https://docs.djangoproject.com/en/1.10/ref/models/expressions/#avoiding-race-conditions-using-f) for how to prevent this. (briefly: use the `F()` function to make changes in the database without involving python)

### <a name="genviews"></a> generic views

Use generic views to abstract common patterns. They're like abstract base classes or interfaces.

**Note**: to compare before and after, refer to [this commit](https://github.com/bdauer/firstdjangoproject/tree/bc45eef2a3c0ca179a6e9219f8f94dc00ccc0447) made just before the changes below.

To convert the polls app:

#### 1. Convert `polls/urls.py`

Amend `polls/urls.py`:

    from django.conf.urls import url

    from . import views

    app_name = 'polls'
    urlpatterns = [
        url(r'^$', views.IndexView.as_view(), name='index'),
        url(r'^(?P<pk>[0-9]+)/$', views.DetailView.as_view(), name='detail'),
        url(r'^(?P<pk>[0-9]+)/results/$', views.ResultsView.as_view(), name='results'),
        url(r'^(?P<question_id>[0-9]+)/vote/$', views.vote, name='vote'),
    ]

#### Replace redundant views in `polls/views.py`

replace `index`, `detail`, `results` and inherit from generics.

    from django.shortcuts import get_object_or_404, render
    from django.http import HttpResponseRedirect
    from django.urls import reverse
    from django.views import generic

    from .models import Choice, Question


    class IndexView(generic.ListView):
        template_name = 'polls/index.html'
        context_object_name = 'latest_question_list'

        def get_queryset(self):
            """Return the last five published questions."""
            return Question.objects.order_by('-pub_date')[:5]


    class DetailView(generic.DetailView):
        model = Question
        template_name = 'polls/detail.html'


    class ResultsView(generic.DetailView):
        model = Question
        template_name = 'polls/results.html'


    def vote(request, question_id):
        ... # same as above, no changes needed.


#### Explanation

We're using two most common generic views for displaying data (DetailView and ListView). More info [here](https://docs.djangoproject.com/en/1.10/ref/class-based-views/generic-display/)

* The `model` attribute says which model the view should use.
* Use `pk` for the primary key value in `urls.py` with `DetailView`, not `id`.
* By default, generics look for templates as `<app name>/<model name>_<generic name>.html`. `template_name` overrides the default behavior.
* When a model is provided, generics will automatically provide context objects for the template based on the model attributes.
* In the absence of a model, use `context_object_name` to override default variable names. e.g. `ListView` will generate `<object>_list` by default, but we want a context object named `latest_<object>_list`.

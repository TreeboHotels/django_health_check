===================
django-health-check
===================

|version| |ci| |coverage| |health| |license|

This project checks the health for a number of backends and sees if they are able
to connect and do a simple action.

The following health check backends are bundled into this project:

- Cache
- Database
- Storage
- AWS S3 storage
- Celery task queue
- RMQ
- SQS

Writing your own custom health checks is also very quick and easy.

We also like contributions, so don't be afraid to make a pull request.

Installation
------------

First add to requirement file:

.. code::

    treebo_health_check

Add the health checker to an URL you want to use:

.. code:: python

    urlpatterns = [
        # ...
        url(r'^api/health/$', include('health_check.urls')),
    ]

Add the ``health_check`` applications to your ``INSTALLED_APPS``:

.. code:: python

    INSTALLED_APPS += [
        # ...
        'health_check',                             # required
        'health_check.db',                          # stock Django health checkers
        'health_check.contrib.sqs',                 # requires boto3
        'health_check.contrib.rmq',                 #requires pika
    ]

SAMPLE CONFIGURATION:

    HEALTH_CHECK_CONF = dict(
        rmq_host='localhost',
        sqs_queue_name='queue_name',
        region_name='eu-west-1',
        aws_secret_access_key='mykey',
        aws_access_key_id= 'access_id',
        soft_dependencies = []
    )
Setting up monitoring
---------------------

You can use tools like Pingdom_ or other uptime robots to monitor service status.
The ``/api/health/`` endpoint will respond a HTTP 200 if all checks passed
and a HTTP 503 if any of the tests failed.

.. code::

    $ curl -v -X GET -H http://www.example.com/api/health/

    > GET /api/health/ HTTP/1.1
    > Host: www.example.com
    > Accept: */*
    >
    < HTTP/1.1 200 OK
    < Content-Type: text/html; charset=utf-8

    <!-- This is an excerpt -->
    <div class="container">
        <h1>System status</h1>
        <table>
            <tr>
                <td class="status_1"></td>
                <td>CacheBackend</td>
                <td>working</td>
            </tr>
            <tr>
                <td class="status_1"></td>
                <td>DatabaseBackend</td>
                <td>working</td>
            </tr>
            <tr>
                <td class="status_1"></td>
                <td>S3BotoStorageHealthCheck</td>
                <td>working</td>
            </tr>
        </table>
    </div>

Getting machine readable JSON reports
-------------------------------------

If you want machine readable status reports you can request the ``/api/health/``
endpoint with the ``Accept`` HTTP header set to ``application/json``.

The backend will return a JSON response:

.. code::

    $ curl -v -X GET -H "Accept: application/json" http://www.example.com/api/health/

    > GET /ht/ HTTP/1.1
    > Host: www.example.com
    > Accept: application/json
    >
    < HTTP/1.1 200 OK
    < Content-Type: application/json

    {
    "resources":
        [
            {"state": "ok", "resource": "DatabaseBackend", "details": "working"},
            {"state": "fail", "resource": "RMQHealthCheck", "details": "unavailable: Connection Error"}
         ]
     }

Writing a custom health check
-----------------------------

Writing a health check is quick and easy:

.. code:: python

    from health_check.backends import BaseHealthCheckBackend

    class MyHealthCheckBackend(BaseHealthCheckBackend):
        def check_status(self):
            # The test code goes here.
            # You can use `self.add_error` or
            # raise a `HealthCheckException`,
            # similar to Django's form validation.
            pass

        def identifier(self):
            return self.__class__.__name__  # Display name on the endpoint.

After writing a custom checker, register it in your app configuration:

.. code:: python

    from django.apps import AppConfig

    from health_check.plugins import plugin_dir

    class MyAppConfig(AppConfig):
        name = 'my_app'

        def ready(self):
            from .backends import MyHealthCheckBackend
            plugin_dir.register(MyHealthCheckBackend)

Make sure the application you write the checker into is registered in your ``INSTALLED_APPS``.

Customizing output
------------------

You can customize HTML or JSON rendering by inheriting from ``MainView`` in ``health_check.views``
and customizing the ``template_name``, ``get``, ``render_to_response`` and ``render_to_response_json`` properties:

.. code:: python

    # views.py
    from health_check.views import MainView

    class HealthCheckCustomView(MainView):
        template_name = 'myapp/health_check_dashboard.html'  # customize the used templates

        def get(self, request, *args, **kwargs):
            plugins = []
            # ...
            if 'application/json' in request.META.get('HTTP_ACCEPT', ''):
                return self.render_to_response_json(plugins, status)
            return self.render_to_response(plugins, status)

        def render_to_response(self, plugins, status):       # customize HTML output
            return HttpResponse('COOL' if status == 200 else 'SWEATY', status=status)

        def render_to_response_json(self, plugins, status):  # customize JSON output
            return JsonResponse(
                {str(p.identifier()): 'COOL' if status == 200 else 'SWEATY' for p in plugins}
                status=status
            )

    # urls.py
    import views

    urlpatterns = [
        # ...
        url(r'^ht/$', views.HealthCheckCustomView.as_view(), name='health_check_custom'),
    ]

Other resources
---------------

- django-watchman_ is a package that does some of the same things in a slightly different way.
- See this weblog_ about configuring Django and health checking with AWS Elastic Load Balancer.

.. |version| image:: https://img.shields.io/pypi/v/django-health-check.svg
   :target: https://pypi.python.org/pypi/django-health-check/
.. |ci| image:: https://api.travis-ci.org/KristianOellegaard/django-health-check.svg?branch=master
   :target: https://travis-ci.org/KristianOellegaard/django-health-check
.. |coverage| image:: https://coveralls.io/repos/KristianOellegaard/django-health-check/badge.svg?branch=master
   :target: https://coveralls.io/r/KristianOellegaard/django-health-check
.. |health| image:: https://landscape.io/github/KristianOellegaard/django-health-check/master/landscape.svg?style=flat
   :target: https://landscape.io/github/KristianOellegaard/django-health-check/master
.. |license| image:: https://img.shields.io/badge/license-BSD-blue.svg
   :target: LICENSE

.. _Pingdom: https://www.pingdom.com/
.. _django-watchman: https://github.com/mwarkentin/django-watchman
.. _weblog: https://www.vincit.fi/en/blog/deploying-django-to-elastic-beanstalk-with-https-redirects-and-functional-health-checks/

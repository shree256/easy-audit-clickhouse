# django-easy-audit-clickhouse

[![pypi](https://img.shields.io/pypi/v/django-easy-audit.svg)](https://pypi.org/project/django-easy-audit/)
![PyPI - Django Version](https://img.shields.io/pypi/frameworkversions/django/django-easy-audit)

Logging implementation with clickhouse integration on top of django-easy-audit==1.3.7.

This app allows you to keep track of every action taken by your users.

## Quickstart

1. Prerequisites:
   ```python
   django==4.2
   clickhouse-connect>=0.8.15
   celery>=5.4.0
   djangorestframework>=3.15
   ```

1. Install by running `pip install django-easy-audit-clickhouse`.

2. Add 'easyaudit' to your `INSTALLED_APPS` like this:

   ```python
   INSTALLED_APPS = [
       ...
       'easyaudit',
   ]
   ```

3. Add Easy Audit's middleware to your `MIDDLEWARE` (or `MIDDLEWARE_CLASSES`) setting like this:

   ```python
   MIDDLEWARE = (
       ...
       'easyaudit.middleware.easyaudit.EasyAuditMiddleware',
   )
   ```

4. Run `python manage.py migrate easyaudit` to create the app's models.

5. Configure the ClickHouse connection in your settings.py:

   ```python
   CLICKHOUSE_USER = 'user'
   CLICKHOUSE_PASSWORD = 'password'
   CLICKHOUSE_HOST = 'localhost'
   CLICKHOUSE_PORT = 8123
   CLICKHOUSE_DATABASE = 'default'
   CLICKHOUSE_SECURE = False
   ```

6. Create shared task of `send_logs_to_clickhouse` to sync data from django to clickhouse:

   ```python
   @shared_task
   def send_audit_logs_to_clickhouse():
      from easyaudit.tasks import send_logs_to_clickhouse

      send_logs_to_clickhouse()

   app.conf.beat_schedule = {
        "send-logs-to-clickhouse": {
            "task": "path.to.send_logs_to_clickhouse",
            "schedule": crontab(hour=9, minute=10),  # 12:00 AM PST
        },
    }
   ```

   - Add `SEND_LOGS_TO_CLICKHOUSE` to settings with True/False for enable/disable data push to clickhouse

## Settings

For an exhaustive list of available settings, please [check our wiki](https://github.com/soynatan/django-easy-audit/wiki/Settings).

Below are some of the settings you may want to use. These should be defined in your project's `settings.py` file:

- `DJANGO_EASY_AUDIT_WATCH_MODEL_EVENTS`

- `DJANGO_EASY_AUDIT_WATCH_AUTH_EVENTS`

- `DJANGO_EASY_AUDIT_WATCH_REQUEST_EVENTS`

  Set these to `False` to stop logging model, authentication, and/or request events.

- `DJANGO_EASY_AUDIT_UNREGISTERED_CLASSES_EXTRA`

  A list of Django models which will be ignored by Django Easy Audit.
  Use it to prevent logging one or more of your project's models.
  List items can be classes or strings with `app_name.model_name` format.

- `DJANGO_EASY_AUDIT_UNREGISTERED_URLS_EXTRA`

  A list of URLs which will be ignored by Django Easy Audit.
  List items are expected to be regular expressions that
  will be matched against the URL path.

- `DJANGO_EASY_AUDIT_CRUD_DIFFERENCE_CALLBACKS`

  May point to a list of callables/string-paths-to-functions-classes in which the application code can determine
  on a per CRUDEvent whether or not the application chooses to create the CRUDEvent or not. This is different
  from the registered/unregistered settings (e.g. `DJANGO_EASY_AUDIT_UNREGISTERED_CLASSES_EXTRA`).
  This is meant to be for dynamic configurations where the application
  may inspect the current save/create/delete and choose whether or not to save that into the database or ignore it.

- `DJANGO_EASY_AUDIT_USER_DB_CONSTRAINT`

  Default is `True`. This is reserved for future use (does not do anything yet). The functionality provided by the
  setting (whether enabled or disabled) could be handled more explicitly in certain
  code paths (or even internally as custom model managers). For projects that separate the easyaudit database, such
  that the tables are not on the same database as the user table, this could help with making certain queries easier.
  Again, this doesn't do anything yet, and if it ever does, the version will be increased and the README will be
  updated accordingly. If you keep your database together (the standard usage), you have nothing to worry about.

- `DJANGO_EASY_AUDIT_CRUD_EVENT_LIST_FILTER`

- `DJANGO_EASY_AUDIT_LOGIN_EVENT_LIST_FILTER`

- `DJANGO_EASY_AUDIT_REQUEST_EVENT_LIST_FILTER`

  Changeview filters configuration.
  Used to remove filters when the corresponding list of data would be too long.
  Defaults are:

  - ['event_type', 'user', 'created_at', ] for CRUDEventAdmin
  - ['login_type', 'user', 'created_at', ] for LoginEventAdmin

- `DJANGO_EASY_AUDIT_DATABASE_ALIAS`

  By default it is the Django `default` database alias. But for projects that have split databases,
  this is necessary in order to keep database atomicity concerns in check during signal handlers.

  To clarify, this is only _truly_ necessary for the model signals.

- `DJANGO_EASY_AUDIT_PROPAGATE_EXCEPTIONS`

  Default is `False`. When set to `True`, easyaudit will propagate exceptions occurred in own signal handlers. The
  recommended approach is to use Django's `DEBUG` setting in order to only propagate errors in development:

  ```python
  DJANGO_EASY_AUDIT_PROPAGATE_EXCEPTIONS = DEBUG
  ```

- `DJANGO_EASY_AUDIT_CRUD_EVENT_NO_CHANGED_FIELDS_SKIP`

  By default this is `False`, but this allows the calling project not to save `CRUDEvent` if the changed fields as
  determined by the `pre_save` handler sees that there are no changed fields. We are keeping it off by default so that
  projects that wish to use this (potentially less `CRUDEvent`) can choose to turn it on! And those that do not want it (yet or ever),
  or those that do not closely follow the release notes of this project will have one less worry when upgrading.

- `DJANGO_EASY_AUDIT_CHECK_IF_REQUEST_USER_EXISTS`

  By default this is `True`, but this allows the calling project to make easyaudit ignore user validation on audit event creation.
  This is useful when you have a app with soft delete or no delete on users model. With this set to `False`, easyaudit only fetch `request.user` for audit event creation, no db check is made, meaning you can speed up audit events creation and save some DB calls.

- `DJANGO_EASY_AUDIT_READONLY_EVENTS`

  Default is `False`. The events visible through the admin interface are editable by default by a
  superuser. Set this to `True` if you wish to make the recorded events read-only through the admin
  UI.

- `DJANGO_EASY_AUDIT_LOGGING_BACKEND`

  A pluggable backend option for logging. Defaults to `easyaudit.backends.ModelBackend`.
  This class expects to have 3 methods:

  - `login(self, login_info_dict):`
  - `crud(self, crud_info_dict):`
  - `request(self, request_info_dict):`

  each of these methods accept a dictionary containing the info regarding the event.
  example overriding:

  ```python
    import logging

    class PythonLoggerBackend:
        logging.basicConfig()
        logger = logging.getLogger('your-kibana-logger')
        logger.setLevel(logging.DEBUG)

        def request(self, request_info):
            return request_info # if you don't need it

        def login(self, login_info):
            self.logger.info(msg='your message', extra=login_info)
            return login_info

        def crud(self, crud_info):
            self.logger.info(msg='your message', extra=crud_info)
            return crud_info
  ```


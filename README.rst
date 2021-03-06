-------------
django-db-log
-------------

Logs Django exceptions to your database handler.

=========
Upgrading
=========

The first thing you will want to do is confirm your database matches. Do this by verifying your version, or simply taking a look at the changes::

	python manage.py sql djangodblog > dblog.sql
	mysqldump -d --skip-opt -uroot -p yourdatabase djangodblog_error djangodblog_errorbatch > dblog.cur.sql
	diff -u dblog.sql dblog.cur.sql

Note: the above example is using MySQL, and isn't going to give anywhere near a precise diff.

Review the diff, then make any changes which appear nescesary.

###############
Notable Changes
###############

* 1.4.0 Added `logger` column to both Error and ErrorBatch. `traceback` and `class_name` are now nullable.
* 1.3.0 Added `level` column to both Error and ErrorBatch.

=======
Install
=======

The easiest way to install the package is via setuptools::

	easy_install django-db-log

Once installed, update your settings.py and add the middleware and installed apps settings::

	MIDDLEWARE_CLASSES = (
	    'django.middleware.common.CommonMiddleware',
	    'django.contrib.sessions.middleware.SessionMiddleware',
	    'django.contrib.auth.middleware.AuthenticationMiddleware',
	    ...
	    'djangodblog.middleware.DBLogMiddleware',
	)

	INSTALLED_APPS = (
	    'django.contrib.admin',
	    'django.contrib.auth',
	    'django.contrib.contenttypes',
	    'django.contrib.sessions',
	    'djangodblog',
	    ...
	)

Finally, run ``python manage.py syncdb`` to create the database tables.

=============
Configuration
=============

Several options exist to configure django-db-log via your ``settings.py``:

######################
DBLOG_CATCH_404_ERRORS
######################

Enable catching of 404 errors in the logs. Default value is ``False``::

	DBLOG_CATCH_404_ERRORS = True

##############
DBLOG_DATABASE
##############

Warning: This feature is currently in the testing phase.

Use a secondary database to store error logs. This is useful if you have several websites and want to aggregate error logs onto one database server::

	DBLOG_DATABASE = dict(
	    DATABASE_ENGINE='mysql', # defaults to settings.DATABASE_ENGINE
	    DATABASE_NAME='my_db_name',
	    DATABASE_USER='db_user',
	    DATABASE_PASSWORD='db_pass',
	    DATABASE_HOST='localhost', # defaults to localhost
	    DATABASE_PORT='', # defaults to [default port]
	    DATABASE_OPTIONS={}
	)

Some things to note:

* You will need to create the tables by hand if you use this option. Use ``python manage.py sql djangodblog`` and dump that SQL into the correct server.
* This functionality does not yet support Django 1.2.

##################
DBLOG_WITH_LOGGING
##################

django-db-log supports the ability to directly tie into your ``logging`` module entries. To use it simply define ``DBLOG_WITH_LOGGING=True`` in your ``settings.py``.

Note: The `class_name` (also labeled `type`) will use the record's logger name value.

You will also most likely want to configure the logging module, as you would in any logging situation, using something similar to::

	DBLOG_WITH_LOGGING = True
	
	import logging
	
	logging.basicConfig(
	    level=DEBUG and logging.DEBUG or logging.INFO,
	    format='%(asctime)s %(levelname)-8s %(message)s',
	    datefmt='%a, %d %b %Y %H:%M:%S',
	)

If you only want to use django-db-log's logging mechanism, you can remove the default handlers using something similar to::

	logger = logging.getLogger()
	for h in logger.handlers:
	    logger.removeHandler(h)

=====
Usage
=====

You will find two new admin panels in the automatically built Django administration:

* Errors (Error)
* Error batches (ErrorBatch)

It will store every single error inside of the `Errors` model, and it will store a collective, or summary, of errors inside of `Error batches` (this is more useful for most cases). If you are using this on multiple sites with the same database, the `Errors` table also contains the SITE_ID for which it the error appeared on.

If you wish to access these within your own views and models, you may do so via the standard model API::

	from djangodblog.models import Error, ErrorBatch
	
	# Pull the last 10 unresolved errors.
	ErrorBatch.objects.filter(status=0).order_by('-last_seen')[0:10]

You can also record errors outside of middleware if you want::

	from djangodblog.models import Error
	
	try:
		...
	except Exception, exc:
		Error.objects.create_from_exception(exc, [url=None])

If you wish to log normal messages (useful for non-``logging`` integration)::

	from djangodblog.models import Error
	import logging
	
	Error.objects.create_from_text('Error Message'[, level=logging.WARNING, url=None])

Both the ``url`` and ``level`` parameters are optional. ``level`` should be one of the following:

* ``logging.DEBUG``
* ``logging.INFO``
* ``logging.WARNING``
* ``logging.ERROR``
* ``logging.FATAL``

=====
Notes
=====

* django-db-log will automatically integrate with django-idmapper.
* Multi-db support (via ``DBLOG_DATABASE``) will most likely not work in Django 1.2
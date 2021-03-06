===================
Manual installation
===================

This page covers the basic installation of horizon in a production
environment. If you are looking for a developer environment, see
:ref:`quickstart`.

.. _system-requirements-label:

System Requirements
===================

* Python 2.7
* Django 1.8
* An accessible `keystone <https://docs.openstack.org/developer/keystone>`_ endpoint

* All other services are optional.
  Horizon supports the following services as of the Pike release.
  If the keystone endpoint for a service is configured,
  horizon detects it and enables its support automatically.

  * `cinder <https://docs.openstack.org/developer/cinder>`_: Block Storage
  * `glance <https://docs.openstack.org/developer/glance>`_: Image Management
  * `heat <https://docs.openstack.org/developer/heat>`_: Orchestration
  * `neutron <https://docs.openstack.org/developer/neutron>`_: Networking
  * `nova <https://docs.openstack.org/developer/nova>`_: Compute
  * `swift <https://docs.openstack.org/developer/swift>`_: Object Storage
  * Horizon also supports many other OpenStack services via plugins. For more
    information, see the :ref:`install-plugin-registry`.

Installation
============

.. note::

  In the commands below, substitute "<release>" for your version of choice,
  such as "ocata" or "pike".

#. Clone Horizon

   .. code-block:: console

     $ git clone https://git.openstack.org/openstack/horizon -b stable/<release> --depth=1
     $ cd horizon

#. Install the horizon python module into your system

   .. code-block:: console

     $ sudo pip install -c http://git.openstack.org/cgit/openstack/requirements/plain/upper-constraints.txt?h=stable/<release> .

Configuration
=============

This section contains a small summary of the critical settings required to run
horizon. For more details, please refer to :ref:`install-settings`.

Settings
--------

Create ``openstack_dashboard/local/local_settings.py``. It is usually a good
idea to copy ``openstack_dashboard/local/local_settings.py.example`` and
edit it. As a minimum, the follow settings will need to be modified:

``DEBUG``
  Set to ``False``
``ALLOWED_HOSTS``
  Set to your domain name(s)
``OPENSTACK_HOST``
  Set to the IP of your Keystone endpoint. You may also
  need to alter ``OPENSTACK_KEYSTONE_URL``

.. note::

  The following steps in the "Configuration" section are optional, but highly
  recommended in production.

Translations
------------

Compile translation message catalogs for internationalization. This step is
not required if you do not need to support languages other than US English.
GNU ``gettext`` tool is required to compile message catalogs.

.. code-block:: console

  $ sudo apt-get install gettext
  $ ./manage.py compilemessages

Static Assets
-------------

Compress your static files by adding ``COMPRESS_OFFLINE = True`` to your
``local_settings.py``, then run the following commands

.. code-block:: console

  $ ./manage.py collectstatic
  $ ./manage.py compress

Logging
-------

Horizons uses Django's logging configuration mechanism, which can be customized
by altering the ``LOGGING`` dictionary in ``local_settings.py``. By default,
Horizon's logging example sets the log level to ``INFO``.

Horizon also uses a number of 3rd-party clients which log separately. The
log level for these can still be controlled through Horizon's ``LOGGING``
config, however behaviors may vary beyond Horizon's control.

For more information regarding configuring logging in Horizon, please
read the `Django logging directive`_ and the `Python logging directive`_
documentation. Horizon is built on Python and Django.

.. _Django logging directive: https://docs.djangoproject.com/en/dev/topics/logging
.. _Python logging directive: http://docs.python.org/2/library/logging.html

Session Storage
---------------

Horizon uses `Django's sessions framework`_ for handling session data. There
are numerous session backends available, which are selected through the
``SESSION_ENGINE`` setting in your ``local_settings.py`` file.

.. _Django's sessions framework: https://docs.djangoproject.com/en/dev/topics/http/sessions/

Memcached
~~~~~~~~~

.. code-block:: python

  SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
  CACHES = {
      'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache'
      'LOCATION': 'my_memcached_host:11211',
  }

External caching using an application such as memcached offers persistence
and shared storage, and can be very useful for small-scale deployment and/or
development. However, for distributed and high-availability scenarios
memcached has inherent problems which are beyond the scope of this
documentation.

Requirements:

* Memcached service running and accessible
* Python memcached module installed

Database
~~~~~~~~

.. code-block:: python

  SESSION_ENGINE = 'django.core.cache.backends.db.DatabaseCache'
  DATABASES = {
      'default': {
          # Database configuration here
      }
  }

Database-backed sessions are scalable (using an appropriate database strategy),
persistent, and can be made high-concurrency and highly-available.

The downside to this approach is that database-backed sessions are one of the
slower session storages, and incur a high overhead under heavy usage. Proper
configuration of your database deployment can also be a substantial
undertaking and is far beyond the scope of this documentation.

Cached Database
~~~~~~~~~~~~~~~

To mitigate the performance issues of database queries, you can also consider
using Django's ``cached_db`` session backend which utilizes both your database
and caching infrastructure to perform write-through caching and efficient
retrieval. You can enable this hybrid setting by configuring both your database
and cache as discussed above and then using

.. code-block:: python

  SESSION_ENGINE = "django.contrib.sessions.backends.cached_db"

Deployment
==========

#. Set up a web server with WSGI support. For example, install Apache web
   server on Ubuntu

   .. code-block:: console

     $ sudo apt-get install apache2 libapache2-mod-wsgi

   You can either use the provided ``openstack_dashboard/wsgi/django.wsgi`` or
   generate a ``openstack_dashboard/wsgi/horizon.wsgi`` file with the following
   command (which detects if you use a virtual environment or not to
   automatically build an adapted WSGI file)

   .. code-block:: console

     $ ./manage.py make_web_conf --wsgi

   Then configure the web server to host OpenStack dashboard via WSGI.
   For apache2 web server, you may need to create
   ``/etc/apache2/sites-available/horizon.conf``.
   The template in DevStack is a good example of the file.
   http://git.openstack.org/cgit/openstack-dev/devstack/tree/files/apache-horizon.template
   Or, if you previously generated an ``openstack_dashboard/wsgi/horizon.wsgi``
   you can automatically generate an apache configuration file

   .. code-block:: console

     $ ./manage.py make_web_conf --apache > /etc/apache2/sites-available/horizon.conf

   Same as above but if you want SSL support

   .. code-block:: console

     $ ./manage.py make_web_conf --apache --ssl --sslkey=/path/to/ssl/key --sslcert=/path/to/ssl/cert > /etc/apache2/sites-available/horizon.conf

   By default the apache configuration will launch a number of apache processes
   equal to the number of CPUs + 1 of the machine on which you launch the
   ``make_web_conf`` command. If the target machine is not the same or if you
   want to specify the number of processes, add the ``--processes`` option

   .. code-block:: console

     $ ./manage.py make_web_conf --apache --processes 10 > /etc/apache2/sites-available/horizon.conf

#. Enable the above configuration and restart the web server

   .. code-block:: console

     $ sudo a2ensite horizon
     $ sudo service apache2 restart

Next Steps
==========

* :ref:`install-settings` lists the available settings for horizon.
* :ref:`install-customizing` describes how to customize horizon.

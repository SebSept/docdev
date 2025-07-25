Upgrade to GLPI 11.0
--------------------

Removal of input variables auto-sanitize
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Prior to GLPI 11.0, PHP superglobals ``$_GET``, ``$_POST`` and ``$_REQUEST`` were automatically sanitized.
It means that SQL special characters were escaped (prefixed by a ``\``), and HTML special characters ``<``, ``>`` and ``&`` were encoded into HTML entities.
This caused issues because it was difficult, for some pieces of code, to know if the received variables were "secure" or not.

In GLPI 11.0, we removed this auto-sanitization, and any data, whether it comes from a form, the database, or the API, will always be in its raw state.

Protection against SQL injection
++++++++++++++++++++++++++++++++

Protection against SQL injection is now automatically done when DB query is crafted.

All the ``addslashes()`` usages that were used for this purpose have to be removed from your code.

.. code-block:: diff

   - $item->add(Toolbox::addslashes_deep($properties));
   + $item->add($properties);

Protection against XSS
++++++++++++++++++++++

HTML special characters are no longer encoded automatically. Even existing data will be seamlessly decoded when it will be fetched from database.
Code must be updated to ensure that all dynamic variables are correctly escaped in HTML views.

Views built with ``Twig`` templates no longer require usage of the ``|verbatim_value`` filter to correctly display HTML special characters.
Also, Twig automatically escapes special characters, which protects against XSS.

.. code-block:: diff

   - <p>{{ content|verbatim_value }}</p>
   + <p>{{ content }}</p>

Code that outputs HTML code directly must be adapted to use the ``htmlescape()`` function.

.. code-block:: diff

   - echo '<p>' . $content . '</p>';
   + echo '<p>' . htmlescape($content) . '</p>';

Also, code that ouputs javascript must be adapted to prevent XSS by escaping both the HTML code with the ``htmlescape()`` function
and the JS variables with the ``jsescape()`` function.

.. code-block:: diff

   echo '
       <script>
   -        $(body).append('<p>' . $content . '</p>');
   +        $(body).append("' . jsescape('<p>' . htmlescape($content) . '</p>') . '"');
       </script>
   ';

.. note::

   Both the ``htmlescape()`` and the ``jsescape()`` functions have been added to ease the migration to GLPI 11.0
   but will be deprecated and removed when the GLPI HTML and JS code will be completely moved into ``Twig`` templates and JS files.


Query builder usage
+++++++++++++++++++

Since it has been implemented, internal query builder (named ``DBMysqlIterator``) do accept several syntaxes; that make things complex:

1. conditions (including table name as ``FROM`` array key) as first (and only) parameter.
2. table name as first parameter and condition as second parameter,
3. raw SQL queries,

The most used and easiest to maintain was the first. The second has been deprecated and the thrird has been prohibited or security reasons.

If you were using the second syntax, you will need to replace as follows:

.. code-block:: diff

   - $iterator = $DB->request('mytable', ['field' => 'condition']);
   + $iterator = $DB->request(['FROM' => 'mytable', 'WHERE' => ['field' => 'condition']]);

Using raw SQL queries must be replaced with query builder call, among other to prevent syntax issues, and SQL injections; please refer to :doc:devapi/database/dbiterator.

Changes related to web requests handling
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In GLPI 11.0, all the web requests are now handled by a unique entry point, the ``/public/index.php`` script.
This allowed us to centralize a large number of things, including GLPI's initialization mechanics and error management.

Removal of the ``/inc/includes.php`` script
+++++++++++++++++++++++++++++++++++++++++++

All the logic that was executed by the inclusion of the ``/inc/includes.php`` script is now made automatically.
Therefore, it is no longer necessary to include it, even if it is still present to ease the migration to GLPI 11.0.

.. code-block:: diff

   - include("../../../inc/includes.php");

Resource access restrictions
++++++++++++++++++++++++++++

In GLPI 11.0, we restrict the resources that can be accessed through a web request.

To ease the migration to GLPI 11.0, we still support public access to the PHP scripts located in the ``/ajax``, ``/front`` and ``/report`` directories,
and their URL remains unchanged.

All static assets or other PHP scripts that must be accessible through a web request must be moved in the ``/public`` directory.
The ``/public`` part of the path must not be present in their URL, for instance:

* the URL of the ``/public/css/styles.css`` stylesheet of your plugin will be ``/plugins/myplugin/css/styles.css``;
* the URL of the ``/public/mypluginapi.php`` script of your plugin will be ``/plugins/myplugin/mypluginapi.php``.

Legacy scripts access policy
++++++++++++++++++++++++++++

By default, the access to any PHP script will be allowed only to authenticated users.
If you need to change this default policy for some of your PHP scripts, you will need to do this in your plugin ``init`` function,
using the ``Glpi\Http\Firewall::addPluginStrategyForLegacyScripts()`` method.

.. code-block:: php

   <?php
   
   use Glpi\Http\Firewall;
   
   function plugin_init_myplugin() {
       Firewall::addPluginStrategyForLegacyScripts('myplugin', '#^/front/faq.php$#', Firewall::STRATEGY_FAQ_ACCESS);
       Firewall::addPluginStrategyForLegacyScripts('myplugin', '#^/front/dashboard.php$#', Firewall::STRATEGY_CENTRAL_ACCESS);
   }

The following strategies are available:

* ``Firewall::STRATEGY_NO_CHECK``: no check is done, anyone can access your script, even unauthenticated users;
* ``Firewall::STRATEGY_AUTHENTICATED``: only authenticated users can access your script, it is the default strategy for all PHP scripts;
* ``Firewall::STRATEGY_CENTRAL_ACCESS``: only users with access to the standard interface can access your script;
* ``Firewall::STRATEGY_HELPDESK_ACCESS``: only users with access to the simplified interface can access your script;
* ``Firewall::STRATEGY_FAQ_ACCESS``: only users with a read access to the FAQ will be allowed to access your script, unless the FAQ is configured to be public.

Stateless endpoints
+++++++++++++++++++

By default, GLPI will automatically start the PHP session, and use a session cookie to share the current session ID
between web requests. If there is no active session, it will redirect the client to the login page.
This behaviour should be disabled for stateless endpoints, such as APIs endpoints.
To do this, you will need to call the ``\Glpi\Http\SessionManager::registerPluginStatelessPath()`` method from the ``boot`` hook of your plugin,
located in the ``setup.php`` file.

.. code-block:: php

   <?php
   
   use Glpi\Http\SessionManager;
   
   function plugin_init_myplugin() {
       SessionManager::registerPluginStatelessPath('myplugin', '#^/front/api.php/#');
   }


Handling of response codes and early script exit
++++++++++++++++++++++++++++++++++++++++++++++++

Usage of the ``exit()``/``die()`` language construct is now discouraged as it prevents the execution of routines that might take place after the request has been executed.
Also, due to a PHP bug (see https://bugs.php.net/bug.php?id=81451), the usage of the ``http_response_code()`` function will produce unexpected results, depending on the server environment.

In the case they were used to exit the script early due to an error, you can replace them by throwing an exception.
Any exception thrown will now be caught correctly and forwarded to the error handler.
If this exception is thrown during the execution of a web request, the GLPI error page will be shown, unless this exception is handled by a specific routine.

.. code-block:: diff

   if ($item->getFromDB($_GET['id']) === false) {
   -    http_response_code(404);
   -    exit();
   +    throw new \Glpi\Exception\Http\NotFoundHttpException();
   }

In case the ``exit()``/``die()`` language construct was used to just ignore the following line of code in the script, you can replace it with a ``return`` instruction.

.. code-block:: diff

   if ($action === 'foo') {
       // specific action
       echo "foo action executed";
   -    exit();
   +    return;
   }
   
   MypluginItem::displayFullPageForItem($_GET['id']);

Crafting plugins URLs
+++++++++++++++++++++

We changed the way to handle URLs to plugin resources so that they no longer need to reflect the location of the plugin on the file system.
For instance, the same URL could be used to access a plugin file whether it was installed manually in the ``/plugins`` directory or via the marketplace.

To maintain backwards compatibility with previous behavior, we will continue to support URLs using the ``/marketplace`` path prefix.
However, their use is deprecated and may be removed in a future version of GLPI.

The ``Plugin::getWebDir()`` PHP method has been deprecated.

.. code-block:: diff

   - $path = Plugin::getWebDir('myplugin', false) . '/front/myscript.php';
   + $path = '/plugins/myplugin/front/myscript.php';
   
   - $path = Plugin::getWebDir('myplugin', true) . '/front/myscript.php';
   + $path = $CFG_GLPI['root_doc'] . '/plugins/myplugin/front/myscript.php';

The ``GLPI_PLUGINS_PATH`` javascript variable has been deprecated.

.. code-block:: diff

   - var url = CFG_GLPI.root_doc + '/' + GLPI_PLUGINS_PATH.myplugin + '/ajax/script.php';
   + var url = CFG_GLPI.root_doc + '/plugins/myplugin/ajax/script.php';

The ``get_plugin_web_dir`` Twig function has been deprecated.

.. code-block:: diff

   - <form action="{{ get_plugin_web_dir('myplugin') }}/front/config.form.php" method="post">
   + <form action="{{ path('/plugins/myplugin/front/config.form.php') }}" method="post">

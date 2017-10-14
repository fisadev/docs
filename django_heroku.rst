Deployar django 1.10 a Heroku
============================

Asumimos que
============

* Ya tenés tu proyecto django, que funciona localmente (o sea, hacés un ``runserver`` y podés usarlo en tu máquina).
* Estás usando django 1.10 (o posiblemente versiones más nuevas)
* Tu proyecto django está en un repositorio git.
* Tenés un ``requirements.txt`` en **la raiz** de tu repo, con las dependencias python de tu proyecto, donde figura django y cualquier otra cosa que haga falta instalar con pip para que funcione, y que se puede usar con un ``pip install -r requirements.txt``. (Recordá que es posible especificar las versiones de tus dependencias en el ``requirements.txt``. Por ejemplo, ``Django==1.10``. Con ``pip freeze`` podés consultar las versiones que tenés instaladas actualmente).
* Te hiciste una cuenta en `Heroku <http://heroku.com>`_ y recordás tu usuario y contraseña.


Instalar dependencias
=====================

Primero que nada se necesitan las herramientas de heroku, y estas están hechas con ruby. En un ubuntu, basta con hacer:

.. code-block:: bash

    sudo apt-get install ruby
    wget -qO- https://toolbelt.heroku.com/install-ubuntu.sh | sh


Si esto se instaló bien deberías poder ejecutar lo siguiente, que te pida tus credenciales, y loguearte:

.. code-block:: bash

    heroku login


Y ahora las cosas que vamos a usar desde dentro de django para "engancharlo" con heroku (si estás usando virtualenv, esto se corre **sin** ``sudo``):

.. code-block:: bash

    sudo pip install django-toolbelt whitenoise


**Además** de instalarlo, agreguá ``django-toolbelt`` y ``whitenoise`` al ``requirements.txt``.


Crear archivo Procfile
======================

Este archivo es el que le dice a heroku cómo ejecutar nuestro servidor web.

Crea un archivo que se llame ``Procfile`` (**respetá** el nombre, exactamente como dice ahí) en la raiz de tu **repositorio**, y rellenalo con esto:

.. code-block::

    web: gunicorn miproyecto.wsgi --log-file -


Modificá ese archivo, y donde dice ``miproyecto``, poné el nombre de tu proyecto django (es el nombre de la carpeta donde está el ``settings.py``).

Si tu proyecto django no está en la raiz de tu repositorio, modificá ese archivo, agregando el directorio de tu proyecto dentro del repo de esta forma:


.. code-block::

    web: cd directorio/del/proyecto && gunicorn miproyecto.wsgi --log-file -


**Si estás usando python 3**, además creá un segundo archivo llamado ``runtime.txt`` en la misma ubicación que ``Procfile``, con este contenido:


.. code-block::

    python-3.6.1


Para probar si tu ``Procfile`` (y ``runtime.txt``) funciona correctamente, ubicate en la raiz de tu **repositorio** y ejecutá esto:

.. code-block:: bash

    heroku local web


No debería mostrar ningún error, y la última linea del resultado debería decir algo como esto:

.. code-block::

    [INFO] Booting worker with pid: 25416


Si anduvo y sigue corriendo, cerralo con ``Ctrl+c``.


Modificaciones a las settings de nuestro proyecto
=================================================

Es necesario configurar algunas settings de nuestro proyecto (``settings.py``) para que corra bien dentro de heroku.

Primero que nada, asegurate de tener seteada la setting ``STATIC_ROOT``. Si no la tenés, deberías agregarla con algo como esto:

(se suele agregar debajo de la setting ``STATIC_URL``, porque están muy relacionadas)

.. code-block:: python

    STATIC_ROOT = os.path.join(BASE_DIR, 'static_server_files')


Y agregá al final de tu ``settings.py`` esto:

.. code-block:: python

    if os.environ.get('HEROKU', False):
        # settings especificas para heroku
        import dj_database_url
        DATABASES['default'] = dj_database_url.config()
        ALLOWED_HOSTS = ['*']
        SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
        MIDDLEWARE.insert(0, 'whitenoise.middleware.WhiteNoiseMiddleware')
        STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'


Dentro de ese mismo if también podés customizar cualquier setting que quieras que tenga un valor distinto al correr en heroku (ej: ``DEBUG = False``, etc.).

Modificar el WSGI de nuestro proyecto
=====================================

Y por último, hay que modificar el archivo ``wsgi.py`` que está junto al ``settings.py``, que es el archivo que se utiliza para conectar django con el server web. Abrilo, borrá la línea que dice:

.. code-block:: python

    application = get_wsgi_application()


Y en su lugar poné esto:

.. code-block:: python

    if os.environ.get('HEROKU', False):
        from dj_static import Cling
        application = Cling(get_wsgi_application())
    else:
        application = get_wsgi_application()


Crear sitio (aplicación) en heroku por primera vez
==================================================

Tu proyecto ya está listo, solo queda decirle a heroku que lo levante.

Primero (y esto lo hacemos solo una vez), creamos una aplicación en heroku. Para eso, ubicate en la **raiz de tu repo**, y ejecutá:

.. code-block:: bash

    heroku create nombre-de-tu-proyecto


Reemplazando ``nombre-de-tu-proyecto`` por el nombre que quieras que tu app tenga en Heroku.


Y además vamos a setear una configuración en el server para que nuestro django se de cuenta de que está dentro de heroku:

.. code-block:: bash

    heroku config:set HEROKU=1


Actualizar y correr nuestro sitio
=================================

Y ahora podemos mandar el código de nuestro sitio, y heroku lo va a levantar de forma automática:

.. code-block:: bash

    git push heroku master


Si mirás bien toda la salida de eso (y no falló nada), vas a ver que en un punto dice algo como esto:

.. code-block::

    remote: -----> Launching... done, v7
    remote:        https://lit-ridge-5779.herokuapp.com/ deployed to Heroku


(En tu proyecto seguramente van a cambiar algunos números y nombres)
Entrando a esa url, si todo funcionó bien, deberias ver tu sitio andando :)

Cada vez que modifiques tu código, simplemente commitealo y después ejecutá ese mismo push para que heroku tome los cambios y reinicie el servidor.


IMPORTANTE: cosas que seguro vas a necesitar hacer
==================================================

Un último detalle: seguramente tu aplicación falló por no tener la base de datos creada y actualizada. Para correr las migrations de django en el server, simplemente hacelo con:

.. code-block:: bash

    heroku run "python manage.py migrate"


Si tu proyecto no está en la raiz del repo, agregá un ``cd`` al directorio de tu proyecto, así:

.. code-block:: bash

    heroku run "cd directorio/del/proyecto && python manage.py migrate"


Recordá que siempre que hagas cambios a la db, vas a tener que correr las migrations en el servidor **después** de pushear tus cambios.


Cosas útiles
============

Podés ver los logs de la aplicación corriendo:

.. code-block:: bash

    heroku logs

También podés probar la aplicación antes de mandarla al sitio con:

.. code-block:: bash

    heroku local web

Con esto se va a levantar localmente luego de bajar un plugin, de manera automática.


Con el comando ``heroku run`` podés correr comandos arbitrarios en tu server, y ver la salida.

Y desde `el panel de heroku <https://dashboard.heroku.com/apps>`_ podés ver mucha más info y administrar tu aplicación.

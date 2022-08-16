Deployar django 3.0 a Heroku
============================

Asumimos que
============

* Ya tenés tu proyecto django, que funciona localmente (o sea, hacés un ``runserver`` y podés usarlo en tu máquina).
* Estás usando django 3.0 (o posiblemente versiones más nuevas)
* Estás usando python 3
* Tu proyecto django está en un repositorio git.
* Tenés un ``requirements.txt`` en **la raiz** de tu repo, con las dependencias python de tu proyecto, donde figura django y cualquier otra cosa que haga falta instalar con pip para que funcione, y que se puede usar con un ``pip install -r requirements.txt``. (Recordá que es posible especificar las versiones de tus dependencias en el ``requirements.txt``. Por ejemplo, ``Django==3.0``. Con ``pip freeze`` podés consultar las versiones que tenés instaladas actualmente).
* Te hiciste una cuenta en `Heroku <http://heroku.com>`_ y recordás tu usuario y contraseña.


Instalar dependencias
=====================

Primero que nada se necesitan las herramientas de heroku. En un ubuntu, basta con hacer:

.. code-block:: bash

    sudo snap install --classic heroku


Si esto se instaló bien deberías poder ejecutar lo siguiente, que te pida tus credenciales, y loguearte:

.. code-block:: bash

    heroku login


Y ahora las cosas que vamos a usar desde dentro de django para "engancharlo" con heroku (si estás usando virtualenv, esto se corre **sin** ``sudo``):

.. code-block:: bash

    pip3 install django-heroku gunicorn --user
   
   
O si estás usando un virtualenv, lo mismo pero sin "3" y sin "--user".


**Además** de instalarlos, agreguá ``django-heroku`` y ``gunicorn`` al ``requirements.txt``.


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


Además creá un segundo archivo llamado ``runtime.txt`` en la misma ubicación que ``Procfile``, con este contenido:


.. code-block::

    python-3.8.11


Para probar si tu ``Procfile`` (y ``runtime.txt``) funciona correctamente, ubicate en la raiz de tu **repositorio** y ejecutá esto:

.. code-block:: bash

    heroku local web


No debería mostrar ningún error, y la última linea del resultado debería decir algo como esto:

.. code-block::

    [INFO] Booting worker with pid: 25416


No hace falta entrar al link que nos muestra ni chequear nada en el sitio, solo validar que el comando anduvo. Si anduvo y sigue corriendo, cerralo con ``Ctrl+c``.


Modificaciones a las settings de nuestro proyecto
=================================================

Es necesario configurar algunas settings de nuestro proyecto (``settings.py``) para que corra bien dentro de heroku.

Por suerte el paquete ``django-heroku`` hace esto de forma automágica. Solo tenemos que hacer dos cosas:

Agregar esto al inicio de nuestro ``settings.py`` (después de los imports ya existentes):

.. code-block:: python

    import django_heroku


Y agregar esto al final del settings:

.. code-block:: python

    django_heroku.settings(locals())


Crear sitio (aplicación) en heroku por primera vez
==================================================

Tu proyecto ya está listo, solo queda decirle a heroku que lo levante.

Simplemente creamos una aplicación en heroku (y esto lo hacemos solo una vez). Para eso, ubicate en la **raiz de tu repo**, y ejecutá:

.. code-block:: bash

    heroku create nombre-de-tu-proyecto


Reemplazando ``nombre-de-tu-proyecto`` por el nombre que quieras que tu app tenga en Heroku.

Y luego, le pedimos a heroku poder usar una base de datos dentro de ese mismo proyecto:

.. code-block:: bash

    heroku addons:create heroku-postgresql:hobby-dev


(``hobby-dev`` es el nombre del plan gratuito de postgresql en heroku. Si queremos, podemos elegir otro que estemos dispuestos a pagar)

Actualizar y correr nuestro sitio
=================================

Y ahora podemos mandar el código de nuestro sitio, y heroku lo va a levantar de forma automática:

.. code-block:: bash

    git push heroku main


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


Con el comando ``heroku run`` podés correr comandos arbitrarios en tu server, y ver la salida, e incluso interactuar (por ejemplo, probablemente lo necesites para correr tu creación del superuser).

Y desde `el panel de heroku <https://dashboard.heroku.com/apps>`_ podés ver mucha más info y administrar tu aplicación.

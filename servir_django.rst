Las cosas que vamos a usar
--------------------------

* **Django**: lo que tiene la lógica de nuestra app
* **Gunicorn**: lo que sabe ejecutar un servidor con nuestra app, que se pueda bancar tráfico de producción
* **Nginx**: lo que expone nuestro servidor hacia el exterior, porque es rápido y seguro (gunicorn no tienen nada de seguridad, por eso queda "escondido" adentro)
* **Supervisor**: lo que levanta y mantiene corriengo gunicorn como servicio

Asumimos que:
-------------

* Tenes una db andando e instalada
* Estás logueado como root o podés correr las cosas como sudo
* Cuando escribamos algo en MAYUSCULAS, quiere decir que eso **no** va literal, si no que tenes que reemplazarlo por algo.

Crear un directorio donde va a ir todo
--------------------------------------

.. code-block:: bash

    mkdir /opt/NOMBRE_APP_O_COMO_TE_GUSTE

A partir de ahora, a esta carpeta nos vamos a referir como RAIZ_APP.

Entrar al directorio
--------------------

.. code-block:: bash

    cd RAIZ_APP

Crear virualenv y activarlo
---------------------------

(tenes que estar en el directorio RAIZ_APP)

.. code-block:: bash

    virtualenv venv
    source venv/bin/activate

Clonar/bajar el código del proyecto django
------------------------------------------

(tenes que estar en el directorio RAIZ_APP)

.. code-block:: bash

    git clone https://MI_REPO...

Instalar dependencias que el proyecto tenga
-------------------------------------------

(tenes que tener activado el virtualenv)

.. code-block:: bash

    pip install LOS_PAQUETES_QUE_TU_PROYECTO_REQUIERA

Consejo: siempre es bueno tener en el repo un archivo ``requirements.txt`` donde en cada renglón ponemos el nombre de un paquete, para después poder instalarlos haciendo:

.. code-block:: bash

    pip install -r TU_ARCHIVO_REQUIREMENTS.TXT

Instalar django, gunicorn
-------------------------

(tenes que tener activado el virtualenv)

.. code-block:: bash

    pip install django gunicorn

Consejo: ponelos en tu requirements.txt, así ni siquiera tenés que hacer este paso a mano, y se instalan solos en el paso 6)

Probar si un runserver normal anda
----------------------------------

(tenes que tener activado el virtualenv)

Ir a la raiz del proyecto django y correr:

.. code-block:: bash

    python manage.py 0.0.0.0:8000

Entrar por el navegador a IP_DEL_SERVIDOR:8000

Pero seguramente para que tu sitio ande bien vas a tener que hacer un ``syncdb`` para crear las tablas, configurar la base de datos, etc.
Resolvé en este punto todas esas cosas hasta que al hacer un runserver y entrar por navegador, tu sitio ande bien.
Cuando ande bien, matá el runserver con Ctrl-c, no lo vamos a necesitar más.

Probar si gunicorn anda
-----------------------

(tenes que tener activado el virtualenv)

Ir a la raiz del proyecto django y correr:

.. code-block:: bash

    gunicorn NOMBREPROYECTO.wsgi:application -b 0.0.0.0:8000

Si anda bien, no va a mostrar nada, simplemente va a quedar la consola con eso corriendo.

Entrar por el navegador a IP_DEL_SERVIDOR:8000

Deberías ver tu sitio andando bien, salvo que no vas a ver todos los estáticos (no va a cargar los estilos css, imágenes, etc). Eso está bien, después vamos a servirlos con nginx.
Si anda bien, matalo nomás con Ctrl-c, a partir de ahora ya no lo vamos a correr a mano.

Instalar supervisor y nginx
---------------------------

**IMPORTANTE**: antes desactivar el virtualenv corriendo:

.. code-block:: bash

    deactivate

Instalar supervisor y nginx corriendo:

.. code-block:: bash

    apt-get install supervisor nginx

Crear configuración de supervisor
---------------------------------

Dentro de ``/etc/supervisor/conf.d/`` crear un archivo que se llame ``NOMBRE_APP_O_COMO_TE_GUSTE.conf``

Y rellenarlo con este contenido:

.. code-block::

    [program:NOMBRE_APP_O_COMO_TE_GUSTE]
    directory=RAIZ_A_TU_PROYECTO_DJANGO
    user=root
    command=RAIZ_APP/gunicorn.sh
    stdout_logfile=RAIZ_APP/supervisor_stdout.log
    stderr_logfile=RAIZ_APP/supervisor_stderr.log
    autostart=true
    autorestart=true
    stopasgroup=true

Crear script para gunicorn
--------------------------

Dentro de RAIZ_APP crear un archivo que se llame ``gunicorn.sh``

Y rellenarlo con este contenido:

.. code-block:: bash

    #!/bin/bash
    cd RAIZ_A_TU_PROYECTO_DJANGO
    source RAIZ_APP/venv/bin/activate
    RAIZ_APP/venv/bin/gunicorn NOMBREPROYECTO.wsgi:application -t 600 -b 127.0.0.1:8000 -w CANTIDAD_DE_WORKERS --user=root --group=root --log-file=RAIZ_APP/gunicorn.log 2>>RAIZ_APP/gunicorn.log

Recordá reemplazar todo lo que está con MAYUSCULAS.
CANTIDAD_DE_WORKERS es la cantidad de workers gunicorn que queremos correr. Para un server físico, normalmente se usa el doble de la cantidad de núcleos que tiene el procesador + 1 (ej: si tiene 4 núcleos, ponemos 9). Para algo como un amazon o digital ocean, con 3 estaríamos bien.

Despues de crear el archivo, darle permisos de ejecución:

.. code-block:: bash

    cd RAIZ_APP
    chmod +x gunicorn.sh

Crear configuración de nginx
----------------------------

Creamos un archivo dentro de ``/etc/nginx/sites-enabled/`` que se llame ``NOMBRE_APP_O_COMO_TE_GUSTE`` (sin extensión).

Y rellenarlo con este contenido:

.. code-block:: nginx

    server {
        listen 80;
        server_name DOMINIO_DE_TU_SITIO_WEB_O_IP;

        # Permitir subir archivos de hasta 50M
        client_max_body_size 50M;
        
        root RAIZ_A_TU_PROYECTO_DJANGO;

        access_log RAIZ_APP/nginx_access.log;
        error_log RAIZ_APP/nginx_error.log;
        
        location /media/ {
        }
        location /static/ {
        }

        location / {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_connect_timeout 600;
            proxy_send_timeout 600;
            proxy_read_timeout 600;
            proxy_pass http://localhost:8000/;
        }
    }


Hacer collect static de django
------------------------------

Es necesario que tu proyecto django tenga seteadas estas settings (en tu ``settings.py``):

.. code-block:: python

    STATIC_ROOT = 'RAIZ_A_TU_PROYECTO_DJANGO/static'
    MEDIA_ROOT = 'RAIZ_A_TU_PROYECTO_DJANGO/media'

Reemplazando RAIZ_APP como siempre, pero ``STATIC_ROOT`` y ``MEDIA_ROOT`` no son cosas que tengas que reemplazar, son settings de django que se llaman así.

Con las settings bien puestas, activar el virtualenv y correr el collectstatic de django:

.. code-block:: bash

    source RAIZ_APP/venv/bin/activate
    cd RAIZ_A_TU_PROYECTO_DJANGO
    python manage.py collectstatic

Restartear nginx y supervisor para que tomen las configs y corran todo
----------------------------------------------------------------------

Correr:

.. code-block:: bash

    service supervisor restart
    service nginx restart

Cada vez que toques código
--------------------------

A partir de ahora, cada vez que toques cosas en tu código, para que el servidor tome los cambios tenés que hacer esto:

* Te traes los cambios al servidor (con hg, git, etc).

* Activas virtualenv

.. code-block:: bash

    source RAIZ_APP/venv/bin/activate

* Corres un collectstatic

.. code-block:: bash

    cd RAIZ_A_TU_PROYECTO_DJANGO
    python manage.py collectstatic

* Restarteas supervisor 

.. code-block:: bash

    service supervisor restart

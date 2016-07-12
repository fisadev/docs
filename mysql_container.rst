Nunca les molestó tener que instalarse y configurarse un mysql en la máquina para trabajar en un proyecto que lo necesite?

Con esto en un ratito pueden tener un mysql de la versión que quieran andando con docker, con la db creada automáticamente, usuario con permisos, etc. Y es re simple, casi sin hacer nada.
(para uso solo en desarrollo)

La primera vez:

1. instalar docker si no lo tienen de antes
2. ``mkdir datos_server``
3. ``docker run --name mi_server -v /path/a/tu/datos_server:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=toor -e MYSQL_DATABASE=espadas -e MYSQL_USER=fisa -e MYSQL_PASSWORD=paparulo -d mysql/mysql-server:5.6``

Ya está! Eso les creó el server y se los inició, ya está corriendo con la db creada, usuario con acceso y todo.

A partir de ahora no hace falta repetir los pasos anteriores. Con esto pueden levantar y matar el server:

.. code:: bash

    docker start mi_server
    docker stop mi_server


Y para conectarse desde algún cliente de mysql, simplemente apunten a la ip del container, con el usuario, password y db que especificaron en el paso 3. La ip se consulta con ``docker inspect mi_server | grep IPAddress`` 

Usar Haystack localmente indices en Whoosh
==========================================

Asumimos que
============

* Ya tenés tu proyecto django, que funciona localmente (o sea, hacés un ``runserver`` y podés usarlo en tu máquina).
* Estás usando django 3.0.x o superior.
* Tenés un ``requirements.txt`` con las dependencias python de tu proyecto, donde figura django y cualquier otra cosa que haga falta instalar con pip para que funcione, y que se puede usar con un ``pip install -r requirements.txt``. (Recordá que es posible especificar las versiones de tus dependencias en el ``requirements.txt``. Por ejemplo, ``Django==1.9``. Con ``pip freeze`` podés consultar las versiones que tenés instaladas actualmente).

Instalar dependencias
=====================

Primero que nada se necesitan instalar haystack y whoosh:

.. code-block:: bash

    pip3 install whoosh django-haystack --user

   
O si estás usando un virtualenv, lo mismo pero sin "3" y sin "--user".

**Además** de instalarlo, agreguá ``django-haystack`` y ``whoosh`` al ``requirements.txt``.


Configs básicas en django
=========================

Ahora necesitás decirle a django dos cosas. Por un lado, que vas a usar haystack. Para eso agregalo a tu setting ``INSTALLED_APPS``, y que esté **después** de las cosas de django, pero **antes** de tus aplicaciones propias. Así:

.. code-block:: python

    INSTALLED_APPS = (
        # todas las apps de django:
        'django.contrib.messages',
        'django.contrib.staticfiles',
        # etc...

        'haystack',

        # todas tus apps:
        'sitio',
        # etc...
    )


Lo segundo es decirle a django **cómo** vamos a usar haystack, sus settings (haystack puede usarse de muchas formas). Para eso agregá debajo de ``INSTALLED_APPS``, una nueva setting donde vamos a configurar haystack:

.. code-block:: python

    HAYSTACK_CONNECTIONS = {
        'default': {
            'ENGINE': 'haystack.backends.whoosh_backend.WhooshEngine',
            'PATH': BASE_DIR / 'whoosh_index',
        },
    }


(Asumiendo que tu ``settings.py`` ya está definida la variable BASE_DIR, apuntando al directorio raíz de tu proyecto. Si no existe, basta con crearla apuntando al directorio donde quieras que se guarden los índices)


Definiendo índices
==================

Teniendo haystack instalado y configurado, ahora vamos a usarlo para indexar cosas. Para eso tenemos que definir qué cosas queremos indexar de nuestro sitio. 
Esto se hace de forma bastante similar a como se configura el admin de django: con clases que contienen las settings (de indexado) para cada modelo.
Estas clases tienen que definirse dentro de un archivo llamado ``search_indexes.py``, en la carpeta de la **aplicación**.

Imaginemos un modelo ``Noticia`` de ejemplo, con campos:

* ``titulo`` y ``descripcion``, de texto
* ``fecha``
* ``archivada``, booleano
* ``categoria``, relación a otro modelo ``Categoria``
  
Para indexar y poder buscar en las noticias, nuestro ``search_indexes.py`` podría contener esto:

.. code-block:: python

    from haystack import indexes
    from sitio.models import Noticia


    class NoticiaIndex(indexes.SearchIndex, indexes.Indexable):
        text = indexes.CharField(document=True, use_template=True)
        titulo = indexes.CharField(model_attr='titulo')
        fecha = indexes.DateTimeField(model_attr='fecha')

        def get_model(self):
            return Noticia

        def index_queryset(self, using=None):
            """Queremos que se indexen todas las noticias que tengan archivada=False"""
            return self.get_model().objects.filter(archivada=False)


Con esto le estamos diciendo a haystack:

* Queremos indexar noticias
* Queremos que el índice se arme a partir de un template, de ese template se va a extraer el texto a indexar (``text = ...``)
* Queremos que en el índice se guarde también el título original y la fecha de la noticia, para poder ordenar resultados de búsqueda, etc sin tener que ir a leer los objetos de la tabla de noticias (``titulo = ...`` y ``fecha = ...``)
* Queremos que solo se indexen las noticias que no fueron archivadas (``def index_queryset...``)

Usamos un template para poder indexar no solo el texto de un campo en particular, sino un "gran texto" armado como queramos, con varios campos, cosas extras, y todo lo que necesitemos.
Como en ``text = ...`` dijimos que íbamos a usar un template, tenemos que definirlo. Creamos un archivo ``sitio/templates/search/indexes/sitio/noticia_text.txt``, y dentro ponemos este contenido:

.. code-block::

    {{ object.titulo }}
    {{ object.categoria.nombre }}
    {{ object.descripcion }}


De esa forma, vamos a indexar noticias no solo usando su titulo y descripción, sino también el nombre de la categoría a la que pertenecen.
Si alguien busca "policiales", una noticia dentro de la categoría con nombre "policiales" también va a ser encontrada, por más que su título y descripción no tengan esa palabra dentro.

Indexar
=======

Haystack ya sabe qué queremos indexar y cómo. Ahora solo le pedimos que cree los índices:

.. code-block:: bash

    python manage.py rebuild_index


Y listo! nuestras noticias están indexadas, ahora podemos hacer búsquedas de texto completo.

Buscar
======

Para poder buscar, necesitamos una vista que reciba texto del usuario, ejecute la búsqueda usando haystack, y devuelva los resultados.
Por suerte haystack ya tiene esa vista armada, simplemente tenemos que incluirla, y definirle un template para que use.

Primero agregamos a nuestras urls:

.. code-block:: python

    path('search/', include('haystack.urls')),


Y luego agregamos nuestro template de búsqueda en ``sitio/templates/search/search.html``, con este contenido:

(esto asume que tenemos un template ``base.html`` del que los demás templates heredan, y que ese template tiene un bloque llamado ``contenido``)

.. code-block::

    {% extends 'base.html' %}

    {% block contenido %}
        <h2>Buscar:</h2>

        <form method="get" action=".">
            <table>
                {{ form.as_table }}
                <tr>
                    <td>&nbsp;</td>
                    <td>
                        <input type="submit" value="Buscar">
                    </td>
                </tr>
            </table>

            {% if query %}
                <h3>Resultados:</h3>

                {% for result in page.object_list %}
                    <p>{{ result.titulo }}, {{ result.fecha }}</p>
                {% empty %}
                    <p>No se encontraron noticias.</p>
                {% endfor %}

                {% if page.has_previous or page.has_next %}
                    <div>
                        {% if page.has_previous %}<a href="?q={{ query }}&amp;page={{ page.previous_page_number }}">{% endif %}&laquo; Previous{% if page.has_previous %}</a>{% endif %}
                        |
                        {% if page.has_next %}<a href="?q={{ query }}&amp;page={{ page.next_page_number }}">{% endif %}Next &raquo;{% if page.has_next %}</a>{% endif %}
                    </div>
                {% endif %}
            {% endif %}
        </form>
    {% endblock %}


Listo! Ahora podemos hacer búsquedas entrando a la url ``/search/``.

Actualizar índices
==================

Cuando los datos cambian, hay que actualizar los índices. Se puede hacer de muchas formas, pero la más básica es correr:

.. code-block:: bash

    python manage.py update_index


Bonus: En Heroku
================

En Heroku podemos usar haystack, a partir de un addon que provee un motor de búsqueda bastante bueno (aunque configurado para inglés y con poca posibilidad de modificarle settings).
Para usarlo, lo primero que necesitamos es agregar el addon a nuestra aplicación de heroku. Dentro de nuestro proyecto, corremos:

.. code-block:: bash

    heroku addons:create searchbox:starter --es_version=2


Y además, van a necesitar un par de dependencias nuevas. Agreguen esto al ``requirements.txt``:

.. code-block::

    elasticsearch
    certifi


Y luego modificamos nuestro ``settings.py``, agregando esto al final:

.. code-block:: python

    if os.environ.get('SEARCHBOX_URL'):
        HAYSTACK_CONNECTIONS = {
            'default': {
                'ENGINE': 'haystack.backends.elasticsearch_backend.ElasticsearchSearchEngine',
                'URL': os.environ.get('SEARCHBOX_URL'),
                'INDEX_NAME': 'documents',
            },
        }


Con eso ya podemos usar haystack en heroku. Pero recuerden que además de pushear para hacer el deploy, van a tener que correr los comandos para crear los índices en heroku, usando ``heroku run`` como vimos en la doc de deploy.

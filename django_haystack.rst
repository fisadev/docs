Usar Haystack localmente indices en Whoosh
==========================================

Asumimos que
============

* Ya tenés tu proyecto django, que funciona localmente (o sea, hacés un ``runserver`` y podés usarlo en tu máquina).
* Estás usando django 5.0.x o superior.
* Estás administrando las dependencias de tu proyecto con `uv <https://docs.astral.sh/uv/getting-started/installation/>`_ (la nueva herramienta recomendada para manejar dependencias en python, que reemplaza a pip/pipenv/poetry). Si no estás usando uv, las mismas cosas se pueden hacer con cualquier mecanismo de manejo de dependencias, pero los ejemplos de este tutorial son con uv.

Instalar dependencias
=====================

Primero que nada se necesitan agregar haystack y whoosh como dependencias:

.. code-block:: bash

    uv add whoosh django-haystack

   
Luego de hacer eso, no olvides commitear tu ``pyproject.toml`` y ``uv.lock``.


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

    uv run manage.py rebuild_index


Y listo! nuestras noticias están indexadas, ahora podemos hacer búsquedas de texto completo.

Buscar
======

Para poder buscar, necesitamos una vista que reciba texto del usuario, ejecute la búsqueda usando haystack, y devuelva los resultados.
Por suerte haystack ya tiene esa vista armada, simplemente tenemos que incluirla, y definirle un template para que use.

Primero agregamos a nuestras urls:

.. code-block:: python

    path('search/', include('haystack.urls')),


(quizás tengas que importar ``include``: ``from django.urls import path, include``)

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

    uv run manage.py update_index


En producción
=============

En un entorno productivo normal, cambiarían dos cosas importantes:

- Utilizar ElasticSearch o Solr en lugar de Whoosh. Esto no implica dejar de usar Haystack! Haystack soporta todos esos motores de búsqueda. Simplemente implica cambiar las configuraciones de Haystack en el ``settings.py``, y apuntar a una instancia de ElasticSearch o Solr que tengamos disponibles. Whoosh es bastante práctico para probar localmente, pero en general no es un motor lo suficientemente bueno para producción.
- Programar un indexado periódico, con crons, tareas con un schedule, o mecanismo similar. O alternativamente, indexar selectivamente cuando la data es guardada en nuestra base de datos.

Bonus: view para indexar
========================

Si estás corriendo tu aplicación en un servicio gratuito que no provee un mejor motor de búsqueda que whoosh, y que periódicamente borra los archivos de tu servidor, y que además no te deja taskear comandos periódicamente o acceder por shell para correr ``update_index``, podés crear una vista para indexar manualmente la data:

.. code-block:: python

    def rebuild_index(request):
        from django.core.management import call_command
        from django.http import JsonResponse
        try:
            call_command("rebuild_index", noinput=False)
            result = "Index rebuilt"
        except Exception as err:
            result = f"Error: {err}"

        return JsonResponse({"result": result})


Luego es posible exponer esa view en una url, por ejemplo:

.. code-block:: python

    path('rebuild_index/', views.rebuild_index),

Django REST framework
=====================

Namapování dat z pohledů na zdroje
----------------------------------

Pro namapování dat z pohledů na zdroje je jedním z řešení
vytvořit Django modely, pro ty vytvořit serializační třídy a pohledy.
Django REST framework umožňuje serializovat i data, která nepochází z modelů,
ale to by v tomto případě bylo zbytečně složité.

Jeden model, serializační třídu a pohled můžete vidět [v ukázce](#code:drf:mapping);
implementační detaily, jako importy, jsou vynechány pro lepší čitelnost.
Vzhledem k tomu, že je jednodušší rovnou některé položky přejmenovat,
je v této ukázce již tak učiněno; detailnější vysvětlení najdete v další části.

```{caption="{#code:drf:mapping}DRF: Namapování dat z pohledů na zdroje" .python}
# models.py
class Course(Model):
    id = SmallIntegerField(primary_key=True,
                           db_column='id_subjects')
    shortcut = StringField()
    day = SmallIntegerField()
    starts_at = TinyStringField(db_column='begin')
    ends_at = TinyStringField(db_column='end')
    notice = TextField()
    semester = SmallIntegerField()
    sport = ForeignKey(Sport, db_column='sport')
    hall = ForeignKey(Hall, db_column='hall')
    teacher = ForeignKey(Teacher, db_column='lector')

    class Meta:
        db_table = 'v_subjects'

# serializers.py
class CourseSerializer(HyperlinkedModelSerializer):
    class Meta:
        model = Course
        fields = tuple(
            f.name for f in model._meta.fields \
            if not f.name.startswith('_'))

# views.py
class CourseViewSet(ReadOnlyModelViewSet):
    '''
    API endpoint that allows courses to be viewed.
    '''
    queryset = Course.objects.all()
    serializer_class = CourseSerializer

# urls.py
router = DefaultRouter()
router.register(r'courses', CourseViewSet)
```

Některé kroky, například vytvoření serializační třídy, lze jednoduše automatizovat,
jak je vidět [z ukázky](#code:drf:serializer).
Podobným způsobem by šlo zautomatizovat i vytváření pohledů, ale vzhledem k tomu,
že dokumentační řetězce u pohledů se zobrazují ve webově procházetelném API,
je příhodnější nechat je definované jako jednotlivé třídy.

```{caption="{#code:drf:serializer}DRF: Automatizace vytvoření serializační třídy" .python}
def serializer(model_):
    '''Get a default Serializer class for a model'''
    class _Serializer(HyperlinkedModelSerializer):
        class Meta:
            model = model_
            fields = # ...
    return _Serializer


CourseSerializer = serializer(Course)
```

Namapování dat z pohledů na zdroje v Django REST frameworku je
možné,
systematické
a jednoduché, ale pro tento jednoduchý příklad zbytečně komplexní.

Přejmenování položek
--------------------

Pro přejmenování položek stačí jinak pojmenovat atribut a poskytnout konstruktoru argument `db_column` s názvem sloupce.
Ten je potřeba poskytnout u cizích klíčů i v případě, že se atribut jmenuje stejně jako položka, protože Django jinak očekává,
že se sloupec bude jmenovat `{field}_id`.

Konkrétní příklad přejmenování položek u kurzu můžete vidět [na začátku ukázky](#code:drf:mapping).

Přejmenování položek v Django REST frameworku je
možné,
systematické
a triviální.

Prolinkování zdrojů ve stylu HATEOAS
------------------------------------

Django REST framework při použití `HyperlinkedModelSerializer` automaticky serializuje cizí klíče jako odkazy.


Prolinkování zdrojů ve stylu HATEOAS v Django REST frameworku je
možné, automatické,
systematické
a triviální.

Navigační odkazy se vytvářejí rovněž automaticky.

Úprava zobrazených dat
----------------------

Jednou z variant, jak upravit zobrazená data, je vytvořit přímo v modelu metody,
které budou data měnit a místo původních dat serializovat výsledky těchto metod.
Příklad pro kód předmětu v KOSu můžete vidět [v ukázce](#code:drf:modify).

```{caption="{#code:drf:modify}DRF: Úprava zobrazených dat" .python}
class Enrollment(models.Model):
    # ...
    _kos_course_code = ShortStringField(db_column='kos_kod')
    _kos_code_flag = models.BooleanField(db_column='kos_code')

    @property
    def kos_course_code(self):
        return self._kos_course_code if self._kos_code_flag else None


class EnrollmentSerializer(HyperlinkedModelSerializer):
    class Meta:
        model = Enrollment
        fields = ('kos_course_code', ...) # no _kos_course_code
```

Úprava zobrazených dat v Django REST frameworku je
možná,
systematická
a triviální.

Zobrazení dat ve standardizované podobě
---------------------------------------

Django REST framework data zobrazuje ve velmi jednoduché podobě.
Pokud toto chceme změnit, je třeba vytvořit vlastní třídy zodpovědné za stránkování a prezentaci dat.

Naštěstí již existuje modul `drf-hal-json`, ve kterém existují dané třídy pro HAL sterilizaci,
jeho použití najdete [v ukázce](#code:drf:standard) a výstup [v ukázce](#code:drf:hal).
Existují i knihovny pro jiné serializace, např. `djangorestframework-jsonapi` pro JSON API.

```{caption="{#code:drf:standard}DRF: Použití modulu drf-hal-json pro HAL" .python}
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS':
        'drf_hal_json.pagination.HalPageNumberPagination',
    'DEFAULT_PARSER_CLASSES': ('drf_hal_json.parsers.JsonHalParser',),
    'DEFAULT_RENDERER_CLASSES': (
        'drf_hal_json.renderers.JsonHalRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ),
    'URL_FIELD_NAME': 'self',
    # ...
}


# serializers.py
def serializer(model_):
    '''Get a default Serializer class for a model'''
    class _Serializer(HalModelSerializer):
        # ...
    return _Serializer


# views.py
class EnrollmentViewSet(HalCreateModelMixin, ReadOnlyModelViewSet):
    # ...
```

```{caption="{#code:drf:hal}DRF: Příklad výstupu pro HAL" .python}
{
    "_links": {
        "self": "http://127.0.0.1:8000/courses/1/",
        "sport": "http://127.0.0.1:8000/sports/3/",
        "hall": "http://127.0.0.1:8000/halls/1/",
        "teacher": "http://127.0.0.1:8000/teachers/6/"
    },
    "id": 1,
    "shortcut": "BAS01",
    "day": 1,
    "starts_at": "13:30",
    "ends_at": "15:00",
    "notice": null,
    "semester": 1
}
```

Zobrazení dat ve standardizované podobě v Django REST frameworku je
možné,
systematické,
ale pracné, naštěstí existují knihovny, které lze rovnou použít.

Použití přirozených identifikátorů
----------------------------------

Pro použití přirozených identifikátorů stačí v pohledu nastavit hodnotu proměnné `lookup_field`
a změnit odkazy vedoucí na daný zdroj, což můžete vidět [v ukázce](#code:drf:ids).
Změna odkazů vyžaduje poměrně mnoho argumentů, které považuji za zbytečné.

```{caption="{#code:drf:ids}DRF: Použití přirozených identifikátorů" .python}
# serializers.py:
class SportSerializer(HyperlinkedModelSerializer):
    self = HyperlinkedIdentityField(
        read_only=True,
        view_name='sport-detail',
        lookup_field='shortcut')
    # ...

class CourseSerializer(HyperlinkedModelSerializer):
    sport = HyperlinkedRelatedField(
        read_only=True,
        view_name='sport-detail',
        lookup_field='shortcut')
    # ...

# views.py:
class SportViewSet(ReadOnlyModelViewSet):
    # ...
    lookup_field = 'shortcut'
```

Bohužel knihovna `drf-hal-json` v kombinaci s přirozenými identifikátory vede k chybě,
kterou jsem autorům nahlásil. Pokud knihovna `drf-hal-json` není použita, přirozené identifikátory
fungují dle očekávání.

Použití přirozených identifikátorů v Django REST frameworku je
možné,
systematické,
ale zbytečně pracné.

Přístupová práva
----------------




```{caption="{#code:drf:auth}DRF: Autorizační třídy" .python}


```

Přístupová práva v Django REST frameworku jsou
možná,
systematická
a jednoduchá.

Generování dokumentace
----------------------




```{caption="{#code:drf:docs}DRF: Generování dokumentace" .python}

```


Generování dokumentace v Django REST frameworku je
možné a automatické,
systematické
a triviální.

Funkce služby
-------------


### Stránkování



### Filtrování



### Řazení



### Vyjednávání o obsahu



### Rozcestník

Django REST framework automaticky vytváří rozcestník.


Další poznámky
--------------




Kompletní implementace
----------------------

Kompletní implementaci REST API pro rozvrhová data ÚTVS ČVUT v Django REST frameworku
najdete na přiloženém médiu a na adrese
\url{https://github.com/hroncok/utvsapi-django}.
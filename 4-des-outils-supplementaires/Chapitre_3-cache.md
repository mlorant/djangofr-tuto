La mise en cache
================

Lorsqu'un site devient populaire, que son trafic augmente et que toutes les ressources du serveur sont utilisées, nous constatons généralement des ralentissements ou des plantages du site. Ces deux cas de figure sont généralement à éviter, et pour ce faire il faut procéder à des optimisations. Une optimisation courante est la mise en cache, que nous aborderons dans ce chapitre.

Cachez-vous !
-------------

Avant d'aborder l'aspect technique, il faut comprendre l'utilité de la mise en cache. Si vous présentez dans une vue des données issues de calculs très longs et complexes (par exemple, une dizaine de requêtes SQL), l'idée est d'effectuer les calculs une fois, de sauvegarder le résultat et de présenter le résultat sauvegardé pour les prochaines visites, plutôt que de recalculer la même donnée à chaque fois.

En effet, sortir une donnée du cache étant bien plus rapide et plus simple que de la recalculer, la durée d'exécution de la page est largement réduite, ce qui laisse donc davantage de ressources pour d'autres requêtes. Finalement, le site sera moins enclin à planter ou être ralenti.

Cela pose tout de même la question, très importante, de savoir quand est-ce qu'un cache est encore valide. Cette question restera ouverte car elle dépend de votre application ! Faut-il invalider un cache après un certain temps ? Une certaine action ? Nous allons montrer comment effacer des données du cache, à vous de choisir quand le faire.

Une question subsiste : où faut-il sauvegarder les données ? Django enregistre les données dans un système de cache, or il dispose de plusieurs systèmes de cache différents de base. Nous tâcherons de les introduire brièvement dans ce qui suit. 

Chacun de ces systèmes a ses avantages et désavantages, et tous fonctionnent un peu différemment. Il s'agit donc de trouver le système de cache le plus adapté à vos besoins.

La configuration du système de cache se fait grâce à la variable `CACHES` de votre `settings.py`.

### Dans des fichiers

Le système de cache le plus simple et connu est celui enregistrant les données dans des fichiers sur le disque dur du serveur. Pour chaque valeur enregistrée dans le cache, le système va créer un fichier et y enregistrer le contenu de la donnée sauvegardée. Voici comment le configurer :

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```

La clé `'LOCATION'` doit pointer vers un dossier, et non pas vers un fichier spécifique et **doit etre un chemin absolu**. Si vous êtes sous Windows, voici ce à quoi doit ressembler la valeur de `'LOCATION'` : `'c:/mon/dossier'` (ne pas oublier le `c:/` ou autre identifiant de votre disque dur). La clé `'BACKEND'` indique simplement le système de cache utilisé (et sera à chaque fois différente pour chaque système présenté par la suite).

Une fois ce système de cache configuré, Django va créer des fichiers dans le dossier concerné. Ces fichiers seront « sérialisés » en utilisant [le module `pickle`](https://docs.python.org/3/library/pickle.html) pour y encoder les données à sauvegarder. Vous devez également vous assurer que votre serveur web a bien accès en écriture et en lecture sur le dossier que vous avez indiqué.

### Dans la mémoire

Un autre système de cache simple est la mise en mémoire. Toutes vos données seront enregistrées dans la mémoire vive du serveur. Voici la configuration de ce système :

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'cache_crepes'
    }
}
```

Si vous utilisez cette technique, faites attention à la valeur de `'LOCATION'`. En effet, si plusieurs sites avec Django utilisant cette technique de cache tournent sur le même serveur, il est impératif que chacun d'entre eux dispose d'un nom de code différent pour `'LOCATION'`. Il s'agit en réalité juste d'un identifiant de l'instance du cache, qui va allouer une partie de la mémoire. Si plusieurs sites partagent le même identifiant, ils rentreront en conflit.

Ce cache n'est pas très performant au niveau de la gestion mémoire mais est pratique pour un environnement de développement. Il est par ailleurs le système utilisé par défaut si vous ne spécifiez pas de configuration.

### Dans la base de données

Pour utiliser la base de données comme système de cache, il faut avant tout créer une table dans celle-ci pour y accueillir les données. Cela se fait grâce à une commande spéciale de `manage.py` :

```console
python manage.py createcachetable [nom_table_cache]
```

où `[nom_table_cache]` est le nom de la table que vous allez créer (faites bien attention à utiliser un nom valide et pas déjà utilisé). Une fois cela fait, tout ce qu'il reste à faire est de l'indiquer dans le `settings.py` :

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'nom_table_cache',
    }
}
```

Ce système peut se révéler pratique et rapide si vous avez dédié tout un serveur physique à votre base de données ; néanmoins, il faut disposer de telles ressources pour arriver à quelque chose de convenable.

### En utilisant Memcached

[Memcached](http://memcached.org/) est un système de cache un peu à part. Celui-ci est en réalité indépendant de Django, et le framework ne s'en charge pas lui-même. Pour l'utiliser, il faut avant tout lancer un programme responsable lui-même du cache. Django ne fera qu'envoyer les données à mettre en cache et les récupérer par la suite, c'est au programme de sauvegarder et de gérer ces données.

Si cela peut sembler assez pénible à déployer, le système est en revanche très rapide et probablement le plus efficace de tous. Memcached va enregistrer les données dans la mémoire vive, comme le système vu précédemment qui utilisait la même technique, sauf qu'en comparaison de ce dernier Memcached est bien plus efficace et utilise moins de mémoire. Memcached est utile si vous comptez utiliser beaucoup de cache, sur un site à plutôt fort traffic. 

Memcached n'existe officiellement que sous Linux. Si vous êtes sous Debian ou dérivés, vous pouvez l'installer grâce à `apt-get install memcached`. Pour les autres distributions, référez-vous à la liste des paquets fournis par votre gestionnaire de paquets. Une fois que Memcached est installé, vous pouvez lancer le démon en utilisant la commande `memcached -d -m 512 -l 127.0.0.1 -p 11211`, où le paramètre `-d` permet le lancement du démon, `-m` indique la taille maximale de mémoire vive allouée au cache (en mégaoctets), et `-l` et `-p` donnent respectivement l'adresse IP et le port d'écoute du démon.

La configuration côté Django est encore une fois relativement simple :

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```

La clé `'LOCATION'` indique la combinaison adresse `IP:port` depuis laquelle Memcached est accessible. Nous avons adapté la valeur de la variable à la commande indiquée ci-dessus.

### Pour le développement

Pour terminer, il existe un dernier système de cache. Celui-ci ne fait rien (il n'enregistre aucune donnée et n'en renvoie aucune). Il permet juste d'activer le système de cache, ce qui peut se révéler pratique si vous utilisez le cache en production, mais que vous n'en avez pas besoin en développement. Voici sa configuration :


```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}
```

### Au final, quel système choisir ?

Cela dépend de votre site et de vos attentes. En développement, vous pouvez utiliser le cache de développement ou le cache mémoire (le simple, sans Memcached) si vous en avez besoin. En production, si vous avez peu de données à mettre en cache, la solution la plus simple est probablement le système utilisant les fichiers. 
En revanche, lorsque le cache devient fortement utilisé, Memcached est probablement la meilleure solution si vous pouvez l'installer. Sinon utilisez le système utilisant la base de données, même s'il n'est pas aussi efficace que Memcached, il devrait tout de même apaiser votre serveur.

Quand les données jouent à cache-cache
--------------------------------------

Maintenant que notre cache est configuré, il ne reste plus qu'à l'utiliser. Il existe différentes techniques de mise en cache que nous expliquerons dans ce sous-chapitre.

### Cache par vue

Une méthode de cache pratique est la mise en cache d'une vue. Avec cette technique, dès que le rendu d'une vue est calculé, il sera directement enregistré dans le cache. Tant que celui-ci sera dans le cache, la vue ne sera plus appelée et la page sera directement cherchée dans le cache.

Cette mise en cache se fait grâce à un décorateur : `django.views.decorators.cache.cache_page`. Voici son utilisation :


```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)
def lire_article(request, id):
    article = Article.objects.get(id=id)
    # ...
```

Le paramètre du décorateur correspond à la durée après laquelle le rendu dans le cache aura expiré. Cette durée est exprimée en secondes. Autrement dit, ici, après 15 fois 60 secondes — donc 15 minutes — la donnée sera supprimée du cache et Django devra régénérer la page, puis remettre la nouvelle version dans le cache. Grâce à cet argument, vous êtes assurés que le cache restera à jour automatiquement.  
Bien évidemment, chaque URL aura sa propre mise en cache. En effet, pour `lire_article`, il est normal que `/article/42` et `/article/1337` ne partagent pas le même résultat en cache (étant donné qu'ils n'affichent pas le même article). Il est également possible de spécifier une mise en cache directement depuis le `urls.py`. Ainsi, la mise en cache de vues génériques est également possible :

```python
from django.views.decorators.cache import cache_page
from blog.views import lire_article

urlpatterns = ('',
    (r'^article/(\d{1,4})/$', cache_page(60 * 15)(lire_article)),
)
```

Ici, le décorateur `@cache_page` est tout de même appliqué à la vue. Faites bien attention à inclure la vue sous forme de référence, et non pas sous forme de chaîne de caractères.

### Dans les templates

Il est également possible de mettre en cache certaines parties d'un template. Cela se fait grâce au tag `{% cache %}`. Ce tag doit au préalable être inclus grâce à la directive `{% load cache %}`. De plus, `cache` prend deux arguments au minimum : la durée d'expiration de la valeur (toujours en secondes), et le nom de cette valeur en cache (une clé que Django utilisera pour retrouver la bonne valeur dans le cache) :


```jinja
{% load cache %}
{% cache 500 carrousel %}
    /* mon carrousel */
{% endcache %}
```

Ici, nous enregistrons dans le cache notre carrousel. Celui-ci expirera dans 500 secondes et nous utilisons la clé `carrousel`.

Sachez que vous pouvez également enregistrer plusieurs copies en cache d'une même partie de template dépendant de plusieurs variables. Par exemple, si notre carrousel est différent pour chaque utilisateur, vous pouvez réutiliser une clé dynamique et différente pour chaque utilisateur. Ainsi, chaque utilisateur connecté aura dans le cache la copie du carrousel adaptée à son profil. Exemple :

```jinja
{% load cache %}
{% cache 500 carrousel user.username %}
    /* mon carrousel adapté à l'utilisateur actuel */
{% endcache %}
```

Cela peut s'avérer pratique pour garder un cache selon la langue de l'utilisateur ou son état (connecté ou pas).

### La mise en cache de bas niveau

Il arrive parfois qu'enregistrer toute une vue ou une partie de template soit une solution exagérée, et non adaptée. C'est là qu'interviennent plusieurs fonctions permettant de réaliser une mise en cache de variables bien précises. Presque tous les types de variables peuvent être mis en cache. Ces opérations sont réalisées grâce à plusieurs fonctions de l'objet `cache` du module `django.core.cache`. Cet objet `cache` se comporte un peu comme un dictionnaire. Nous pouvons lui assigner des valeurs à travers des clés :

```python
>>> from django.core.cache import cache
>>> cache.set('ma_cle', 'Coucou !', 30)
>>> cache.get('ma_cle')
'Coucou !'
```

Ici, la clé `'ma_cle'` contenant la chaîne de caractères `'Coucou !'` a été enregistrée pendant 30 secondes dans le cache (l'argument de la durée est optionnel ; s'il n'est pas spécifié, la valeur par défaut `TIMEOUT` de la configuration sera utilisée. Par défaut, elle vaut 300 secondes). Vous pouvez essayer, après ces 30 secondes, `get` renvoie `None` si la clé n'existe pas ou plus :

```python
>>> cache.set('ma_cle', 'Coucou !', 10)
>>> cache.get('ma_cle')
'Coucou !'
>>> sleep(10)
>>> cache.get('ma_cle')
>>>
```

Il est possible de spécifier une valeur par défaut si la clé n'existe pas ou plus :

```python
>>> cache.get('ma_cle', 'a expiré')
'a expiré'
```

Pour essayer d'ajouter une clé si elle n'est pas déjà présente, il faut utiliser la méthode `add`. Si cette clé est déjà présente, rien ne se passe :

```python
>>> cache.set('cle', 'Salut')
>>> cache.add('cle', 'Coucou')
>>> cache.get('cle')
'Salut'
```

Pour ajouter et obtenir plusieurs clés à la fois, il existe deux fonctions adaptées, `set_many` et `get_many` :

```python
>>> cache.set_many({'a': 1, 'b': 2, 'c': 3})
>>> cache.get_many(['a', 'b', 'c'])
{'a': 1, 'b': 2, 'c': 3}
```

Vous pouvez également supprimer une clé du cache, voire plusieurs en même temps :

```python
>>> cache.delete('a')
>>> cache.delete_many(['a', 'b', 'c'])
```

Pour vider tout le cache, voici la méthode `clear`. Toutes les clés et leurs valeurs seront supprimées :

```python
>>> cache.clear()
```

Pour terminer, il existe encore deux fonctions, `incr` et `decr`, qui permettent respectivement d'incrémenter et de décrémenter un nombre dans le cache, sans avoir à le récupérer pour le redéfinir :

```python
>>> cache.set('num', 1)
>>> cache.incr('num')
2
>>> cache.incr('num', 10)
12
>>> cache.decr('num')
11
>>> cache.decr('num', 5)
6
```

Le deuxième paramètre permet de spécifier le nombre d'incrémentations ou de décrémentations à effectuer.

Voilà ! Vous avez désormais découvert les bases du système de cache de Django. Cependant, nous n'avons vraiment que couvert les bases, il reste plein d'options à explorer, la possibilité d'implémenter son propre système de cache, etc. Si vous vous sentez limités sur ce sujet ou que vous avez d'éventuelles questions non reprises dans ce chapitre, [consultez la documentation](https://docs.djangoproject.com/en/stable/topics/cache/).


 En résumé
-----------

 - Le cache permet de sauvegarder le résultat de calculs ou traitements relativement longs, afin de présenter le résultat sauvegardé pour les prochaines visites, plutôt que de recalculer la même donnée à chaque fois. 
 - Il existe plusieurs systèmes de mise en cache : par fichier, en base de données, dans la mémoire RAM.
 - La mise en cache peut être définie au niveau de la vue, via `@cache_page`, dans le fichier `urls.py`, ou encore dans les templates avec le tag `{% cache %}`.
 - Django fournit également un ensemble de fonctions permettant d'appliquer une mise en cache à tout ce que nous souhaitons, et d'en gérer précisément l'expiration de la validité.
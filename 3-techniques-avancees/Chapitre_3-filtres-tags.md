Simplifions nos templates : filtres, tags et contextes
======================================================

Comme nous l'avons vu rapidement dans le premier chapitre sur les templates, Django offre une panoplie de filtres et de tags. Cependant, il se peut que vous ayez un jour un besoin particulier impossible à combler avec les filtres et tags de base. Heureusement, Django permet également de créer nos propres filtres et tags, et même de générer des variables par défaut lors de la construction d'un template (ce que nous appelons le contexte du template). Nous aborderons ces différentes possibilités dans ce chapitre.


Préparation du terrain : architecture des filtres et tags
---------------------------------------------------------

Pour construire nos propres filtres et tags, Django impose que ces derniers soient placés dans une application, tout comme les vues ou les modèles. À partir d'ici, nous retrouvons deux écoles dans la communauté de Django :

 - Soit votre fonctionnalité est propre à une application (par exemple un filtre utilisé uniquement lors de l'affichage d'articles sur votre blog), dans ce cas vous pouvez directement le(s) placer au sein de l'application concernée ; nous préférons cette méthode ;
 - Soit vous créez une application à part, qui regroupe tous vos filtres et tags personnalisés de votre projet.

Une fois ce choix fait, la procédure est identique : l'application choisie doit contenir un dossier nommé `templatetags` (attention au `s` final !), dans lequel il faut créer un fichier Python par groupe de filtres/tags (plus de détails sur l'organisation de ces fichiers viendront plus tard). Pour le moment, nous allons en créer un, appelé `blog_extras.py`.   
Ce dossier `templatetags` doit en réalité être un module Python classique afin que les fichiers qu'il contient puissent être importés. Il est donc impératif de créer un fichier `__init__.py` vide, sans quoi Django ne pourra rien faire.

La nouvelle structure de l'application « blog » est donc la suivante : 

```
blog/
   __init__.py
   models.py
   templatetags/
       __init__.py    # À ne pas oublier
      blog_extras.py
   views.py
```

Une fois les fichiers créés, il est nécessaire de spécifier une instance de classe qui nous permettra d'_enregistrer nos filtres et tags_, de la même manière que dans nos fichiers `admin.py` avec `admin.site.register()`. Pour ce faire, il faut déclarer les deux lignes suivantes au début du fichier `blog_extras.py` :

```python
from django import template

register = template.Library()
```

L'application incluant les `templatetags` doit être incluse dans le fameux `INSTALLED_APPS` de notre `settings.py`, si vous avez décidé d'ajouter vos tags et filtres personnalisés dans une application spécifique. Une fois les nouveaux tags et filtres codés, il sera possible de les intégrer dans n'importe quel template du projet via la ligne suivante :

```jinja
{% load blog_extras %}
```

Le nom `blog_extras` dans ce tag vient du nom de fichier que nous avons renseigné plus haut, à savoir `blog_extras.py`, sans l'extension `.py`.  
Tous les dossiers `templatetags` de toutes les applications partagent le même espace de noms. Si vous utilisez des filtres et tags de plusieurs applications, veillez à ce que leur noms de fichiers soient différents, afin qu'il n'y ait pas de conflit !

Nous pouvons désormais entrer dans le vif du sujet, à savoir la création de filtres et de tags !


Personnaliser l'affichage de données avec nos propres filtres
-------------------------------------------------------------

Commençons par les filtres. En soi, un filtre est une fonction classique qui prend 1 ou 2 arguments :

 - La variable à afficher, qui peut être n'importe quel objet en Python ;
 - Et de façon facultative, un paramètre.

Comme petit rappel au cas où vous auriez la mémoire courte, voici un exemple d'utilisation de deux filtres : l'un sans paramètre, le deuxième avec.

```jinja
{{ texte|upper }}            -> Filtre upper sur la variable "texte"
{{ texte|truncatewords:80 }} -> Filtre truncatewords, avec comme argument "80" sur la variable "texte"
```

Les fonctions Python associées à ces filtres ne sont appelées qu'au sein du template. Pour cette raison, il faut éviter de lancer des exceptions, et toujours renvoyer un résultat. En cas d'erreur, il est plus prudent de renvoyer l'entrée de départ ou une chaîne vide, afin d'éviter des effets de bord lors du « chaînage » de filtres par exemple.

### Un premier exemple de filtre sans argument

Attaquons la réalisation de notre premier filtre. Pour commencer, prenons comme exemple le modèle [« Citation » de Wikipédia](http://fr.wikipedia.org/wiki/Modèle:Citation) : nous allons encadrer la chaîne fournie par des guillemets français doubles.
Ainsi, si dans notre template nous avons `{{ "Bonjour le monde !"|citation }}`, le résultat dans notre page sera _« Bonjour le monde ! »_.

Pour ce faire, il faut ajouter une fonction nommée `citation` dans `blog_extras.py`. Cette fonction n'a pas d'argument particulier à Django et son écriture est assez intuitive :

```python
def citation(texte):   
    """
    Affiche le texte passé en paramètre, encadré de guillemets français
    doubles et d'espaces insécables
    """
    return "« %s »" % texte
```

Une fois la fonction écrite, il faut préciser au framework d'attacher cette méthode au filtre qui a pour nom `citation`. Encore une fois, il y a deux façons différentes de procéder :

 - Soit en ajoutant la ligne `@register.filter` comme décorateur de la fonction. L'argument `name` peut être indiqué pour choisir le nom du filtre ;
 - Soit en appelant la méthode `register.filter('citation', citation)`.
 
Notons qu'avec ces deux méthodes le nom du filtre n'est donc pas directement lié au nom de la fonction, et cette dernière aurait pu s'appeler `filtre_citation` ou autre, cela n'aurait posé aucun souci tant qu'elle est correctement renseignée par la suite.

Ainsi, ces trois fonctions sont équivalentes :


```python
#-*- coding:utf-8 -*-
from django import template

register = template.Library()

@register.filter
def citation(texte):   
    return "« %s »" % texte


@register.filter(name='mon_filtre_citation')
def citation2(texte):
    return "« %s »" % texte

def citation3(texte):
    return "« %s »" % texte

register.filter('un_autre_filtre_citation', citation3)
```

Par commodité, nous n'utiliserons plus que les première et deuxième méthodes dans ce cours. La dernière est pour autant tout à fait valide, libre à vous de l'utiliser si vous préférez celle-ci.

Nous pouvons maintenant essayer le nouveau filtre dans un template. Il faut tout d'abord charger les filtres dans notre template, via le tag `load`, introduit récemment, puis appeler notre filtre `citation` sur une chaîne de caractères quelconque :

```jinja
{% load blog_extras %}
Un jour, une certaine personne m'a dit : {{ "Bonjour le monde !"|citation }}
```

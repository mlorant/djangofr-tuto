Les messages
============

Il est souvent utile d'envoyer des notifications au visiteur, pour par exemple lui confirmer qu'une action s'est bien réalisée, ou au contraire lui indiquer une erreur. Django propose un système de notification permettant d'enregistrer une liste de messages à afficher à la prochaine page de l'utilisateur. Nous le présenterons dans ce chapitre.

Envoyons des messages à nos utilisateurs 
----------------------------------------

Avant tout, il faut s'assurer que l'application et ses dépendances sont bien installées. Elles le sont généralement par défaut, néanmoins il est toujours utile de vérifier. Autrement dit, dans votre `settings.py`, vous devez avoir :

 - Dans `MIDDLEWARE_CLASSES`, `'django.contrib.messages.middleware.MessageMiddleware'` ; 
 - Dans `INSTALLED_APPS`, `'django.contrib.messages'` ;
 - Dans `TEMPLATE_CONTEXT_PROCESSORS` (si cette variable n'est pas dans votre `settings.py`, inutile de la rajouter, elle contient déjà l'entrée par défaut), `'django.contrib.messages.context_processors.messages'`.

Cela étant fait, nous pouvons entrer dans le vif du sujet.

Django peut envoyer des notifications (aussi appelées messages) à tous les visiteurs, qu'ils soient connectés ou non. Il existe plusieurs niveaux de messages par défaut (nous verrons comment en ajouter par la suite) :

 - `DEBUG` : message destiné pour la phase de développement uniquement. Ces messages ne seront affichés que si `DEBUG=True` dans votre `settings.py`.
 - `INFO` : message d'information pour l'utilisateur.
 - `SUCCESS` : confirmation qu'une action s'est bien déroulée.
 - `WARNING` : une erreur n'a pas été rencontrée, mais pourrait être imminente.
 - `ERROR` : une action ne s'est pas déroulée correctement ou une erreur quelconque est apparue.

Chaque niveau a son « tags » associé, qui est le niveau en minuscules (debug, info, ...). Ces tags sont notamment utilisés pour pouvoir définir un style CSS précis à chaque niveau, afin de pouvoir les différencier.

Il existe une fonction standard pour envoyer un message depuis une vue à l'utilisateur courant :

```python
from django.contrib import messages
messages.add_message(request, messages.INFO, 'Bonjour visiteur !')
```

Ici, nous avons envoyé un message, avec le niveau `INFO`, au visiteur, contenant « *Bonjour visiteur !* ». Il est important de ne pas oublier le premier argument : `request`, l'objet `HttpRequest` donné à la vue. Sans cela, Django ne saura pas où stocker le message.

Il existe également quelques raccourcis pour les niveaux par défaut, plus rapide à écrire :

```python
messages.debug(request, u'%s requêtes SQL ont été exécutées.' % compteur)
messages.info(request, u'Rebonjour !')
messages.success(request, u'Votre article a bien été mis à jour.')
messages.warning(request, u'Votre compte expire dans 3 jours.')
messages.error(request, u'Cette image n\'existe plus.')
```

Maintenant que vous savez envoyer des messages, il ne reste plus qu'à les afficher. Pour ce faire, Django se charge de la majeure partie du travail : une variable `messages` est inclus dans le contexte grâce *context_processors* installé. Tout ce qu'il reste à faire, c'est de choisir où afficher les notifications dans le template. Voici une ébauche de code de template pour afficher les messages au visiteur :

```jinja
{% if messages %}
<ul class="messages">
    {% for message in messages %}
    <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
    {% endfor %}
</ul>
{% endif %}
```

La variable `messages` étant ajoutée automatiquement par le framework, il suffit de l'itérer, étant donné qu'elle peut contenir plusieurs notifications. À l'itération de chaque message, ce dernier sera supprimé tout seul, ce qui assure que l'utilisateur ne verra pas deux fois la même notification. Finalement, un message dispose de l'attribut `tags` pour d'éventuels styles CSS (il peut y avoir plusieurs tags, ils seront séparés par une espace), et il suffit ensuite d'afficher le contenu de la variable.

Ce code est de préférence à intégrer dans une base commune (appelée depuis `{% extends %}`) pour éviter de devoir le réécrire dans tous vos templates.

Niveau de messages et filtrage
------------------------------

Nous avons dit qu'il était possible de créer des niveaux de messages personnalisés. Il faut savoir qu'en réalité les niveaux de messages ne sont en fait que des entiers constants :

```python
>>> from django.contrib import messages
>>> messages.INFO
20
```

Voici la relation entre niveau et entier par défaut :

- `DEBUG` : 10
- `INFO` : 20
- `SUCCESS` : 25
- `WARNING` : 30
- `ERROR` : 40

Au final, il suffit de créer une constante pour ajouter un nouveau niveau. Par exemple :

```python
CRITICAL = 50

messages.add_message(request, CRITICAL, 'Une erreur critique est survenue.')
```

Il est également possible d'ajouter des tags à un message, pour ajouter des classes CSS notamment. Ici, le tag « fail » sera ajouté :

```python
messages.add_message(request, CRITICAL, 'Une erreur critique est survenue.', extra_tags="fail")
```

Sachez que vous pouvez également limiter l'affichage des messages à un certain niveau (égal ou supérieur). Cela peut se faire de deux manières différentes. Soit depuis le `settings.py` en mettant `MESSAGE_LEVEL` au niveau minimum des messages à afficher (par exemple 25 pour ne pas montrer les messages `DEBUG` et `INFO`), soit en faisant cette requête dans la vue :

```python
messages.set_level(request, messages.DEBUG)
```

En utilisant cela et pour cette vue uniquement, tous les messages dont le niveau est égal ou supérieur à 10 (la valeur de `messages.DEBUG`) seront affichés. On préfèrera la méthode globale avec le `settings.py`, permettant d'afficher des informations supplémentaires lors du développement, sans avoir à changer le code. En effet, il suffit de mettre le minimum à `SUCCESS` sur votre environnement de production pour cacher les messages de debug et d'info.

Enfin, si vous souhaitez faire une application réutilisable, il se peut que les gens qui souhaitent l'intégrer n'utilisent pas le système de messages sur leur projet. Pour éviter de provoquer des erreurs, si vous pensez que le message est facultatif, vous pouvez spécifier à Django d'échoue silenciement : 

```python
messages.info(request, 'Message à but informatif.', fail_silently=True)
```  

Dans ce cas, le message sera bien envoyé si les messages sont activés pour le projet en cours. Sinon la ligne sera ignorée, tout simplement.

En résumé
---------

 - Les messages permettent d'afficher des notifications à l'utilisateur, en cas de succès ou d'échec lors d'une action, ou si un événement particulier se produit. 
 - Il existe différents types de messages : ceux d'information, de succès, d'erreur, d'avertissement, et enfin de débogage.
 - Il est possible de créer son propre type de message et de configurer son comportement.
 - L'affichage de messages peut être limité : chaque type de message est caractérisé par une constante entière, et nous pouvons afficher les messages ayant un niveau supérieur ou égal à un certain seuil, via `messages.set_level`.
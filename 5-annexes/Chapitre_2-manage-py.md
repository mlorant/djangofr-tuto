L'utilitaire manage.py
======================

Tout au long de ce cours, vous avez utilisé la commande `manage.py` fournie par Django pour différents besoins : créer un projet, créer une application, mettre à jour la structure de la base de données, ajouter un super-utilisateur, enregistrer et compiler des traductions, etc. Pour chaque tâche différente, `manage.py` possède une commande adaptée. Bien évidemment, nous n'avons vu qu'une fraction des possibilités offertes par cet outil. Dans ce chapitre, nous aborderons toutes les commandes une par une, avec leurs options, qu'elles aient déjà été introduites auparavant ou non.

N'hésitez pas à survoler cette liste, vous découvrirez peut-être une commande qui pourrait vous être utile à un moment donné, et vous n'aurez qu'à revenir dans ce chapitre pour découvrir comment elle marche exactement.

Les commandes de base
---------------------

### Prérequis

La plupart des commandes acceptent des arguments. Leur utilisation est propre à chaque commande et est indiquée dans le titre de celle-ci. Un argument entre chevrons `< … >` indique qu'il est obligatiore, tandis qu'un argument entre crochets `[ … ]` indique qu'il est optionnel. Référez-vous ensuite à l'explication de la commande pour déterminer comment ces arguments sont utilisés. 

Certaines commandes possèdent également des options pour modifier leur fonctionnement. Celles-ci commencent généralement toutes par deux tirets « `--` » et s'ajoutent avant ou après les arguments de la commande, s'il y en a. Les options de chaque commande seront présentées après l'explication du fonctionnement global de la commande.

Les commandes seront introduites par thème : nous commencerons par les outils de base dont la plupart ont déjà été expliqués, puis nous expliquerons toutes les commandes liées à la gestion de la base de données, et nous finirons par les commandes spécifiques à certains modules introduits auparavant, comme le système utilisateurs.

### Liste des commandes


#### runserver [port ou adresse:port]

Lance un serveur de développement local pour le projet en cours. Par défaut, ce serveur est accessible depuis le port 8000 et l'adresse IP 127.0.0.1, qui est l'adresse locale de votre machine. Le serveur redémarre à chaque modification du code dans le projet. Bien évidemment, ce serveur n'est pas destiné à la production. Le framework ne garantit ni les performances, ni la sécurité du serveur de développement pour une utilisation en production.

À chaque démarrage du serveur et modification du code, la commande `check` (expliquée par la suite) est lancée afin de vérifier si les modèles sont corrects ou si aucun soucis est présent dans votre code après une mise à jour de Django.

Il est possible de spécifier un port d'écoute différent de celui par défaut en spécifiant le numéro de port. Ici, le serveur écoutera sur le port 9000 :

```console
python manage.py runserver 9000
```

De même, il est possible de spécifier une adresse IP différente de 127.0.0.1 (pour l'accessibilité depuis le réseau local par exemple). Il n'est pas possible de spécifier une adresse IP sans spécifier le port :

```console
python manage.py runserver 192.168.1.6:7000
```

Le serveur de développement prend également en charge l'IPv6. Nous pouvons dès lors spécifier une adresse IPv6, tout comme nous spécifierions une adresse IPv4, à condition de la mettre entre crochets :

```console
python manage.py runserver [2001:0db8:1234:5678::9]:7000
```

**`--ipv6, -6`**
Il est également possible de remplacer l'adresse locale IPv4 127.0.0.1 par l'adresse locale IPv6::1 en spécifiant l'option `--ipv6 ou -6` :

```console
python manage.py runserver -6
```

**`--noreload`**
Empêche le serveur de développement de redémarrer à chaque modification du code. Il faudra procéder à un redémarrage manuel pour que les changements soient pris en compte.

**`--nothreading`**
Le serveur utilise par défaut des threads. En utilisant cette option, ceux-ci ne seront pas utilisés.


#### shell

Lance un interpréteur interactif Python. L'interpréteur sera configuré pour le projet et des modules de ce dernier pourront être importés directement.

Si vous avez [iPython](http://ipython.scipy.org/) ou [bpython](http://bpython-interpreter.org/) d'installé, Django utilisera votre interpréteur favori. Vous pouvez toutefois forcer l'interpréteur classique avec l'option `--plain`.

#### version

Indique la version de Django installée :

```console
python manage.py version
1.7
```

#### help <commande>

Affiche de l'aide pour l'utilisation de `manage.py`. Utilisez `manage.py help <commande>` pour accéder à la description d'une commande avec ses options.

#### startproject <nom> [destination]

[[attention]]
| Cette commande s'utilise obligatoirement avec `django-admin.py` : vous ne disposez pas encore de `manage.py` vu que le projet n'existe pas.

Crée un nouveau projet utilisant le nom donné en paramètre. Un nouveau dossier dans le répertoire actuel utilisant le nom du projet sera créé et les fichiers de base y seront insérés (`manage.py`, le sous-dossier contenant le `settings.py` notamment, etc.).

Il est possible d'indiquer un répertoire spécifique pour accueillir le nouveau projet :

```console
django-admin.py startproject crepes_bretonnes /home/crepes/projets/crepes
```

Ici, tous les fichiers seront directement insérés dans le dossier `/home/crepes/projets/crepes`.

**`--template`**
Cette option permet d'indiquer un modèle de projet à copier, plutôt que d'utiliser celui par défaut. Il est possible de spécifier un chemin vers le dossier contenant les fichiers ou une archive (.tar.gz, .tar.bz2, .tgz, .tbz, .zip) contenant également le modèle. Exemple :

```console
django-admin.py startproject --template=./projets/modele_projet crepes_bretonnes
```

Indiquer une URL (http, https ou ftp) vers une archive est également possible :

```console
django-admin.py startproject --template=http://monsite.com/modele_projet.zip crepes_bretonnes
```

Django se chargera de télécharger l'archive, de l'extraire, et de la copier dans le nouveau projet.

#### startapp <nom> [destination]

Crée une nouvelle application dans un projet. L'application sera nommée selon le nom passé en paramètre. Un nouveau dossier sera créé dans le projet avec les fichiers de base (`models.py`, `views.py`, …).

Si vous le souhaitez, vous pouvez indiquer un répertoire spécifique pour accueillir l'application, sinon l'application sera ajoutée dans le répertoire actuel :

```console
python manage.py startapp blog /home/crepes/projets/crepes/global/blog
```

**`--template`**
Tout comme `startproject`, cette option permet d'indiquer un modèle d'application à copier, plutôt que d'utiliser celui par défaut. Il est possible de spécifier un chemin vers le dossier contenant les fichiers ou une archive (.tar.gz, .tar.bz2, .tgz, .tbz, .zip) contenant le modèle. Exemple :

```console
python manage.py startapp --template=./projets/modele_app crepes_bretonnes/blog
```

Indiquer une URL (http, https ou ftp) vers une archive est également possible :

```console
python manage.py startapp --template=http://monsite.com/modele_app.zip crepes_bretonnes
```

Django se chargera de télécharger l'archive, de l'extraire, et de la copier dans la nouvelle application.


#### diffsettings

Indique les variables de votre `settings.py` qui ne correspondent pas à la configuration par défaut d'un projet neuf. Les variables se terminant par `###` sont des variables qui n'apparaissent pas dans la configuration par défaut, comme `SITE_ID` ou `ROOT_URLCONF`.
L'option `--all` permet de montrer toutes les variables même celle qui ont la valeur par défaut. Dans ce cas, elles sont *préfixées* par `###`

#### check

Vérifie votre projet Django pour détecter les problèmes courants :
 
 - dans les modèles ;
 - vos classes dans `admin.py` ;
 - des problèmes de compatibilité avec la version de Django utilisée (méthodes dépréciées par exemple)


#### test <application ou identifiant de test>

Lance les tests unitaires d'un projet, d'une application, d'un test ou d'une méthode en particulier. Pour lancer tous les tests d'un projet, il ne faut spécifier aucun argument. Pour une application, il suffit d'indiquer son nom (sans inclure le nom du projet) :

```console
python manage.py test blog
```

Pour ne lancer qu'une seule suite de tests unitaire, il suffit d'ajouter le chemin Python du test après le nom de l'application :

```console
python manage.py test blog.tests.BlogUnitTest
```


Finalement, pour ne lancer qu'une seule méthode d'un test unitaire, il faut également la spécifier après l'identifiant du test :


```console
python manage.py test blog.tests.BlogUnitTest.test_lecture_article
```

**`--failfast`**
Arrête le processus de vérification de tous les tests dès qu'un seul test a échoué et rapporte l'échec en question.

#### testserver <fixture fixture …>

Lance un serveur de développement avec les données des fixtures indiquées. Les fixtures sont des fichiers contenant des données pour remplir une base de données. Pour mieux comprendre le fonctionnement des fixtures, référez-vous à la commande `loaddata`.

Cette commande effectue trois actions :

 1. Elle crée une base de données vide. 
 2. Une fois la base de données créée, elle la remplit avec les données des fixtures passées en paramètre.
 3. Un serveur de développement est lancé avec la base de données venant d'être remplie.

Cette commande peut se révéler particulièrement utile lorsque vous devez régulièrement changer de données pour tester la vue que vous venez d'écrire. Au lieu de devoir à chaque fois créer et supprimer des données manuellement dans votre base pour vérifier chaque situation possible, il vous suffira de lancer le serveur de développement avec des fixtures adaptées à chaque situation.

**`--addrport [port ou adresse:port]`**
Tout comme pour `runserver`, le serveur de développement sera accessible par défaut depuis 127.0.0.1:8000. Il est également possible de spécifier un port d'écoute ou une adresse spécifique :

```console
python manage.py testserver --addrport 7000 fixture.json
```

```console
python manage.py testserver --addrport 192.168.1.7:9000 fixture.json
```
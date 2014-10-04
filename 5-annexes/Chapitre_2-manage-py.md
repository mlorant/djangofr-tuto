L'utilitaire manage.py
======================

Tout au long de ce cours, vous avez utilisé la commande `manage.py` fournie par Django pour différents besoins : créer un projet, créer une application, mettre à jour la structure de la base de données, ajouter un super-utilisateur, enregistrer et compiler des traductions, etc. Pour chaque tâche différente, `manage.py` possède une commande adaptée. Bien évidemment, nous n'avons vu qu'une fraction des possibilités offertes par cet outil. Dans ce chapitre, nous aborderons toutes les commandes une par une, avec leurs options, qu'elles aient déjà été introduites auparavant ou non.

N'hésitez pas à survoler cette liste, vous découvrirez peut-être une commande qui pourrait vous être utile à un moment donné, et vous n'aurez qu'à revenir dans ce chapitre pour découvrir comment elle marche exactement.

Les commandes de base
---------------------

### Prérequis

La plupart des commandes acceptent des arguments. Leur utilisation est propre à chaque commande et est indiquée dans le titre de celle-ci. Un argument entre crochets `[…]` indique qu'il est optionnel. Référez-vous ensuite à l'explication de la commande pour déterminer comment ces arguments sont utilisés. 

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
L'option `--all` permet de montrer toutes les variables même celle qui ont la valeur par défaut. Dans ce cas, elles sont *préfixées* par `###`.

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

Pour ne lancer qu'une seule suite de tests unitaire, il suffit d'utiliser le chemin Python du test :

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


La gestion de la base de données
--------------------------------

Toutes les commandes de cette section ont une option commune : `--database` qui permet de spécifier l'alias (indiqué dans votre `settings.py`) de la base de données sur laquelle la commande doit travailler si vous disposez de plusieurs bases de données. Exemple :

```console
python manage.py migrate --database=master
```

#### makemigrations [app\_name app\_name...]

Crée un ensemble de fichiers Python qui décrivent les changements de la base de données en fonction des précédentes migrations et de l'état actuel de vos `models.py`. Les fichiers créés sont versionnés, afin de pouvoir retourner en avant ou en arrière dans les modifications de votre modèle.

Il est possible de spécifier qu'une ou plusieurs applications à analyser.

**`--empty`**  
Cette option permet de créer une migration "vide", afin d'écrire soi même le contenu. Cette option est utile pour les utilisateurs avancées, familiers avec le système de migrations de Django.

**`--dry-run`**  
Permet de visualiser les changements dans vos modèles, sans enregistrer le fichier de migration correspondant. En combinant cette option avec `--verbosity 3`, vous pouvez voir le fichier qui serait créé.

**`--merge`**  
Permet de résoudre les conflits de mises à jour.

#### migrate [app\_name [migration\_name]]

Synchronise votre base de données avec l'état actuel de vos modèles (ajout de champs, suppression de tables SQL, etc.) Il y a trois cas d'exécution :

 - `migrate` : Applique toutes les modifications de toutes les applications installées ;
 - `migrate app_name` : Applique toutes les modifications pour l'application concerné et éventuellement ses dépendances en cas de clés étrangères ;
 - `migrate app_name migration_name` : Permet de retourner à l'état de la migration donné en paramètre. La valeur `zero` pour ce paramètre permet de défaire toutes les migrations déjà faites. 

**`--fake`**  
Marque la migration comme appliquée (ou "non appliquée", si vous retournez en arrière), sans pour autant appliquer les changements SQL. 

Ceci est réservé aux utilisateurs avancés qui veulent appliquer manuellement les changements. Sachez ce que vous faites : la base peut devenir instable si vous utilisez cette option précaution.

**`--list, -l`**  
Liste les applications installées et marque les migrations appliquées ou non, avec une croix. Les applications sans modèles seront également affichées mais avec la ligne `(no migrations)`.

#### dbshell

Lance le client de gestion de votre base de données en ligne de commande, selon les paramètres spécifiés dans votre `settings.py`. Cette commande présume que vous avez installé le client adapté à votre base de données et que celui-ci est accessible depuis votre console.

 - Pour PostgreSQL, l'utilitaire `psql` sera lancé ; 
 - Pour MySQL, l'utilitaire `mysql` sera lancé ;
 - Pour SQLite, l'utilitaire `sqlite3` sera lancé.

#### dumpdata [application application.modele …]

Affiche toutes les données d'applications ou de modèles spécifiques contenues dans votre base de données dans un format texte sérialisé. Par défaut, le format utilisé sera le JSON. Ces données sont des fixtures qui pourront être utilisées par la suite dans des commandes comme `loaddata` ou `testserver`. Exemple :

```console
python manage.py dumpdata blog.Article
[{"pk": 1, "model": "blog.article", "fields": {"date": "2012-07-11T15:51:08.607Z", "titre": "Les crêpes c'est trop bon", "categorie": 1,
 "auteur": "Maxime", "contenu": "Vous saviez que les crêpes bretonnes c'est trop bon ? La pêche c'est nul."}},
 {"pk": 2, "model": "blog.article", "fields": {"date": "2012-07-11T16:25:53.262Z", "titre": "Un super titre d'article !",
 "categorie": 1, "auteur": "Mathieu", "contenu": "Un super contenu ! (ou pas)"}}]
```

Il est possible de spécifier plusieurs applications et modèles directement :

```console
python manage.py dumpdata blog.Article blog.Categorie auth
```

Ici, les modèles `Article` et `Categorie` de l'application `blog` et tous les modèles de l'application `django.contrib.auth` seront sélectionnés et leurs données affichées. Si vous ne spécifiez aucune application ou modèle, tous les modèles du projet seront repris.

**`--all`**  
Force l'affichage de tous les modèles, à utiliser si certains modèles sont filtrés.

**`--format <fmt>`**  
Change le format utilisé. Utilisez `--format xml` pour utiliser le format XML par exemple.

**`--indent <nombre d'espace>`**  
Par défaut, comme vu plus tôt, toutes les données sont affichées sur une seule et même ligne. Cela ne facilite pas la lecture pour un humain, vous pouvez donc utiliser cette option pour indenter le rendu avec le nombre d'espaces spécifié pour chaque indentation.

**`--exclude`**  
Empêche certains modèles ou applications d'être affichés. Par exemple, si vous souhaitez afficher toutes les données de l'application `blog`, sauf celles du modèle `blog.Article`, vous pouvez procéder ainsi :

```console
python manage.py dumpdata blog --exclude blog.Article
```

**`--natural-foreign`**  
Utilise une représentation différente pour les relations `ForeignKey` et `ManyToMany` pour d'éventuelles situations à problèmes. Si vous affichez `contrib.auth.Permission` ou utilisez des `ContentType` dans vos modèles, vous devriez utiliser cette option.
   
**`--natural-primary`**  
Ne précisera pas les clés primaires (l'ID auto-incrémenté) puisqu'il pourra être déterminé à la désérialisation. 

**`--pks`**  
Permet de définir la liste des clés primaires à afficher, séparées par des virgules. Cette option n'est disponible uniquement pour l'affichage de modèles précis :

```console
python manage.py dumpdata blog.Article --pks=1 --indent 2
[
{
  "fields": {
    "date": "2014-07-03T18:53:49.743Z",
    "titre": "Exemple d'article",
    "categorie": 1,
    "auteur": "Maxime L.",
    "contenu": "Exemple de contenu"
  },
  "model": "blog.article",
  "pk": 1
}
]```

#### loaddata [fixture…]

Enregistre dans la base de données les fixtures passées en argument. Les fixtures sont des fichiers contenant des données de votre base de données. Ces données sont enregistrées dans un format texte spécifique, généralement en JSON ou en XML. Django peut dès lors directement lire ces fichiers et ajouter leur contenu à la base de données. Vous pouvez créer des fixtures à partir de la commande `dumpdata`.

La commande `loaddata` ira chercher les fixtures dans trois endroits différents :

 - Dans un dossier `fixtures` dans chaque application ;
 - Dans un dossier indiqué par la variable `FIXTURE_DIRS` dans votre `settings.py` ;
 - À partir du chemin vers le fichier donné en argument, absolu ou relatif.


Vous pouvez omettre d'indiquer l'extension de la fixture :

```console
manage.py loaddata ma_fixture
```

Dans ce cas, Django ira chercher dans tous les endroits susmentionnés et prendra toutes les fixtures ayant une terminaison correspondant à un format de fixtures (.json pour le JSON ou .xml pour le XML par exemple). Dès lors, si vous avez un fichier `ma_fixture.json` dans un dossier `fixtures` d'une application, celui-ci sera sélectionné.

Bien entendu, vous pouvez également spécifier un fichier avec l'extension :

```console
manage.py loaddata ma_fixture.xml
```

Dans ce cas, le fichier devra obligatoirement s'appeler `ma_fixture.xml` pour être sélectionné.

Django peut également gérer des fixtures compressées. Si vous indiquez `ma_fixture.json` comme fixture à utiliser, Django cherchera `ma_fixture.json`, `ma_fixture.json.zip`, `ma_fixture.json.gz` ou `ma_fixture.json.bz2`. S'il tombe sur une fixture compressée, il la décompressera, puis lancera le processus de copie.

**`--app`**  
Vous pouvez limiter la recherche de fixtures à une application précise via `--app`. 

#### inspectdb

Inspecte la base de données spécifiée dans votre `settings.py` et crée à partir de sa structure un `models.py`. Pour chaque table dans la base de données, un modèle correspondant sera créé. Cette commande construit donc des modèles à partir de tables, il s'agit de l'opération inverse des commandes `makemigrations` et `migrate`.

Notons qu'il ne s'agit ici que d'un raccourci. Les modèles créés automatiquement doivent être relus et vérifiés manuellement. Certains champs nommés avec des mots-clés de Python (comme `class` ou `while` par exemple) peuvent avoir été renommés pour éviter d'éventuelles collisions. Ces collisions sont toutefois bien gérées par Django, via l'attribut `db_column` qui fera le lien entre le nouveau nom et celui de votre base.   
Il se peut également que Django n'ait pas réussi à identifier le type d'un champ et le remplace par un `TextField`. Il convient également d'être particulièrement attentif aux relations `ForeignKey`, `ManyToManyField`, etc.

Voici un extrait d'un `inspectdb`, reprenant le modèle `Article` de notre application `blog` créée dans le cours :

```python
class BlogArticle(models.Model):
    id = models.IntegerField(primary_key=True)   # AutoField?
    titre = models.CharField(max_length=100)
    auteur = models.CharField(max_length=42)
    contenu = models.TextField(blank=True)
    date = models.DateTimeField()
    categorie = models.ForeignKey('BlogCategorie')
    
    class Meta:
        managed = False
        db_table = 'blog_article'
```

Par défaut, les modèles générés sont marqués avec `managed` à `False`. Cela signifie qu'ils seront ignorés par les commandes `makemigrations` et `migrate`. Vous pouvez enlever cette ligne si vous souhaitez appliquer des modifications à la base après modification de vos modèles.

#### flush

Réinitialise la base de données et insère les fixtures `initial_data`. Vous pouvez utiliser `--no-initial-data` pour avoir des tables totalement vides.


#### sql* application [application …]

De nombreuses commandes permettent d'afficher les requêtes SQL correspondant à des opérations précises : 

- `sql` : requêtes de création de tables ;
- `sqlall` : requêtes de créations de tables et d'ajout des données initiales ;
- `sqlclear` : requêtes de suppression des tables ;
- `sqldropindexes` : requêtes de suppression des index SQL ;
- `sqlflush` : ensemble des requêtes exécutées par la commande `flush` ;
- `sqlindexes` : requêtes de création des index SQL ;
- `sqlmigrate` : requêtes d'une migration précise. En plus du nom de l'application, il faut fournir un nom de migration en paramètre ;
- `sqlsequencereset` : requête pour réinitialiser les index de séquences de certains SGBD. (les séquences permettent de déterminer l'index numérique à assigner à la prochaine entrée créée) ;

Seules les commandes `sqlmigrate` et `sqlflush` marchent avec les tables crées via les migrations Django.
 
#### sqlcustom application [application …]

Affiche des requêtes SQL contenues dans des fichiers. Django affiche les requêtes contenues dans les fichiers `<application>/sql/<modele>.sql` où `<application>` est le nom de l'application donné en paramètre et `<modele>` un modèle quelconque de l'application. Si nous avons l'application `blog` incluant le modèle `Article`, la commande `manage.py sqlcustom blog` affichera les requêtes du fichier `blog/sql/article.sql` s'il existe, et y ajoutera toutes les autres requêtes des autres modèles de l'application.

À chaque fois que vous réinitialisez votre base de données par exemple, cette commande vous permet d'exécuter facilement des requêtes SQL spécifiques en les joignant à la commande `dbshell` via des pipes sous Linux ou Mac OS X :

```console
python manage.py sqlcustom blog | python manage.py dbshell
```

Les commandes d'applications
----------------------------

#### clearsessions

Supprime les sessions expirées de `django.contrib.sessions`. Cette commande peut être utilisée dans une tâche cron pour nettoyer régulièrement cette table.


#### changepassword [pseudo]

Permet de changer le mot de passe d'un utilisateur en spécifiant son pseudo :

```console
python manage.py changepassword Mathieu
```

Si aucun nom d'utilisateur n'est spécifié, Django prendra le nom d'utilisateur de la session console actuelle. Cette commande n'est disponible que si le système d'utilisateurs (`django.contrib.auth`) est installé.

#### createsuperuser

Permet de créer un super-utilisateur (un utilisateur avec tous les pouvoirs). Cette commande demandera le pseudo, l'adresse e-mail — si ceux-ci n'ont pas été spécifiés à partir des options — et le mot de passe. 

**`--username` et `--email`**  
Permettent de spécifier directement le nom et l'adresse e-mail de l'utilisateur.

Si ces deux options sont indiquées, vous devrez spécifier le mot de passe manuellement par la suite afin que l'utilisateur puisse se connecter.


#### makemessages

Parcourt tous les fichiers de l'arborescence à partir du dossier actuel pour déterminer les chaînes de caractères à traduire et crée ou met à jour les fichiers de traduction. Référez-vous au chapitre sur l'internationalisation pour plus d'informations.

**`--all, -a`**  
Met à jour les chaînes à traduire pour tous les langages.

**`--extensions`**  
Indique de ne sélectionner que les fichiers qui ont une extension spécifique :

```console
python manage.py makemessages --extension=html,txt
```

… ne prendra que les fichiers HTML et TXT.

**`--locale`**, **`-l`**  
Permet de ne mettre à jour qu'une ou plusieurs langues :

```console
python manage.py makemessages --locale fr_FR
python manage.py makemessages -l fr_FR -l da-dk
```

**`--symlinks`**  
Autorise Django à suivre les liens symboliques en explorant les fichiers.

**`--ignore, -i`**  
Permet d'ignorer certains fichiers :

```console
python manage.py makemessages --ignore=blog/* --ignore=*.html
```

Tous les fichiers HTML et du dossier `blog` seront ignorés.

**`--no-wrap`**  
Empêche Django de répartir les longues chaînes de caractères en plusieurs lignes dans les fichiers de traduction.

**`--no-location`**  
Empêche Django d'indiquer la source de la chaîne de caractères dans les fichiers de traduction (nom du fichier et ligne dans celui-ci).


#### compilemessages

Compile les fichiers de traduction `.po` vers des fichiers `.mo` afin que gettext puisse les utiliser. Référez-vous au chapitre sur l'internationalisation pour plus d'informations.

**`--locale`**  
Permet de ne compiler qu'une seule langue :

```console
python manage.py compilemessages --locale fr_FR
```

#### createcachetable table\_name

Crée une table de cache pour le système de cache. Référez-vous au chapitre sur le système de cache pour comprendre son fonctionnement. La table sera nommée selon le nom donné en paramètre.

Il existe d'autres commandes d'applications que nous avons déjà abordés en détails et qui ne nécéssite pas d'ajout comme `collectstatic`.   
Il y a également des commandes pour des outils que nous n'avons pas vu (gis, les sitemaps, ...), donc il ne nous sert à rien de les voir ici.
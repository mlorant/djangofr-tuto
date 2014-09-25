L'internationalisation
======================

De nos jours, la plupart des sites web proposent plusieurs langues à leurs utilisateurs, et ciblent même la langue par défaut en fonction du visiteur. Ce concept apporte son lot de problèmes lors de la réalisation d'un site : que faut-il traduire ? Quelle méthode faut-il utiliser pour traduire facilement l'application, sans dupliquer le code ?

Nous allons voir dans ce chapitre comment traduire notre site en plusieurs langues, de façon optimale sans dupliquer nos templates et vues, via des méthodes fournies dans Django et l'outil **gettext** permettant de créer des fichiers de langue.

Sachez que par convention en informatique le mot « internationalisation » est souvent abrégé par « i18n » ; cela est dû au fait que 18 lettres séparent le « i » du « n » dans ce mot si long à écrire ! Nous utiliserons également cette abréviation tout au long de ce chapitre.

Qu'est-ce que le i18n et comment s'en servir ?
----------------------------------------------

Avant de commencer, vous devez avoir installé gettext sur votre machine. 


### Sous Mac OS X

Vous devez [télécharger le code source](http://www.gnu.org/software/gettext/) et le compiler, ou installer le paquet gettext à partir des MacPorts.

### Sous Linux

Gettext est généralement installé par défaut. Si ce n'est pas le cas, cherchez un paquet nommé « gettext » adapté à votre distribution, ou procédez à l'installation manuelle, comme pour Mac OS X.

### Sous Windows

Voici la méthode d'installation complète :

 1. Téléchargez les archives suivantes [depuis le serveur GNOME](http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/), ou [un de ses miroirs](http://ftp.gnome.org/pub/GNOME/MIRRORS) (avec X supérieur ou égal à 0.15) :
    - `gettext-runtime-X.zip`
    - `gettext-tools-X.zip`

 2. Extrayez le contenu des deux dossiers dans un répertoire `\bin` commun (par exemple `C:\Program Files\gettext\bin`).
 3. Mettez à jour votre variable d'environnement `PATH` (voir le chapitre d'installation de Django pour plus d'informations) en ajoutant `;C:\Program Files\gettext\bin` à la fin de la valeur de la variable.

Vérifiez en ouvrant une console que la commande `xgettext --version` fonctionne sans erreur. En cas d'erreur, retéléchargez gettext via les liens précédents !

Pour commencer, nous devons nous attarder sur quelques définitions, afin d'avoir les idées au clair. Dans l'introduction, nous avons parlé d'_internationalisation_, dont le but est de traduire une application dans une autre langue. En réalité, ce travail se divise en deux parties :

 1. L'internationalisation (i18n) à proprement parler, qui consiste à adapter la partie technique (le code même de l'application) afin de permettre la traduction littérale par la suite ;
 2. La localisation (l10n), qui est la traduction (et parfois l'adaptation culturelle) de l'application.

La figure suivante schématise les différentes étapes.

![Cycle de traduction d'un logiciel](images/i18n_schema.png)

Tout au long de ce chapitre, nous allons suivre le déroulement du cycle de la figure précédente, dont tous les détails vous seront expliqués en temps et en heure. Mais tout d'abord, il nous faut configurer un peu notre projet, via `settings.py`.

Par défaut, lors de la création du projet, Django prédéfinit trois variables concernant l'internationalisation et la localisation :

```python
# Internationalization
# https://docs.djangoproject.com/en/1.7/topics/i18n/

LANGUAGE_CODE = 'fr-fr'

TIME_ZONE = 'UTC'

USE_I18N = True

USE_L10N = True
```

`LANGUAGE_CODE` permet de définir la langue par défaut utilisée dans vos applications. La variable `USE_I18N` permet d'activer l'internationalisation, et donc de charger plusieurs éléments en interne permettant son fonctionnement. Si votre site n'utilise pas l'internationalisation, il est toujours bon d'indiquer cette variable à `False`, afin d'éviter le chargement de modules inutilisés. De la même manière, `USE_L10N` permet d'automatiquement formater certaines données en fonction de la langue de l'utilisateur : les dates ou encore la représentation des nombres.

Par exemple, la ligne `{% now "DATETIME_FORMAT" %}` renvoie des résultats différents selon la valeur de `USE_L10N` et de `LANGUAGE_CODE` :

+--------------------------+------------------------------+------------------------+
|`LANGUAGE_CODE`/`USE_L10N`|`False`                       |`True`                  |
+--------------------------+------------------------------+------------------------+
|en-us                     |Jan. 30, 2014, 9:50 p.m.      |Jan. 30, 2014, 9:50 p.m.|
+--------------------------+------------------------------+------------------------+
|fr-fr                     |jan. 30, 2014, 9:50 après-midi|30 janvier 2014 21:50:46|
+--------------------------+------------------------------+------------------------+

Ensuite, nous devons préciser la liste des langues disponibles pour notre projet. Cette liste nous permet d'afficher un formulaire de choix, mais permet aussi à Django de limiter les choix possibles et donc d'éviter de chercher une traduction dans une langue qui n'existe pas. Cette liste se présente comme ceci :

```python
gettext = lambda x: x

LANGUAGES = (
   ('fr', gettext('French')),
   ('en', gettext('English')),
)
```

Au lieu d'importer la fonction `gettext`, introduite par la suite, nous avons créé une fausse fonction qui ne fait rien. En effet, il ne faut pas importer les fonctions de `django.utils.translation` dans notre fichier de configuration, car ce module dépend de notre fichier, ce qui créerait une boucle infinie dans les importations de modules !

Pour la suite de ce chapitre, il sera nécessaire d'avoir les réglages suivants dans votre `settings.py` :

```python
# Nous avons par défaut écrit l'application en français
LANGUAGE_CODE = 'fr-fr'

# Nous souhaitons générer des fichiers contenant les traductions, 
# afin de permettre à l'utilisateur de choisir sa langue par la suite
USE_I18N = True

# Nous adaptons les formats d'écriture de certains champs à la langue française
USE_L10N = True

gettext = lambda x: x

LANGUAGES = (
   ('fr', gettext('French')),
   ('en', gettext('English')),
)
```

Finalement, il nous faut encore ajouter deux lignes : un **middleware** et un **processeur de contexte**. Le premier permet de déterminer selon un certain ordre la langue courante de l'utilisateur. En effet, Django va tenter pour chaque visiteur de trouver la langue la plus adaptée en procédant par étape :

 1. Dans un premier temps, il est possible de configurer les URL pour les préfixer avec la langue voulue. Si ce préfixe apparaît, alors la langue sera forcée.
 2. Si aucun préfixe n'apparaît, le middleware vérifie si une langue est précisée dans la session de l'utilisateur.
 3. En cas d'échec, le middleware vérifie dans les cookies du visiteur si un cookie nommé `_language` (défini par Django) existe.
 4. En cas d'échec, il vérifie la requête HTTP et vérifie si l'en-tête `Accept-Language` est envoyé. Cet en-tête, envoyé par le navigateur du visiteur, spécifie les langues de prédilection, par ordre de priorité. Django essaie chaque langue une par une, selon celles disponibles dans notre projet.
 5. Enfin, si aucune de ces méthodes ne fonctionne, alors Django se rabat sur le paramètre `LANGUAGE_CODE`.

Le second, le _**template context processor**_, nous permettra d'utiliser les fonctions de traduction, via des tags, dans nos templates.

Pour activer correctement l'internationalisation, nous devons donc modifier la liste des middlewares et contextes à charger :

```python
MIDDLEWARE_CLASSES = (
   […] # Liste des autres middlewares déjà chargés
   'django.middleware.locale.LocaleMiddleware',
)
TEMPLATE_CONTEXT_PROCESSORS = (
   […] # Liste des autres template context déjà chargés
   "django.core.context_processors.i18n",
)
```

Nous en avons terminé avec les fichiers de configuration et pouvons dès lors passer à l'implémentation de l'internationalisation dans notre code !

Traduire les chaînes dans nos vues et modèles
---------------------------------------------

Adaptons notre code pour l'internationalisation. Nous devons gérer deux cas distincts : les vues et modèles, dans lesquels nous pouvons parfois avoir des chaînes de caractères à internationaliser, et nos templates, qui contiennent également de nombreuses chaînes de caractères à traduire. Notez qu'ici nous parlerons toujours de traduire dans une autre langue sans pour autant préciser laquelle. En effet, nous nous occupons de préciser ici ce qui doit être traduit, sans spécifier pour autant les différentes traductions.

Commençons par les vues. Pour rendre traduisibles les chaînes de caractères qui y sont présentes, nous allons appliquer une fonction à chacune. Cette fonction se chargera ensuite de renvoyer la bonne traduction selon la langue de l'utilisateur. Si la langue n'est pas supportée ou si la chaîne n'a pas été traduite, la chaîne sera alors affichée dans la langue par défaut.

La bibliothèque dont provient cette fonction spéciale est bien connue dans le domaine de l'internationalisation, puisqu'il s'agit de **gettext**, une bibliothèque logicielle dédiée à l'internationalisation. Dans Django, celle-ci se décompose en plusieurs fonctions que nous allons explorer dans cette partie :

 - `gettext` et `ugettext` ;
 - `gettext_lazy` et `ugettext_lazy` ;
 - `ngettext` et `ungettext` ;
 - `ngettext_lazy` et `ungettext_lazy` ;
 - Etc.

Avant d'explorer les fonctions une à une, sachez que nous allons tout d'abord diviser leur nombre par 2 : si vous regardez attentivement cette liste, vous remarquez que chacune existe avec et sans « u » comme première lettre. Les fonctions commençant par « u » signifient qu'elles supportent l'unicode, alors que les autres retournent des chaînes en ASCII. Nous partons du principe que nous utiliserons uniquement celles préfixées d'un « u », étant donné que la majorité de nos chaînes de caractères utilisent l'unicode.

Pour commencer, créons d'abord une nouvelle vue, qui renverra plusieurs chaînes de caractères au template :

```python
def test_i18n(request):
    nb_chats = 1
    couleur = "blanc"
    chaine = u"Bonjour les zéros !"
    ip = "Votre IP est %s" % request.META['REMOTE_ADDR']
    infos = "… et selon mes informations, vous avez %s chats %s !" % (nb_chats, couleur)

    return render(request, 'test_i18n.html', locals())
```

Et voici le fichier `test_i18n.html` :

```html
<p>
  Bonjour les zéros !<br />
  {{ chaine }}
</p>
<p>
  {{ ip }} {{ infos }}
</p>
```

[[information]]
| Tout au long de ce chapitre, nous ne vous indiquerons plus les directives de routage de `urls.py`. Vous devriez en effet être capables de faire cela vous-mêmes.

Ce fichier contient plusieurs chaînes écrites en français, qu'il faudra nécessairement traduire par la suite. Nous remarquons par ailleurs que certaines peuvent poser problème au niveau des formes du pluriel (dans le cas présent, il ne devrait pas y avoir de « s » à « chats », étant donné qu'il n'y en a qu'un seul !)

Tout d'abord, nous allons importer `ugettext`, afin de pouvoir utiliser la fonction dont nous avons parlé plus haut. Nous pourrions l'importer de la sorte :

```python
from django.utils.translation import ugettext
```

Cependant, nous serons amenés à utiliser très souvent cette méthode, et tout le temps sur le même type d'objet (des chaînes de caractères statiques). Vous verrez alors apparaître de nombreux `ugettext(…)` dans votre application, ce qui serait très redondant. Une habitude, presque devenue une convention, est d'imposer cette fonction avec l'alias `_` (_underscore_), afin de rendre votre code plus lisible :

```python
from django.utils.translation import ugettext as _
```

Dès lors, il ne nous reste plus qu'à appliquer cette méthode à nos chaînes :

```python
def test_i18n(request):
    nb_chats = 1
    couleur = "blanc"
    chaine = _("Bonjour les zéros !")
    ip = _("Votre IP est %s") % request.META['REMOTE_ADDR']
    infos = _("… et selon mes informations, vous avez %s chats %s !") % (nb_chats, couleur)

    return render(request, 'test_i18n.html', locals())
```

En testant ce code, rien ne change à l'affichage de la page. En effet, la fonction `gettext` n'a aucun effet pour le moment, puisque nous utilisons la langue par défaut à l'affichage. Avant d'aller plus loin, il convient de préciser que ce code n'est en réalité pas totalement correct. Dans un premier temps, les pluriels ne s'accordent pas en fonction de la valeur de `nb_chats`, et cela même dans la langue par défaut. Pour corriger cela, il faut utiliser une autre fonction, cousine de `gettext`, qui est `ungettext`, dans le même module. Cette fonction permet de fournir une chaîne pour le cas singulier, une pour le cas pluriel et enfin la variable déterminant le cas à afficher. Pour le cas de nos amis les félins, cela donne :

```python
from django.utils.translation import ugettext as _
from django.utils.translation import ungettext

def test_i18n(request):
    nb_chats = 2
    couleur = "blanc"
    chaine = _("Bonjour les zéros !")
    ip = _("Votre IP est %s") % request.META['REMOTE_ADDR']
    infos = ungettext("… et selon mes informations, vous avez %s chat %s !",
                      "… et selon mes informations, vous avez %s chats %ss !",
                      nb_chats) % (nb_chats, couleur)

    return render(request, 'test_i18n.html', locals())
```

Vous pouvez déjà tester, en changeant la variable `nb_chats` de 2 à 1, les « s » après « chat » et « blanc » disparaissent.

Cependant, nous avons encore un autre problème. Lorsque nous imaginons une traduction en anglais de cette chaîne, une des solutions pour le cas singulier serait : « … and according to my information, you have %s %s cat », afin d'avoir « … and according to my information, you have 1 white cat » par exemple. Cependant, en français, l'adjectif se situe après le nom contrairement à l'anglais où il se situe avant. Cela nécessite d'inverser l'ordre de nos variables à l'affichage : dans le cas présent nous allons plutôt obtenir « you have white 1 cat » !

Pour ce faire, il est possible de nommer les variables au sein de la chaîne de caractères, ce que nous vous recommandons de faire tout le temps, même si vous n'avez qu'une variable dans votre chaîne. Une troisième version de notre vue est donc :

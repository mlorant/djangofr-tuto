Les templates
============= 

Nous avons vu comment créer une vue et renvoyer du code HTML à l'utilisateur. Cependant, la méthode que nous avons utilisée n'est pas très pratique, le code HTML était en effet intégré à la vue elle-même ! Le code Python et le code HTML deviennent plus difficiles à éditer et à maintenir pour plusieurs raisons :

- Les indentations HTML et Python se confondent ;
- La coloration syntaxique de votre éditeur favori ne fonctionnera généralement pas pour le code HTML, celui-ci n'étant qu'une simple chaîne de caractères ;
- Si vous avez un designer dans votre projet, celui-ci risque de casser votre code Python en voulant éditer le code HTML ;
- Etc.

C'est à cause de ces raisons que tous les frameworks web actuels utilisent un moteur de templates. Les templates sont écrits dans un mini-langage de programmation propre à Django et qui possède des expressions et des structures de contrôle basiques (`if`/`else`, boucle `for`, etc.) que nous appelons des tags. Le moteur transforme les tags qu'il rencontre dans le fichier par le rendu HTML correspondant. Grâce à ceux-ci, il est possible d'effectuer plusieurs actions algorithmiques : afficher une variable, réaliser des conditions ou des boucles, faire des opérations sur des chaînes de caractères, etc.

Lier templates et vue
---------------------
Avant d'aborder le cœur même du fonctionnement des templates, retournons brièvement vers les vues. Dans la première partie, nous avons vu que nos vues étaient liées à des templates (et des modèles), comme le montre la figure suivante.

![Schéma d'exécution d'une requête](images/requete-schema.png "Schéma d'exécution d'une requête")

C'est la vue qui se charge de transmettre l'information de la requête au template, puis de retourner le HTML généré au client. Dans le chapitre précédent, nous avons utilisé la méthode `HttpResponse(text)` pour renvoyer le HTML au navigateur. Cette méthode prend comme paramètre une chaîne de caractères et la renvoie sous la forme d'une réponse HTTP. La question ici est la suivante : comment faire pour appeler notre template, et générer la réponse à partir de celui-ci ? La fonction `render` a été conçue pour résoudre ce problème.

<div class="info">La fonction `render` est en réalité une méthode de `django.shortcut` qui nous simplifie la vie : elle génère un objet `HttpResponse` après avoir traité notre template. Pour les puristes qui veulent savoir comment cela fonctionne en interne, n'hésitez pas à aller fouiller dans [la documentation officielle](https://docs.djangoproject.com/en/1.7/ref/templates/api/#using-the-template-system).</div>

Nous allons commencer par quelques exemples avec deux vues. La première renvoie juste la date actuelle à l'utilisateur, alors que la seconde va effectuer l'addition de deux entiers.


	from datetime import datetime
	from django.shortcuts import render
	
	def date_actuelle(request):
	    return render(request, 'blog/date.html', {'date': datetime.now()})

    def addition(request, nombre1, nombre2):	
        total = int(nombre1) + int(nombre2)
    
        # retourne nombre1, nombre2 et la somme des deux
        return render(request, 'blog/addition.html', locals())

----------

	url(r'^date$', 'date_actuelle'),
    url(r'^addition/(?P<nombre1>\d+)/(?P<nombre2>\d+)/$', 'addition'),

Comme vous pouvez le voir, la fonction `render` prend en argument trois paramètres :

- La requête initiale, qui a permis de construire la réponse (`request` dans notre cas) ;
- Le chemin vers le template adéquat dans une application ou un des dossiers de templates donnés dans `settings.py` ;
- Un dictionnaire reprenant les variables qui seront accessibles dans le template.

Avant d'écrire notre template, il faut d'abord se demander où est-ce que l'on va l'enregistrer. Par défaut, Django va chercher les templates aux endroits suivants :

1. Dans la liste des dossiers fournis dans la variable de configuration `TEMPLATES_DIR`.
2. S'il ne l'a pas trouvé, dans le dossier `templates` de l'application en cours

Nous n'avons pas parlé de `TEMPLATES_DIR` lors de la configuration de notre projet, car cette variable n'est pas présente par défaut dans le fichier `settings.py`. En réalité, il vous est possible de définir une liste de dossiers où Django ira chercher les templates en priorité. Classiquement, cette liste est composée d'un dossier `templates` à la racine de votre projet : 

    TEMPLATE_DIRS = (
        os.path.join(BASE_DIR, 'templates'),
    )

Nous vous conseillons de créer un dossier `templates` à la racine du projet. Vous pourrez y déposer des templates plutôt propres à votre projet (erreurs 404, squelette de votre design, pages statiques...).

Pour nos applications, nous allons utiliser la deuxième catégorie : le dossier `templates` de l'application actuelle. En effet, il est préférable de conserver les templates propre à une application dans son dossier, afin de permettre la réusabilité de l'application. Si vous souhaitez partager votre application, tout marchera sans devoir déplacer les templates en fonction de l'installation.

Enfin, pour éviter les conflits, l'usage est de créer un dossier du nom de l'application au sein du dossier `templates`. On obtient alors la hiérarchie suivante : 

    crepes_bretonnes/
            blog/
                    __init__.py
                    admin.py
                    migrations/
                        __init__.py
                    models.py
                    templates/
                        blog/
                            addition.html
                            date.html
                    tests.py
                    views.py
            crepes_bretonnes/
                    __init__.py
                    settings.py
                    urls.py
                    wsgi.py
            templates/
            db.sqlite3
            manage.py

L'avantage de cette structure est que les personnes qui utiliseraient votre application pourrait surcharger votre template en écrivant un nouveau dans leur propre dossier `templates/` global !

Maintenant que nous avons notre structure, créeons nos fichiers templates. Occupons-nous d'abord de `date.html`, le plus simple. Créez ce nouveau fichier dans `blog/templates/blog` :

    <h1>Bienvenue sur mon blog</h1>
    <p>La date actuelle est : {{ date }}</p>

Nous retrouvons `date`, comme passé dans `render()` ! Si vous accédez à cette page (après lui avoir assigné une URL), le `{{ date }}` est bel et bien remplacé par la date actuelle !  
Dans ce fichier, nous avons accès qu'à cette variable, puisque c'est la seule que nous avons passé au template via la fonction `render`.

<div class="warning">Si vous obtenez l'exception `TemplateDoesNotExist`, spécifiant que `blog/date.html` n'existe pas, vérifiez que l'application `blog` est bien dans `INSTALLED_APPS`, dans le fichier `settings.py`. Django ne va chercher les templates que dans les applications installées !</div>

On peut rapidement observer le second exemple avec le template `addition.html` :

    <h1>Ma super calculatrice</h1>
    <p>{{ nombre1 }} + {{ nombre2 }}, ça fait <strong>{{ total }}</strong> !<br />
    Nous pouvons également calculer la somme dans le template : {{ nombre1|add:nombre2 }}.</p>

Nous expliquerons bientôt les structures présentes dans ce template, ne vous inquiétez pas. Cependant, en allant sur `http://127.0.0.1:8000/blog/addition/5/3`, vous obtiendrez le résultat de l'addition 5 + 3.

![Exemple d'appel à notre page d'addition](images/1er-template.png "Exemple d'appel à notre page d'addition")

<div class="warning">Encore une fois, un piège peut se glisser ici. Du fait des accents, faites attention à ce que votre fichier de templates soit encodé en **UTF8**, sinon vous pouvez obtenir l'erreur suivante :

    UnicodeDecodeError at /blog/addition/5/3/

    'utf8' codec can't decode byte 0xe7 in position 73: invalid continuation byte

Ceci est dû à la présence d'accent dans le template, cher à notre langue française.
</div>

La seule différence dans la vue réside dans le deuxième argument donné à `render`. Au lieu de lui passer un dictionnaire directement, nous faisons appel à la fonction `locals()` qui va retourner un dictionnaire contenant toutes les variables locales de la fonction depuis laquelle `locals()` a été appelée. Les clés seront les noms de variables (par exemple `total`), et les valeurs du dictionnaire seront tout simplement… les valeurs des variables de la fonction ! Ainsi, si `nombre1` valait 42, la valeur `nombre1` du dictionnaire vaudra elle aussi 42. Le dictionnaire pour notre seconde vue est semblable à ceci :

    {'total': 8, 'nombre1': 5, 'nombre2': 3, 'request': <WSGIRequest ...>}

Affichons nos variables à l'utilisateur 
---------------------------------------

### Affichage d'une variable

Comme nous l'avons déjà expliqué, la vue transmet au template les données destinées à l'utilisateur. Ces données correspondent à des variables classiques de la vue. Nous pouvons les afficher dans le template grâce à l'expression `{{ }}` qui prend à l'intérieur des accolades un argument (on pourrait assimiler cette expression à une fonction), le nom de la variable à afficher. Le nom des variables est également limité aux caractères alphanumériques et aux underscores.

    Bonjour {{ pseudo }}, nous sommes le {{ date }}.

Ici, nous considérons que la vue a transmis deux variables au template : `pseudo` et `date`. Ceux-ci seront affichés par le moteur de template. Si pseudo vaut « Jean » et date « 28 décembre », le moteur de templates affichera « _Bonjour Jean, nous sommes le 28 décembre._ ».

Si jamais la variable n'est pas une chaîne de caractères, le moteur de templates utilisera la méthode `__str__` de l'objet pour l'afficher. Par exemple, les listes seront affichés sous la forme `['element 1', 'element 2'...]`, comme si vous demandiez son affichage dans une console Python. Il est possible d'accéder aux attributs d'un objet comme en Python, en les juxtaposant avec un point. Plus tard, nos articles de blog seront représentés par des objets, avec des attributs `titre`, `contenu`, etc. Pour y accéder, la syntaxe sera la suivante :

    {# Nous supposons que notre vue a fourni un objet nommé article contenant les attributs titre, auteur et contenu #}
    <h2>{{ article.titre }}</h2>
    <p><em>Article publié par {{ article.auteur }}</em></p>
    
    <p>{{ article.contenu }}</p>

<div class="info">Si jamais une variable n'existe pas, ou n'a pas été envoyée au template, la valeur qui sera affichée à sa place est celle définie par `TEMPLATE_STRING_IF_INVALID` dans votre `settings.py`, qui est une chaîne vide par défaut.</div>

### Les filtres

Les filtres permettent de modifier l'affichage en fonction d'une variable, sans passer par la vue. Prenons un exemple concret : sur la page d'accueil des sites d'actualités, le texte des dernières nouvelles est généralement tronqué, seul le début est affiché. Pour réaliser la même chose avec Django, nous pouvons utiliser un filtre qui limite l'affichage aux 80 premiers mots de notre article :

    {{ texte|truncatewords:80 }}

Ici, le filtre `truncatewords` (qui prend comme paramètre un nombre, séparé par un deux-points) est appliqué à la variable `texte`. À l'affichage, cette dernière sera tronquée et l'utilisateur ne verra que les 80 premiers mots de celle-ci.

Ces filtres ont pour but d'effectuer des opérations de façon claire, afin d'alléger les vues, et ne marchent que lorsqu'une variable est affichée (avec la structure `{{ }}` donc). Il est par exemple possible d'accorder correctement les phrases de votre site avec le filtre `pluralize` :

    Vous avez {{ nb_messages }} message{{ nb_messages|pluralize }}.

Dans ce cas, un « s » sera ajouté si le le nombre de messages est supérieur à 1. Il est possible de passer des arguments au filtre afin de coller au mieux à notre chère langue française :

    Il y a {{ nb_chevaux }} chev{{ nb_chevaux|pluralize:"al,aux" }} dans l'écurie.

Ici, nous aurons « cheval » si `nb_chevaux` est égal à 1 et « chevaux » pour le reste.

Et un dernier pour la route : imaginons que vous souhaitiez afficher le pseudo du membre connecté, ou le cas échéant « visiteur ». Il est possible de le faire en quelques caractères, sans avoir recours à une condition !

    Bienvenue {{ pseudo|default:"visiteur" }}

En bref, il existe des dizaines de filtres par défaut : safe, length, etc. Tous les filtres sont répertoriés et expliqués dans [la documentation officielle de Django](https://docs.djangoproject.com/en/1.7/ref/templates/builtins/#built-in-filter-reference), n'hésitez pas à y jeter un coup d'œil pour découvrir d'éventuels filtres qui pourraient vous être utiles.

Manipulons nos données avec les tags 
------------------------------------

Abordons maintenant le second type d'opération implémentable dans un template : les tags. C'est grâce à ceux-ci que les conditions, boucles, etc. sont disponibles.
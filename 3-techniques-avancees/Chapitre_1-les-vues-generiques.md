Les vues génériques
===================

Sur la plupart des sites web, il existe certains types de pages où créer une vue comme nous l'avons fait précédemment est lourd et presque inutile : pour une simple page HTML sans informations dynamiques par exemple, ou encore de simple listes d'objets sans traitements particuliers. Django est conçu pour n'avoir à écrire que le *minimum* (philosophie DRY), le framework inclut donc un système de vues génériques, qui évite au développeur de devoir écrire des fonctions simples et identiques, nous permettant de gagner du temps et des lignes de code.

Ce chapitre est assez long et dense en informations. Nous vous conseillons de lire en plusieurs fois : nous allons faire plusieurs types distincts de vues, qui ont chacune une utilité différente. Il est tout à fait possible de poursuivre ce cours sans connaître tous ces types.

Premiers pas avec des pages statiques 
-------------------------------------

Les vues génériques sont en quelque sorte des vues très modulaires, prêtes à être utilisées directement, incluses par défaut dans le framework et cela sans devoir écrire la vue elle-même. Pour illustrer le fonctionnement global des vues génériques, prenons cet exemple de vue classique, qui ne s'occupe que d'afficher un template à l'utilisateur, sans utiliser de variables : 


    from django.shortcuts import render
    
    def faq(request):
        return render(request, 'blog/faq.html', {})

... et son URL associée :

    url('faq', 'blog.views.faq', name='faq'),

Une première caractéristique des vues génériques est que ce ne sont pas des fonctions, comme la vue que nous venons de présenter, mais des classes. L'amalgame *1 vue = 1 fonction* en est du coup quelque peu désuet. 

Il existe deux méthodes principales d'utilisation pour les vues génériques :

1. Soit nous créons une classe, héritant d'un type de vue générique dont nous surchargerons les attributs ;
2. Soit nous appelons directement la classe générique, en passant en arguments les différentes informations à utiliser.

Toutes les classes de vues génériques sont situées dans `django.views.generic`. Un premier type de vue générique est `TemplateView`. Typiquement, `TemplateView` permet, comme son nom l'indique, de créer une vue qui s'occupera du rendu d'un template.  
Comme dit précédemment, créons une classe héritant de `TemplateView`, et surchargeons ses attributs :

    from django.views.generic import TemplateView
    
    class FAQView(TemplateView):
        template_name = "blog/faq.html"  # chemin vers le template à afficher

Dès lors, il suffit de router notre URL vers une méthode héritée de la classe `TemplateView`, nommée `as_view` :

    from django.conf.urls import patterns, url, include
    from blog.views import FAQView  # N'oubliez pas d'importer la classe mère
    
    urlpatterns = patterns('',
       (r'^faq/$', FAQView.as_view()),   # Nous demandons la vue correspondant à la classe FAQView
    )

C'est tout ! Lorsqu'un visiteur accède à `/blog/faq/`, le contenu du fichier `templates/blog/faq.html` sera affiché. En réalité, la méthode `as_view` de `FAQView` retourne une vue (donc une fonction classique) qui se basera sur ses attributs pour déterminer son fonctionnement. Étant donné que nous avons indiqué un template à utiliser depuis l'attribut `template_name`, la classe l'utilisera pour générer une vue adaptée.

Nous avons indiqué précédemment qu'il y avait deux méthodes pour utiliser les vues génériques. Le principe de la seconde est de directement instancier `TemplateView` dans le fichier `urls.py`, en lui passant en argument notre `template_name` :

    from django.conf.urls import patterns, url, include
    from django.views.generic import TemplateView  # L'import a changé, attention !
    
    urlpatterns = patterns('',
       url(r'^faq/', TemplateView.as_view(template_name='blog/faq.html')),
    )

Vous pouvez alors retirer `FAQView`, la classe ne sert plus à rien. Pour les `TemplateView`, la première méthode présente peu d'intérêt, cependant nous verrons par la suite qu'hériter d'une classe sera plus facile que tout définir dans `urls.py`.

Lister et afficher des données 
------------------------------

Jusqu'ici, nous avons vu comment *afficher des pages statiques* avec des vues génériques. Bien que ce soit pratique, il n'y a jusqu'ici rien de très puissant.

Abordons maintenant quelque chose de plus intéressant. Un schéma utilisé presque partout sur le web est le suivant : vous avez une liste d'objets (des articles, des images, etc.), et lorsque vous cliquez sur un élément, vous êtes redirigés vers une page présentant plus en détail ce même élément.

Nous avons déjà réalisé quelque chose de semblable dans le chapitre sur les modèles précédemment avec notre liste d'articles et l'affichage individuel d'articles. Nous allons repartir de la même idée, mais cette fois-ci avec des vues génériques. Pour ce faire, nous utiliserons deux nouvelles classes : `ListView` et `DetailView`. Nous réutiliserons les deux modèles `Article` et `Categorie`, qui ne changeront pas.

### Une liste d'objets en quelques lignes avec ListView

Commençons par une simple liste de nos articles, sans pagination. À l'instar de `TemplateView`, nous pouvons utiliser `ListView` directement en lui passant en paramètre le modèle à traiter :

    from django.conf.urls import patterns, url, include
    from django.views.generic import ListView
    from blog.models import Article
    
    urlpatterns = patterns('',
        # Nous allons réécrire l'URL de l'accueil
        url(r'^$', ListView.as_view(model=Article,)),
    
        # Et nous avons toujours nos autres pages…
        url(r'^article/(?P<id>\d+)$', 'blog.views.lire'),
        url(r'^(?P<page>\d+)$', 'blog.views.archives'),
        url(r'^categorie/(?P<slug>.+)$', 'blog.views.voir_categorie'),
    )

Avec cette méthode, Django impose quelques conventions :

- Le template devra s'appeler `<app>/<model>_list.html`. Dans notre cas, le template serait nommé `blog/article_list.html`.
- L'unique variable retournée par la vue générique et utilisable dans le template est appelée `object_list`, et contiendra ici tous nos articles.

Il est possible de redéfinir ces valeurs en passant des arguments supplémentaires à notre `ListView` : 

    urlpatterns = patterns('',
        url(r'^$', ListView.as_view(model=Article,
                        context_object_name="derniers_articles",
                        template_name="blog/accueil.html")),
        ...
    )

Par souci d'économie, nous souhaitons réutiliser le template `blog/accueil.html` qui utilisait comme nom de variable `derniers_articles` à la place d'`object_list`, celui par défaut de Django.

Vous pouvez dès lors supprimer la fonction `accueil` dans `views.py`, et vous obtiendrez le même résultat qu'avant (ou presque, si vous avez plus de 5 articles) ! L'ordre d'affichage des articles est celui défini dans le modèle, via l'attribut `ordering` de la sous-classe `Meta`, qui se base par défaut sur la clé primaire de chaque entrée. 

Il est possible d'aller plus loin : nous ne souhaitons généralement pas tout afficher sur une même page, mais par exemple filtrer les articles affichés. Il existe donc plusieurs attributs et méthodes de `ListView` qui étendent les possibilités de la vue.

Par souci de lisibilité, nous vous conseillons plutôt de renseigner les classes dans `views.py`, comme vu précédemment. Tout d'abord, changeons notre `urls.py`, pour appeler notre nouvelle classe :

    from blog.views import ListeArticles
    
    urlpatterns = patterns('',
        url(r'^$', ListeArticles.as_view(), name="blog_liste"),  # Via la fonction as_view, comme vu tout à l'heure
        ...
    )

C'est l'occasion d'expliquer l'intérêt de l'argument `name`. Pour profiter au maximum des possibilités de Django, il est possible d'écrire les URL via la fonction `reverse`, et son tag associé dans les templates. L'utilisation du tag `url` se fera dès lors ainsi : {% url "blog_liste" %}. Cette fonctionnalité ne dépend pas des vues génériques, mais est inhérente au fonctionnement des URL en général. Vous pouvez donc également associer le paramètre `name` à une vue normale.

Ensuite, créons notre classe qui reprendra les mêmes attributs que notre `ListView` de tout à l'heure :

    class ListeArticles(ListView):
        model = Article
        context_object_name = "derniers_articles"
        template_name = "blog/accueil.html"


Désormais, nous souhaitons paginer nos résultats, afin de n'afficher que 5 articles par page, par exemple. Il existe un attribut adapté : 

    class ListeArticles(ListView):
        model = Article
        context_object_name = "derniers_articles"
        template_name = "blog/accueil.html"
        paginate_by = 5

De cette façon, la page actuelle est définie via l'argument page, passé dans l'URL (`/?page=2` par exemple). Il suffit dès lors d'adapter le template pour faire apparaître la pagination. Sachez que vous pouvez définir le style de votre pagination dans un template séparé, et l'inclure à tous les endroits nécessaires, via `{% include 'pagination.html' %}` par exemple. Nous parlerons plus loin du fonctionnement de la pagination, dans la partie IV.

    <h1>Bienvenue sur le blog des crêpes bretonnes !</h1>
    
    {% for article in derniers_articles %}
        <div class="article">
           <h3>{{ article.titre }}</h3>
           <p>{{ article.contenu|truncatewords_html:80 }}</p>
           <p><a href="{% url "blog.views.lire" article.id %}">Lire la suite</a>
        </div>
    {% endfor %}
    
    {# Mise en forme de la pagination ici #}
    {% if is_paginated %}
        <div class="pagination">
               {% if page_obj.has_previous %}
                   <a href="?page={{ page_obj.previous_page_number }}">Précédente</a> —
               {% endif %}
               Page {{ page_obj.number }} sur {{ page_obj.paginator.num_pages }} 
               {% if page_obj.has_next %}
                  — <a href="?page={{ page_obj.next_page_number }}">Suivante</a>
               {% endif %}
        </div>
    {% endif %}

Allons plus loin ! Nous pouvons également surcharger la sélection des objets à récupérer, et ainsi soumettre nos propres filtres :

    class ListeArticles(ListView):
        model = Article
        context_object_name = "derniers_articles"
        template_name = "blog/accueil.html"
        paginate_by = 5
        queryset = Article.objects.filter(categorie__id=1)

Ici, seuls les articles de la première catégorie créée seront affichés. Vous pouvez bien entendu effectuer des requêtes identiques à celle des vues, avec du tri, plusieurs conditions, etc. Il est également possible de passer des arguments pour rendre la sélection un peu plus dynamique en ajoutant l'ID souhaité dans l'URL.

    from django.conf.urls import patterns, url
    from blog.views import ListeArticles
    
    urlpatterns = patterns('blog.views',
        url(r'^categorie/(\d+)$', ListeArticles.as_view(), name='blog_categorie'),
        url(r'^article/(?P<id>\d+)$', 'lire'),
        url(r'^(?P<page>\d+)$', 'archives')
    )

Dans la vue, nous sommes obligés de surcharger `get_queryset`, qui renvoie la liste d'objets à afficher. En effet, il est impossible d'accéder aux paramètres lors de l'assignation d'attributs comme nous le faisons depuis le début.

    class ListeArticles(ListView):
        model = Article
        context_object_name = "derniers_articles"
        template_name = "blog/accueil.html"
        paginate_by = 10
    
        def get_queryset(self):
           return Article.objects.filter(categorie__id=self.args[0])

Tâchez tout de même de vous poser des limites, le désavantage ici est le suivant : la lecture de votre `urls.py` devient plus difficile, tout comme votre vue (imaginez si vous avez quatre arguments, à quoi correspond le deuxième ? le quatrième ?).

Enfin, il est possible d'ajouter des éléments au contexte, c'est-à-dire les variables qui sont renvoyées au template. Par exemple, renvoyer l'ensemble des catégories, afin de faire une liste de liens vers celles-ci. Pour ce faire, nous allons ajouter au tableau `context` une clé `categories` qui contiendra notre liste :

    def get_context_data(self, **kwargs):
           # Nous récupérons le contexte depuis la super-classe
           context = super().get_context_data(**kwargs)
           # Nous ajoutons la liste des catégories, sans filtre particulier
           context['categories'] = Categories.objects.all()
           return context

Il est facile d'afficher la liste des catégories dans notre template :

    <h3>Catégories disponibles</h3>
    <ul>
    {% for categorie in categories %}
        <li><a href="{% url "blog_categorie" categorie.id %}">{{ categorie.nom }}</a></li>
    {% endfor %}
    </ul>

Afficher un article via DetailView
----------------------------------

Malgré tout cela, nous ne pouvons afficher que des listes, et non pas un objet précis. Heureusement, la plupart des principes vus précédemment avec les classes héritant de `ListView` sont applicables avec celles qui héritent de `DetailView`. Le but de `DetailView` est de renvoyer un seul objet d'un modèle, et non une liste. Pour cela, il va falloir passer un paramètre bien précis dans notre URL : `pk`, qui représentera *la clé primaire de l'objet à récupérer* :

    from blog.views import ListeArticles, LireArticle
    
    urlpatterns = patterns('blog.views',
        url(r'^categorie/(\w+)$', ListeArticles.as_view(), name='blog_categorie'),
        url(r'^article/(?P<pk>\d+)$', LireArticle.as_view(), name='blog_lire'),
    )

Maintenant que nous avons notre URL, avec la clé primaire en paramètre, il nous faut écrire la classe qui va récupérer l'objet voulu et le renvoyer à un template précis :

    from django.views.generic import DetailView 

    class LireArticle(DetailView):
        context_object_name = "article"
        model = Article
        template_name = "blog/lire.html"

… et encore une fois l'ancienne vue devient inutile. Souvenez-vous que notre fonction `lire` gérait le cas où l'ID de l'article n'existait pas, il en est de même ici. Comme tout à l'heure, vu que nous avons nommé notre objet `article`, il n'y a *aucune modification à faire* dans le template :

    <h1>{{ article.titre }} <span class="small">dans {{ article.categorie.nom }}</span></h1>
    <p class="infos">Rédigé par {{ article.auteur }}, le {{ article.date|date:"DATE_FORMAT" }}</p>
    <div class="contenu">{{ article.contenu|linebreaks }}</div>

Comme pour les `ListView`, il est possible de personnaliser la sélection avec get_queryset, afin de ne sélectionner l'article que s'il est public par exemple. Une autre spécificité utile lorsque nous affichons un objet, c'est d'avoir la possibilité de modifier un de ses attributs, par exemple son nombre de vues ou sa date de dernier accès. Pour faire cette opération, il est possible de surcharger la méthode `get_object`, qui renvoie l'objet à afficher :

    class LireArticle(DetailView):
        context_object_name = "article"
        model = Article
        template_name = "blog/lire.html"
    
        def get_object(self):
            # Nous récupérons l'objet, via la super-classe
            article = super().get_object()
        
            article.nb_vues += 1  # Imaginons que nous ayons un attribut « Nombre de vues »
            article.save()
        
            return article

Enfin, sachez que la variable `request`, qui contient les informations sur la requête et l'utilisateur, est également disponible dans les vues génériques. C'est un *attribut de la classe*, que vous pouvez donc appeler dans n'importe quelle méthode via `self.request`.

Agir sur les données
--------------------

Jusqu'ici, nous n'avons fait qu'*afficher des données*, statiques ou en provenance de modèles. Nous allons maintenant nous occuper de la gestion de données. Pour cette partie, nous reprendrons comme exemple notre application de raccourcisseur d'URL que nous avons développée lors du chapitre précédent. Pour rappel, dans le [schéma CRUD](https://fr.wikipedia.org/wiki/CRUD), il y a quatre types d'actions applicables sur une donnée :

- `Create` (créer) ;
- `Read` (lire, que nous avons déjà traité juste au-dessus) ;
- `Update` (mettre à jour) ;
- `Delete` (supprimer).

Nous montrerons comment réaliser ces opérations dans l'ordre indiqué. En effet, chacune possède une vue générique associée.
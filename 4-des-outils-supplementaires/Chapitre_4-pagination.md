La pagination 
=============

Un autre outil fréquemment utilisé sur les sites web de nos jours est la pagination. La pagination est le fait de *diviser une liste d'objets en plusieurs pages*, afin d'alléger la lecture d'une page (et son chargement). Nous allons voir dans ce chapitre comment réaliser une pagination avec les outils que Django propose.

Exerçons-nous en console 
------------------------

Django permet de _répartir des ensembles d'objets sur plusieurs pages_ : des listes, des `QuerySet`, etc. En réalité, tous les objets ayant une méthode `count` ou `__len__` sont acceptés.

Notre premier exemple utilisera une simple liste et sera effectué dans l'interpréteur interactif. Ouvrez une console et tapez la commande `python manage.py shell` pour lancer l'interpréteur.

Django fournit une classe nommée `Paginator` qui effectue la pagination. Elle se situe dans le module `django.core.paginator`. Nous devons donc en premier lieu l'importer et créer une liste quelconque, ici de grandes villes :

```python
>>> from django.core.paginator import Paginator
>>> villes = ['Tokyo', 'Mexico', 'Seoul', 'New York', 'Bombay', 'Karachi', 
'Sao Paulo', 'Manille', 'Bangkok', 'New Delhi', 'Djakarta', 'Shanghai',
'Los Angeles', 'Kyoto', 'Le Caire', 'Calcutta', 'Moscou', 'Istanbul',
'Buenos Aires', 'Dacca', 'Gauteng', 'Teheran', 'Pekin']
```

La classe `Paginator` est instanciable avec deux paramètres : la liste d'objets à répartir et le nombre maximum d'objets à afficher par page. Imaginons que nous souhaitions afficher 5 villes par page, nous pouvons instancier la classe de la manière suivante :

```python
>>> p = Paginator(villes, 5)
```

Cet objet possède les attributs suivants :

```python
>>> p.count      # Nombre d'objets au total, toutes pages confondues
23
>>> p.num_pages  # Nombre de pages nécessaires pour répartir toutes les villes
5                # En effet, 4 pages avec 5 villes et 1 page avec 3 villes

>>> p.page_range  # La liste des pages disponibles
[1, 2, 3, 4, 5]
```

Nous pouvons obtenir les villes d'une page précise grâce la méthode `page()`. Cette méthode renvoie un objet `Page`, dont voici les méthodes principales :

```python
>>> page1 = p.page(1)  # Renvoie un objet Page pour notre première page
>>> page1
<Page 1 of 5>
>>> page1.object_list      # Le contenu de cette première page
['Tokyo', 'Mexico', 'Seoul', 'New York', 'Bombay']  
>>> p.page(5).object_list  # Même opération pour la cinquième page
['Gauteng', 'Teheran', 'Pekin']
>>> page1.has_next()      # Est-ce qu'il y a une page suivante ?
True   
>>> page1.has_previous()  # Est-ce qu'il y a une page précédente ?
False  
```

Soyez vigilants, la *numérotation des pages commence bien à 1*, et non pas à 0 comme pour les listes par exemple. La classe renvoie plusieurs types d'exceptions en cas d'erreur à la récupération d'une page :

```python
>>> p.page(0)
Traceback (most recent call last):
   […]
django.core.paginator.EmptyPage: That page number is less than 1
>>> p.page(6)
Traceback (most recent call last):
   […]
django.core.paginator.EmptyPage: That page contains no results
>>> p.page('abc') 
Traceback (most recent call last):
  […]
django.core.paginator.PageNotAnInteger: That page number is not an integer
```

Avant d'attaquer l'utilisation de la pagination dans nos vues et templates, étudions deux autres situations permettant de compléter au mieux notre système de pagination. Il y a deux arguments de `Paginator` que nous n'avons pas traités. En effet, le constructeur complet de `Paginator` accepte deux paramètres optionnels.

Tout d'abord, le paramètre `orphans` permet de préciser le nombre minimum d'éléments qu'il faut pour afficher une dernière page. Si le nombre d'éléments est inférieur au nombre requis, alors ces éléments sont déportés sur la page précédente (qui devient elle-même la dernière page), en plus des éléments qu'elle contient déjà. Prenons notre exemple précédent :

```python
>>> p = Paginator(villes, 10, orphans=5)
>>> p.num_pages
2
>>> p.page(1).object_list
['Tokyo', 'Mexico', 'Seoul', 'New York', 'Bombay', 'Karachi', 'Sao Paulo', 'Manille', 'Bangkok', 'New Delhi']
>>> p.page(2).object_list
['Djakarta', 'Shanghai', 'Los Angeles', 'Kyoto', 'Le Caire', 'Calcutta', 'Moscou', 'Istanbul', 'Buenos Aires', 'Dacca', 'Gauteng', 'Teheran', 'Pekin']
```

Nous voyons que la dernière page théorique (la 3^e^) aurait du contenir 3 éléments (`Gauteng`, `Teheran` et `Pekin`), ce qui est inférieur à 5. Ces éléments sont donc affichés en page 2, qui devient la dernière, avec 13 éléments.

Le second attribut optionnel, `allow_empty_first_page`, permet de lancer une exception si la première page est vide. Autrement dit, une exception est levée s'il n'y a aucun élément à afficher. Un exemple est encore une fois plus parlant :

```python
# Nous initialisons deux Paginator avec une liste vide
>>> pagination_avec_vide = Paginator([], 42)  
>>> pagination_sans_vide = Paginator([], 42, allow_empty_first_page=False)
  
>>> pagination_avec_vide.page(1)  # Comportement par défaut si la liste est vide
<Page 1 of 1>
>>> pagination_avec_vide.page(1).object_list
[]
>>> pagination_sans_vide.page(1) 
Traceback (most recent call last):
 […]
django.core.paginator.EmptyPage: That page contains no results
```

Nous avons désormais globalement fait le tour, place à la pratique !

Utilisation concrète dans une vue 
---------------------------------

Nous avons vu comment utiliser la pagination de façon autonome, maintenant nous allons l'utiliser dans un cas concret. Nous reprenons notre vue simple (sans les vues génériques) du TP sur la minification d'URL :

```python
def liste(request):
    """Affichage des redirections"""
    minis = MiniURL.objects.order_by('-nb_acces')

    return render(request, 'mini_url/liste.html', locals())
```

Nous allons tout d'abord ajouter un argument `page` à notre vue, afin de savoir quelle page l'utilisateur souhaite voir. Pour ce faire, il y a deux méthodes :

 - Passer le paramètre `page` via un paramètre `GET` (`/url/?page=1`) ;
 - Modifier la définition de l'URL et la vue pour prendre en compte un numéro de page (`/url/1` pour la première page).

Nous traiterons ici le second cas. Le premier cas se résume à un simple `request.GET.get('page')` dans la vue pour récupérer le numéro de page. Nous modifions donc légèrement notre vue pour le paramètre `page` :

```python
def liste(request, page=1):
   """ Affichage des redirections """
   minis = MiniURL.objects.order_by('-nb_acces')

   return render(request, 'mini_url/liste.html', locals())
```

Et notre fichier `urls.py` :

```python
urlpatterns = patterns('mini_url.views',
    url(r'^$', 'liste', name='url_liste'),   # Pas d'argument page précisé -> vaudra 1 par défaut
    url(r'^(?P<page>\d+)$', 'liste', name='url_liste'),
    # …
```

Nous créons donc un objet `Paginator` à partir de cette liste, comme nous avons pu le faire au début de ce chapitre. Nous avons également vu que `Paginator` permettait de récupérer les objets d'une page précise : c'est ce que nous utiliserons désormais pour renvoyer au template la liste d'URL à afficher.

```python
from django.core.paginator import Paginator, EmptyPage

def liste(request, page=1):
    """ Affichage des redirections enregistrées """
    minis_list = MiniURL.objects.order_by('-nb_acces')
    paginator = Paginator(minis_list, 5)  # 5 liens par page

    try:
        # La définition de nos URL autorise comme argument « page » uniquement 
        # des entiers, nous n'avons pas à nous soucier de PageNotAnInteger
        minis = paginator.page(page)
    except EmptyPage:
        # Nous vérifions toutefois que nous ne dépassons pas la limite de page
        # Par convention, nous renvoyons la dernière page dans ce cas
        minis = paginator.page(paginator.num_pages)

    return render(request, 'mini_url/liste.html', locals())
```

En ajoutant 5 lignes de code (sans prendre en compte les commentaires), nous disposons désormais d'une pagination robuste, gérant tous les cas limites. Si vous testez la vue actuellement, vous verrez que l'adresse [`http://127.0.0.1:8000/m/`](`http://127.0.0.1:8000/m/`) renvoie les 5 premières URL (ajoutez-en si vous en avez moins), [`http://127.0.0.1:8000/m/2`](`http://127.0.0.1:8000/m/2`) les 5 suivantes, etc.

Passons désormais au template. En effet, pour l'instant il est impossible de passer d'une page à l'autre sans jouer avec l'URL et il est impossible de savoir le nombre de pages qu'il y a. Pour renseigner toutes ces informations, nous allons utiliser les informations que nous avons vues précédemment :

```jinja
<h1>Le raccourcisseur d'URL spécial crêpes bretonnes !</h1>

<p><a href="{% url 'url_nouveau' %}">Raccourcir une URL.</a></p>

<p>Liste des URL raccourcies :</p>
<ul>
    {% for mini in minis %}
    <li> <a href="{% url 'url_update' mini.code %}">Mettre à jour</a> -  <a href="{% url 'url_delete' mini.code %}">Supprimer</a>
    | {{ mini.url }} via <a href="http://{{ request.get_host }}{% url 'url_redirection' mini.code %}">
                            {{ request.get_host }}{% url 'url_redirection' mini.code %}
                         </a> {% if mini.pseudo %}par {{ mini.pseudo }}{% endif %} 
    ({{ mini.nb_acces }} accès)</li>
    {% empty %}
    <li>Il n'y en a pas actuellement.</li>
    {% endfor %}
</ul>

<div class="pagination">
   {% if minis.has_previous %}
       <a href="{% url 'url_liste' minis.previous_page_number %}">Précédente</a> -
   {% endif %}

   <span class="current">
       Page {{ minis.number }} sur {{ minis.paginator.num_pages }}
   </span>

   {% if minis.has_next %}
       - <a href="{% url 'url_liste' minis.next_page_number %}">Suivante</a>
   {% endif %}
</div>
```

Nous utilisons ici les méthodes `has_next` et `has_previous` pour savoir s'il faut afficher les liens « Précédent » et « Suivant ». Nous profitons également de l'attribut `num_pages` de `Paginator` afin d'afficher le total de pages.

Un bon conseil que nous pouvons vous donner, et en même temps un bon exercice à faire, est de créer un template générique gérant la pagination et de l'appeler où vous en avez besoin, via `{% include "pagination.html" with liste=minis view="url_liste" %}`.

Vous pouvez maintenant adapter la pagination comme vous voulez en modifiant uniquement la ligne appelant `Paginator` ! Par exemple, on peut réutiliser le troisième argument optionnel `orphans` pour avoir un minimum de liens sur la dernière page : 

```python
paginator = Paginator(minis_list, 20, 5)  # 20 liens par page, avec un minimum de 5 liens sur la dernière
```

Nous en avons fini avec la pagination. Ce module est l'exemple le plus frappant de ce que nous pouvons faire avec Django en seulement quelques lignes, tout en changeant très peu de code par rapport à la base de départ.


En résumé
---------

 - La classe `django.core.paginator.Paginator` permet de générer la pagination de plusieurs types de listes d'objets et s'instancie avec au minimum une liste et le nombre d'éléments à afficher par page.
 - Les attributs et méthodes clés de `Paginator` à retenir sont `p.num_pages` et `p.page()`. La classe `Page` a notamment les méthodes `has_next()`, `has_previous()` et est itérable afin de récupérer les objets de la page courante.
 - Il est possible de rendre la pagination plus pratique en prenant en compte l'argument `orphans` de `Paginator`.
 - Pensez à uniformiser vos paginations en terme d'affichage au sein de votre site web, pour ne pas perturber vos visiteurs, via un template générique.
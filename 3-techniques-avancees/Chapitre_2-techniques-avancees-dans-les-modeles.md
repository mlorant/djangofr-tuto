Techniques avancées dans les modèles
====================================

Dans la partie précédente, nous avons vu comment créer, lier des modèles et faire des requêtes sur ceux-ci. Cependant, les modèles ne se résument pas qu'à des opérations basiques, Django propose en effet des techniques plus avancées qui peuvent se révéler très utiles dans certaines situations. Ce sont ces techniques que nous aborderons dans ce chapitre.

Les requêtes complexes avec Q
-----------------------------

Django propose un outil très puissant et utile, nommé Q, pour créer des requêtes complexes sur des modèles. Il se peut que vous vous soyez demandé lors de l'introduction aux requêtes comment formuler des requêtes avec la clause « OU » (OR en anglais ; par exemple, la catégorie de l'article que je recherche doit être « Crêpes OU Bretagne »). Eh bien, c'est ici qu'intervient l'objet Q ! Il permet aussi de créer des requêtes de manière plus dynamique.

Avant tout, prenons un modèle simple pour illustrer nos exemples :

    class Eleve(models.Model):
        nom = models.CharField(max_length=31)
        moyenne = models.IntegerField(default=10)

        def __str__(self):
            return "Élève {0} ({1}/20 de moyenne)".format(self.nom, self.moyenne)

Ajoutons quelques élèves dans la console interactive (pour rappel : `manage.py shell`) :

    >>> from test.models import Eleve
    >>> Eleve(nom="Mathieu",moyenne=18).save()
    >>> Eleve(nom="Maxime",moyenne=7).save()  # Le vilain petit canard !
    >>> Eleve(nom="Thibault",moyenne=10).save()
    >>> Eleve(nom="Sofiane",moyenne=10).save()

Pour créer une requête dynamique, rien de plus simple, nous pouvons formuler une condition avec un objet Q ainsi :

    >>> from django.db.models import Q
    >>> Q(nom="Maxime")
    <django.db.models.query_utils.Q object at 0x222f650>  # Nous voyons bien que nous possédons ici un objet de la classe Q
    >>> Eleve.objects.filter(Q(nom="Maxime"))
    [<Eleve: Élève Maxime (7/20 de moyenne)>]
    >>> Eleve.objects.filter(nom="Maxime")
    [<Eleve: Élève Maxime (7/20 de moyenne)>]

En réalité, les deux dernières requêtes sont équivalentes. On pourrait dès lors se demander quelle est l'utilité de Q. Il s'agit pourtant d'un puissant outil, il est en effet possible de construire une clause « OU » à partir de Q :

    Eleve.objects.filter(Q(moyenne__gt=16) | Q(moyenne__lt=8)) # Nous prenons les moyennes strictement au-dessus de 16 ou en dessous de 8
    [<Eleve: Élève Mathieu (18/20 de moyenne)>, <Eleve: Élève Maxime (7/20 de moyenne)>]

L'opérateur | est généralement connu comme l'opérateur de disjonction (« OU ») dans l'algèbre de Boole, il est repris ici par Django pour désigner cette fois l'opérateur « OR » du langage SQL.

Sachez qu'il est également possible d'utiliser l'opérateur & pour signifier « ET » :

    >>> Eleve.objects.filter(Q(moyenne=10) & Q(nom="Sofiane"))
    [<Eleve: Élève Sofiane (10/20 de moyenne)>]

Néanmoins, cet opérateur n'est pas indispensable, car il suffit de séparer les objets Q avec une virgule, le résultat est identique :

    >>> Eleve.objects.filter(Q(moyenne=10),Q(nom="Sofiane"))
    [<Eleve: Élève Sofiane (10/20 de moyenne)>]

Il est aussi possible de prendre la négation d'une condition. Autrement dit, demander la condition inverse (« NOT » en SQL). Cela se fait en faisant précéder un objet Q dans une requête par le caractère ~.

    >>> Eleve.objects.filter(Q(moyenne=10),~Q(nom="Sofiane"))
    [<Eleve: Élève Thibault (10/20 de moyenne)>]

Pour aller plus loin, construisons quelques requêtes dynamiquement !

Tout d'abord, il faut savoir qu'un objet Q peut se construire de la façon suivante : `Q(('moyenne',10))`, ce qui est identique à `Q(moyenne=10)`.

Quel intérêt ? Imaginons que nous devions obtenir les objets qui remplissent une des conditions dans la liste suivante :

    conditions = [ ('moyenne',15), ('nom','Thibault'), ('moyenne',18) ]

Nous pouvons construire plusieurs objets Q de la manière suivante :

    objets_q = [Q(x) for x in conditions]

et les incorporer dans une requête ainsi (avec une clause « OU ») :

    >>> import operator
    >>> Eleve.objects.filter(reduce(operator.or_, objets_q))
    [<Eleve: Élève Mathieu (18/20 de moyenne)>, <Eleve: Élève Thibault (15/20 de moyenne)>]

`reduce` est une fonction par défaut de Python qui permet d'appliquer une fonction à plusieurs valeurs successivement. Petit exemple pour comprendre plus facilement :

`reduce(lambda x, y: x+y, [1, 2, 3, 4, 5])` va calculer ((((1+2)+3)+4)+5), donc 15. La même chose sera faite ici, mais avec l'opérateur « OU » qui est accessible depuis `operator.or_`. En réalité, Python va donc faire :

    Eleve.objects.filter(objets_q[0] | objets_q[1] | objets_q[2])

C'est une méthode très puissante et très pratique !

L'agrégation
------------

Il est souvent utile d'extraire une information spécifique à travers plusieurs entrées d'un seul et même modèle. Si nous reprenons nos élèves de la sous-partie précédente, leur professeur aura plus que probablement un jour besoin de calculer la moyenne globale des élèves. Pour ce faire, Django fournit plusieurs outils qui permettent de tels calculs très simplement. Il s'agit de la méthode d'agrégation.

En effet, si nous voulons obtenir la moyenne des moyennes de nos élèves (pour rappel, Mathieu (moyenne de 18), Maxime (7), Thibault (10) et Sofiane(10)), nous pouvons procéder à partir de la méthode aggregate :

    from django.db.models import Avg
    >>> Eleve.objects.aggregate(Avg('moyenne'))
    {'moyenne__avg': 11.25}

En effet, (18+7+10+10)/4 = 11,25 !
Cette méthode prend à chaque fois une fonction spécifique fournie par Django, comme `Avg` (pour `Average`, signifiant « moyenne » en anglais) et s'applique sur un champ du modèle. Cette fonction va ensuite parcourir toutes les entrées du modèle et effectuer les calculs propres à celle-ci.

Notons que la valeur retournée par la méthode est un dictionnaire, avec à chaque fois une clé générée automatiquement à partir du nom de la colonne utilisée et de la fonction appliquée (nous avons utilisé la fonction `Avg` dans la colonne `'moyenne'`, Django renvoie donc `'moyenne__avg'`), avec la valeur calculée correspondante (ici 11,25 donc).

Il existe d'autres fonctions comme `Avg`, également issues de `django.db.models`, dont notamment :

- Max : prend la plus grande valeur ;
- Min : prend la plus petite valeur ;
- Count : compte le nombre d'entrées.

Il est même possible d'utiliser plusieurs de ces fonctions en même temps :

    >>> Eleve.objects.aggregate(Avg('moyenne'), Min('moyenne'), Max('moyenne'))
    {'moyenne__max': 18, 'moyenne__avg': 11.25, 'moyenne__min': 7}

Si vous souhaitez préciser une clé spécifique, il suffit de la faire précéder de la fonction :

    >>> Eleve.objects.aggregate(Moyenne=Avg('moyenne'), Minimum=Min('moyenne'), Maximum=Max('moyenne'))
    {'Minimum': 7, 'Moyenne': 11.25, 'Maximum': 18}

Bien évidemment, il est également possible d'appliquer une agrégation sur un `QuerySet` obtenu par la méthode filter par exemple :

    >>> Eleve.objects.filter(nom__startswith="Ma").aggregate(Avg('moyenne'), Count('moyenne'))
    {'moyenne__count': 2, 'moyenne__avg': 12.5}

Étant donné qu'il n'y a que Mathieu et Maxime comme prénoms qui commencent par « Ma », uniquement ceux-ci seront sélectionnés, comme l'indique `moyenne__count`.

En réalité, la fonction `Count` est assez inutile ici, d'autant plus qu'une méthode pour obtenir le nombre d'entrées dans un `QuerySet` existe déjà :

    >>> Eleve.objects.filter(nom__startswith="Ma").count()
    2

Cependant, cette fonction peut se révéler bien plus intéressante lorsque nous l'utilisons avec des liaisons entre modèles.
Pour ce faire, ajoutons un autre modèle :

    class Cours(models.Model):
        nom = models.CharField(max_length=31)
        eleves = models.ManyToManyField(Eleve)

        def __str__(self):
            return self.nom

Créons deux cours :

    >>> c1 = Cours(nom="Maths")
    >>> c1.save()
    >>> c1.eleves.add(*Eleve.objects.all())
    >>> c2 = Cours(nom="Anglais")
    >>> c2.save()
    >>> c2.eleves.add(*Eleve.objects.filter(nom__startswith="Ma"))

Il est tout à fait possible d'utiliser les agrégations depuis des liaisons comme une `ForeignKey`, ou comme ici avec un `ManyToManyField` :

    >>> Cours.objects.aggregate(Max("eleves__moyenne"))
    {'eleves__moyenne__max': 18}

Nous avons été chercher la meilleure moyenne parmi les élèves de tous les cours enregistrés.

Il est également possible de compter le nombre d'affiliations à des cours :

    >>> Cours.objects.aggregate(Count("eleves"))
    {'eleves__count': 6}

En effet, nous avons 6 « élèves », à savoir 4+2, car Django ne vérifie pas si un élève est déjà dans un autre cours ou non.

Pour terminer, abordons une dernière fonctionnalité utile. Il est possible d'ajouter des attributs à un objet selon les objets auxquels il est lié. Nous parlons d'annotation. Exemple :

    >>> Cours.objects.annotate(Avg("eleves__moyenne"))[0].eleves__moyenne__avg
    11.25

Un nouvel attribut a été créé. Au lieu d'être retournées dans un dictionnaire, les valeurs sont désormais directement ajoutées à l'objet lui-même. Il est bien évidemment possible de redéfinir le nom de l'attribut comme vu précédemment :

    >>> Cours.objects.annotate(Moyenne=Avg("eleves__moyenne"))[1].Moyenne
    12.5

Et pour terminer en beauté, il est même possible d'utiliser l'attribut créé dans des méthodes du QuerySet comme `filter`, `exclude` ou `order_by` ! Par exemple :

    >>> Cours.objects.annotate(Moyenne=Avg("eleves__moyenne")).filter(Moyenne__gte=12)
    [<Cours: Anglais>]

En définitive, l'agrégation et l'annotation sont des outils réellement puissants qu'il ne faut pas hésiter à utiliser si l'occasion se présente !

L'héritage de modèles
---------------------

Les modèles étant des classes, ils possèdent les mêmes propriétés que n'importe quelle classe, y compris l'héritage de classes. Néanmoins, Django propose trois méthodes principales pour gérer l'héritage de modèles, qui interagiront différemment avec la base de données. Nous les aborderons ici une à une.

### Les modèles parents abstraits

Les modèles parents abstraits sont utiles lorsque vous souhaitez utiliser plusieurs méthodes et attributs dans différents modèles, sans devoir les réécrire à chaque fois. Tout modèle héritant d'un modèle abstrait récupère automatiquement toutes les caractéristiques de la classe dont elle hérite. La grande particularité d'un modèle abstrait réside dans le fait que Django ne l'utilisera pas comme représentation pour créer une table dans la base de données. En revanche, tous les modèles qui hériteront de ce parent abstrait auront bel et bien une table qui leur sera dédiée.

Afin de rendre un modèle abstrait, il suffit de lui assigner l'attribut `abstract=True` dans sa sous-classe `Meta`. Django se charge entièrement du reste.

Pour illustrer cette méthode, prenons un exemple simple :

    class Document(models.Model):
        titre = models.CharField(max_length=255)
        date_ajout = models.DateTimeField(auto_now_add=True, verbose_name="Date d'ajout du document")
        auteur = models.CharField(max_length=255, null=True, blank=True)

        class Meta:
            abstract = True
    
    class Article(Document):
        contenu = models.TextField()
    
    class Image(Document):
        image = models.ImageField(upload_to="images")

Ici, deux tables seront créées dans la base de données : `Article` et `Image`. Le modèle `Document` ne sera pas utilisé comme table, étant donné que celui-ci est abstrait. En revanche, les tables `Article` et `Image` auront bien les champs de `Document` (donc par exemple la table `Article` aura les champs titre, `date_ajout`, `auteur` et `contenu`).

Bien entendu, il est impossible de faire des requêtes sur un modèle abstrait, celui-ci n'ayant aucune table dans la base de données pour enregistrer des données. Vous ne pouvez interagir avec les champs du modèle abstrait que depuis les modèles qui en héritent.

### Les modèles parents classiques

Contrairement aux modèles abstraits, il est possible d'hériter de modèles tout à fait normaux. Si un modèle hérite d'un autre modèle non abstrait, il n'y aura aucune différence pour ce dernier, il sera manipulable comme n'importe quel modèle. Django créera une table pour le modèle parent et le modèle enfant.

Prenons un exemple simple :

    class Lieu(models.Model):
        nom = models.CharField(max_length=50)
        adresse = models.CharField(max_length=100)

        def __str__(self):
            return self.nom
    
    class Restaurant(Lieu):
        menu = models.TextField()

À partir de ces deux modèles, Django créera bien deux tables, une pour `Lieu`, l'autre pour `Restaurant`. Il est important de noter que la table `Restaurant` ne contient pas les champs de `Lieu` (à savoir `nom` et `adresse`). En revanche, elle contient bien le champ menu et une clé étrangère vers `Lieu` que le framework ajoutera tout seul.
En effet, si `Lieu` est un modèle tout à fait classique, `Restaurant` agira un peu différemment.

Lorsque nous sauvegardons une nouvelle instance de `Restaurant` dans la base de données, une nouvelle entrée sera créée dans la table correspondant au modèle `Restaurant`, mais également dans celle correspondant à `Lieu`. Les valeurs des deux attributs nom et adresse seront enregistrées dans une entrée de la table `Lieu`, et l'attribut menu sera enregistré dans une entrée de la table `Restaurant`. Cette dernière entrée contiendra donc également la clé étrangère vers l'entrée dans la table `Lieu` qui possède les données associées.

Pour résumer, l'héritage classique s'apparente à la liaison de deux classes avec une clé étrangère telle que nous en avons vu dans le chapitre introductif sur les modèles, à la différence que c'est Django qui se charge de réaliser lui-même cette liaison.

Sachez aussi que lorsque vous créez un objet `Restaurant`, vous créez aussi un objet `Lieu` tout à fait banal qui peut être obtenu comme n'importe quel objet `Lieu` créé précédemment. De plus, même si les attributs du modèle parent sont dans une autre table, le modèle fils a bien hérité de toutes ses méthodes et attributs :

    >>> Restaurant(nom=u"La crêperie bretonne",adresse="42 Rue de la crêpe 35000 Rennes", menu=u"Des crêpes !").save()
    >>> Restaurant.objects.all()
    [<Restaurant: La crêperie bretonne>]
    >>> Lieu.objects.all()
    [<Lieu: La crêperie bretonne>]

Pour finir, tous les attributs de `Lieu` sont directement accessibles depuis un objet `Restaurant` :

    >>> resto = Restaurant.objects.all()[0]
    >>> print resto.nom+", "+resto.menu
    La crêperie bretonne, Des crêpes !

En revanche, il n'est pas possible d'accéder aux attributs spécifiques de `Restaurant` depuis une instance de `Lieu` :

    >>> lieu = Lieu.objects.all()[0]
    >>> print lieu.nom
    La crêperie bretonne
    >>> print lieu.menu #Ça ne marche pas
    Traceback (most recent call last):
      File "<console>", line 1, in <module>
    AttributeError: 'Lieu' object has no attribute 'menu'

Pour accéder à l'instance de `Restaurant` associée à `Lieu`, Django crée tout seul une relation vers celle-ci qu'il nommera selon le nom de la classe fille :

    >>> print type(lieu.restaurant)
    <class 'blog.models.Restaurant'>
    >>> print lieu.restaurant.menu
    Des crêpes !

### Les modèles proxy

Dernière technique d'héritage avec Django, et probablement la plus complexe, il s'agit de modèles *proxy* (en français, des modèles « passerelles »).

Le principe est trivial : un modèle proxy hérite de tous les attributs et méthodes du modèle parent, mais aucune table ne sera créée dans la base de données pour le modèle fils. En effet, le modèle fils sera en quelque sorte une passerelle vers le modèle parent (tout objet créé avec le modèle parent sera accessible depuis le modèle fils, et vice-versa).

Quel intérêt ? À première vue, il n'y en a pas, mais pour quelque raison de structure du code, d'organisation, etc., nous pouvons ajouter des méthodes dans le modèle proxy, ou modifier des attributs de la sous-classe `Meta` sans que le modèle d'origine ne soit altéré, et continuer à utiliser les mêmes données.

Petit exemple de modèle proxy qui hérite du modèle `Restaurant` que nous avons défini tout à l'heure (notons qu'il est possible d'hériter d'un modèle qui hérite lui-même d'un autre !) :

    class RestoProxy(Restaurant):
        class Meta:
            proxy = True # Nous spécifions qu'il s'agit d'un proxy
            ordering = ["nom"] # Nous changeons le tri par défaut, tous les QuerySet seront triés selon le nom de chaque objet

        def crepes(self):
            if "crêpe" in self.menu: #Il y a des crêpes dans le menu
                return True
            return False

Depuis ce modèle, il est donc possible d'accéder aux données enregistrées du modèle parent, tout en bénéficiant des méthodes et attributs supplémentaires :

    >>> from blog.models import RestoProxy
    >>> print RestoProxy.objects.all()
    [<RestoProxy: La crêperie bretonne>]
    >>> resto = RestoProxy.objects.all()[0]
    >>> print resto.adresse
    42 Rue de la crêpe 35000 Rennes
    >>> print resto.crepes()
    True

L'application ContentType
-------------------------

Il est possible qu'un jour vous soyez amenés à devoir jouer avec des modèles. Non pas avec des instances de modèles, mais des modèles mêmes.

En effet, jusqu'ici, nous avons pu lier des entrées à d'autres entrées (d'un autre type de modèle éventuellement) avec des liaisons du type `ForeignKey`, `OneToOneField` ou `ManyToManyField`. Néanmoins, Django propose un autre système qui peut se révéler parfois utile : la liaison d'une entrée de modèle à un autre modèle en lui-même. Si l'intérêt d'une telle relation n'est pas évident à première vue, nous pouvons pourtant vous assurer qu'il existe, et nous vous l'expliquerons par la suite. Ce genre de relation est possible grâce à ce que nous appelons des `ContentTypes`.

Avant tout, il faut s'assurer que l'application `ContentTypes` de Django est bien installée. Elle l'est par défaut, néanmoins, si vous avez supprimé certaines entrées de votre variable `INSTALLED_APPS` dans le fichier `settings.py`, il est toujours utile de vérifier si le tuple contient bien l'entrée `'django.contrib.contenttypes'`. Si ce n'est pas le cas, ajoutez-la.

Un `ContentType` est en réalité un modèle assez spécial. Ce modèle permet de représenter un autre modèle installé. Par exemple, nous avons déclaré plus haut le modèle `Eleve`. Voici sa représentation depuis un `ContentType` :

    >>> from blog.models import Eleve
    >>> from django.contrib.contenttypes.models import ContentType
    >>> ct = ContentType.objects.get(app_label="blog", model="eleve")
    >>> ct
    <ContentType: eleve>

Désormais, notre variable `ct` représente le modèle `Eleve`. Que pouvons-nous en faire ? Le modèle `ContentType` possède deux méthodes :

- `model_class` : renvoie la classe du modèle représenté ;
- `get_object_for_this_type` : il s'agit d'un raccourci vers la méthode `objects.get` du modèle. Cela évite de faire `ct.model_class().objects.get(attr=arg)`.

Illustration :

    >>> ct.model_class()
    <class 'blog.models.Eleve'>
    >>> ct.get_object_for_this_type(nom="Maxime")
    <Eleve: Élève Maxime (7/20 de moyenne)>

Maintenant que le fonctionnement des `ContentType` vous semble plus familier, passons à leur utilité. À quoi servent-ils ? Imaginons que vous deviez implémenter un système de commentaires, mais que ces commentaires ne se restreignent pas à un seul modèle. En effet, vous souhaitez que vos utilisateurs puissent commenter vos articles, vos images, vos vidéos, ou même des commentaires eux-mêmes.

Une première solution serait de faire hériter tous vos modèles d'un modèle parent nommée `Document` (un peu comme démontré ci-dessus) et de créer un modèle `Commentaire` avec une `ForeignKey` vers `Document`. Cependant, vous avez déjà écrit vos modèles `Article`, `Image`, `Video`, et ceux-ci n'héritent pas d'une classe commune comme `Document`. Vous n'avez pas envie de devoir réécrire tous vos modèles, votre code, adapter votre base de données, etc. La solution magique ? Les relations génériques des `ContentTypes`.

Une relation générique d'un modèle est une relation permettant de lier une entrée d'un modèle défini à une autre entrée de n'importe quel modèle. Autrement dit, si notre modèle `Commentaire` possède une relation générique, nous pourrions le lier à un modèle `Image`, `Video`, `Commentaire`… ou n'importe quel autre modèle installé, sans la moindre restriction.

Voici une ébauche de ce modèle `Commentaire` avec une relation générique :

    from django.contrib.contenttypes.models import ContentType
    from django.contrib.contenttypes.fieds import GenericForeignKey
    
    class Commentaire(models.Model):
        auteur = models.CharField(max_length=255)
        contenu = models.TextField()
        content_type = models.ForeignKey(ContentType)
        object_id = models.PositiveIntegerField()
        content_object = GenericForeignKey('content_type', 'object_id')

        def __str__(self):
            return "Commentaire de {0} sur {1}".format(self.auteur, self.content_object)

La relation générique est ici l'attribut `content_object`, avec le champ `GenericForeignKey`. Cependant, vous avez sûrement remarqué l'existence de deux autres champs : `content_type` et `object_id`. À quoi servent-ils ?

Auparavant, avec une `ForeignKey` classique, tout ce qu'il fallait indiquer était, dans la déclaration du modèle, la classe à laquelle la clé serait liée, et lors de la déclaration de l'instance, l'entrée de l'autre modèle à laquelle la clé serait liée, cette dernière contenant alors l'ID de l'entrée associée.

Le fonctionnement est similaire ici, à une exception près : le modèle associé n'est pas défini lors de la déclaration de la classe (vu que ce modèle peut être n'importe lequel). Dès lors, il nous faut un champ supplémentaire pour représenter ce modèle, et ceci se fait grâce à un `ContentType` (que nous avons expliqué plus haut). Si nous avons le modèle, il ne nous manque plus que l'ID de l'entrée pour pouvoir récupérer la bonne entrée. C'est ici qu'intervient le champ `object_id`, qui est juste un entier positif, contenant donc l'ID de l'entrée. Dès lors, le champ `content_object`, la relation générique, est en fait une sorte de produit des deux autres champs. Il va aller chercher le type de modèle associé dans `content_type`, et l'ID de l'entrée associée dans `object_id`. Finalement, la relation générique va aller dans la table liée au modèle obtenu, chercher l'entrée ayant l'ID obtenu, et renvoyer la bonne entrée. C'est pour cela qu'il faut indiquer à la relation générique lors de son initialisation les deux attributs depuis lesquels il va pouvoir aller chercher le modèle et l'ID.

Maintenant que la théorie est explicitée, prenons un petit exemple pratique :

    >>> from blog.models import Commentaire, Eleve
    >>> e = Eleve.objects.get(nom="Sofiane")
    >>> c = Commentaire.objects.create(auteur="Le professeur",contenu="Sofiane ne travaille pas assez.", content_object=e)
    >>> c.content_object
    <Eleve: Élève Sofiane (10/20 de moyenne)>
    >>> c.object_id
    4
    >>> c.content_type.model_class()
    <class 'blog.models.Eleve'>

Lors de la création d'un commentaire, il n'y a pas besoin de remplir les champs `object_id` et `content_type`. Ceux-ci seront automatiquement déduits de la variable donnée à `content_object`.
Bien évidemment, vous pouvez adresser n'importe quelle entrée de n'importe quel modèle à l'attribut `content_object`, cela marchera !

Avant de terminer, sachez qu'il est également possible d'ajouter une relation générique « en sens inverse ». Contrairement à une `ForeignKey` classique, aucune relation inverse n'est créée. Si vous souhaitez tout de même en créer une sur un modèle bien précis, il suffit d'ajouter un champ nommé `GenericRelation`.

Si nous reprenons le modèle `Eleve`, cette fois modifié :

    from django.contrib.contenttypes.fields import GenericRelation

    class Eleve(models.Model):
        nom = models.CharField(max_length=31)
        moyenne = models.IntegerField(default=10)
        commentaires = GenericRelation('Commentaire')

        def __str__(self):
            return "Élève {0} ({1}/20 de moyenne)".format(self.nom, self.moyenne)

Dès lors, le champ commentaires contient tous les commentaires adressés à l'élève :

    >>> e.commentaires.all()
    [u"Commentaire de Le professeur sur Élève Sofiane (10/20 de moyenne)"]

Sachez que si vous avez utilisé des noms différents que `content_type` et `object_id` pour construire votre `GenericForeignKey`, vous devez également le spécifier lors de la création de la `GenericRelation` :

    commentaires = GenericRelation(Commentaire,
                                          content_type_field="le_champ_du_content_type",
                                          object_id_field="le champ_de_l_id")

En résumé
---------

- La classe Q permet d'effectuer des requêtes complexes avec les opérateurs « OU », « ET » et « NOT ».
- `Avg`, `Max` et `Min` permettent d'obtenir respectivement la moyenne, le maximum et le minimum d'une certaine colonne dans une table. Elles peuvent être combinées avec Count pour déterminer le nombre de lignes retournées.
- L'héritage de modèles permet de factoriser des modèles ayant des liens entre eux. Il existe plusieurs types d'héritage : abstrait, classique et les modèles proxy.
- L'application `ContentType` permet de décrire un modèle et de faire des relations génériques avec vos autres modèles (pensez à l'exemple des commentaires !).



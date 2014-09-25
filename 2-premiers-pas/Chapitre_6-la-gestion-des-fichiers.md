La gestion des fichiers
=======================

Autre point essentiel du web actuel : il est souvent utile d'envoyer des fichiers sur un site web afin que celui-ci puisse les réutiliser par la suite (avatar d'un membre, album photos, chanson…). Nous couvrirons dans cette partie la gestion des fichiers côté serveur et les méthodes proposées par Django.

Enregistrer une image
---------------------

Préambule : avant de commencer à jouer avec des images, il est nécessaire d'installer la bibliothèque Pillow. Django se sert en effet de cette dernière pour faire ses traitements sur les images. Vous pouvez télécharger la bibliothèque en utilisant `pip install pillow` (conseillé), ou depuis [cette adresse](https://pypi.python.org/pypi/Pillow/2.4.0).

Pour introduire la gestion des images, prenons un exemple simple : considérons un répertoire de contacts dans lequel les contacts ont trois caractéristiques : leur nom, leur adresse et une photo. Pour ce faire, créons un nouveau modèle (placez-le dans l'application de votre choix, personnellement nous réutiliserons ici l'application « blog ») :

    class Contact(models.Model):
        nom = models.CharField(max_length=255)
        adresse = models.TextField()
        photo = models.ImageField(upload_to="photos/")

        def __str__(self):
               return self.nom

La nouveauté ici est bien entendu `ImageField`. Il s'agit d'un champ Django comme les autres, si ce n'est qu'il contiendra une image (au lieu d'une chaîne de caractères, une date, un nombre…).

`ImageField` prend un argument obligatoire : `upload_to`. Ce paramètre permet de désigner l'endroit où seront enregistrées sur le disque dur les images assignées à l'attribut photo pour toutes les instances du modèle. Nous n'avons pas indiqué d'adresse absolue ici, car en réalité le répertoire indiqué depuis le paramètre sera ajouté au chemin absolu fourni par la variable `MEDIA_ROOT` dans votre `settings.py`. Il est impératif de configurer correctement cette variable avant de commencer à jouer avec des fichiers, par exemple : `MEDIA_ROOT = '/home/mathx/crepes_bretonnes/media/'`.

Afin d'avoir une vue permettant de créer un nouveau contact, il faudra créer un formulaire adapté. Créons un formulaire similaire au modèle (un `ModelForm` est tout à fait possible aussi), tout ce qu'il y a de plus simple :

    class NouveauContactForm(forms.Form):
        nom = forms.CharField()
        adresse = forms.CharField(widget=forms.Textarea)
        photo = forms.ImageField()

Le champ `ImageField` vérifie que le fichier envoyé est bien une image valide, sans quoi le formulaire sera considéré comme invalide. Et le tour est joué !

Revenons-en donc à la vue. Elle est également similaire à un traitement de formulaire classique, à un petit détail près :

    def nouveau_contact(request):
        sauvegarde = False
        if request.method == "POST":
               form = NouveauContactForm(request.POST, request.FILES)
               if form.is_valid():
                       contact = Contact()
                       contact.nom = form.cleaned_data["nom"]
                       contact.adresse = form.cleaned_data["adresse"]
                       contact.photo = form.cleaned_data["photo"]
                       contact.save()
                       sauvegarde = True
        else:
               form = NouveauContactForm()
        return render(request, 'contact.html',locals())

Faites bien attention à la ligne 5 : un deuxième argument a été ajouté, il s'agit de `request.FILES`. En effet, `request.POST` ne contient que des données textuelles, tous les fichiers sélectionnés sont envoyés depuis une autre méthode, et sont finalement recueillis par Django dans le dictionnaire `request.FILES`. Si vous ne passez pas cette variable au constructeur, celui-ci considérera que le champ photo est vide et n'a donc pas été complété par l'utilisateur, le formulaire sera donc invalide.

Le champ `ImageField` renvoie une variable du type `UploadedFile`, qui est une classe définie par Django. Cette dernière hérite de la classe `django.core.files.File`. Sachez que si vous souhaitez créer une entrée en utilisant une photo sur votre disque dur (autrement dit, vous ne disposez pas d'une variable `UploadedFile` renvoyée par le formulaire), vous devez créer un objet `File` (prenant un fichier ouvert classiquement) et le passer à votre modèle. Exemple depuis la console :

    >>> from blog.models import Contact
    >>> from django.core.files import File
    >>> c = Contact(nom="Jean Dupont",adresse="Rue Neuve 34, Paris")
    >>> photo = File(open('/chemin/vers/photo/dupont.jpg','r'))
    >>> c.photo = photo
    >>> c.save()

Pour terminer, le template est également habituel, toujours à une exception près :

    <h1>Ajouter un nouveau contact</h1>
    {% if sauvegarde %}
        <p>Ce contact a bien été enregistré.</p>
    {% endif %}
       
    <p>
        <form method="post" enctype="multipart/form-data" action=".">
           {% csrf_token %}
           {{ form.as_p }}
           <input type="submit"/>
        </form>
    </p>

Faites bien attention au nouvel attribut de la balise form : `enctype="multipart/form-data"`. En effet, sans ce dernier, le navigateur n'enverra pas les fichiers au serveur web. Oublier cet attribut et le dictionnaire `request.FILES` décrit précédemment sont des erreurs courantes qui peuvent vous faire perdre bêtement beaucoup de temps, ayez le réflexe d'y penser !

Sachez que Django n'acceptera pas n'importe quel fichier. En effet, il s'assurera que le fichier envoyé est bien une image, sans quoi il retournera une erreur.

Vous pouvez essayer le formulaire : vous constaterez qu'un nouveau fichier a été créé dans le dossier renseigné dans la variable `MEDIA_ROOT`. Le nom du fichier créé sera en fait le même que celui sur votre disque dur (autrement dit, si vous avez envoyé un fichier nommé `mon_papa.jpg`, le fichier côté serveur gardera le même nom). Il est possible de modifier ce comportement, nous y reviendrons plus tard.

Afficher une image
------------------

Maintenant que nous possédons une image enregistrée côté serveur, il ne reste plus qu'à l'afficher chez le client. Cependant, un petit problème se pose : par défaut, Django ne s'occupe pas du service de fichiers média (images, musiques, vidéos…), et généralement il est conseillé de laisser un autre serveur s'en occuper (voir l'annexe sur le déploiement du projet en production). Néanmoins, pour la phase de développement, il est tout de même possible de laisser le serveur de développement s'en charger. Pour ce faire, il vous faut compléter la variable `MEDIA_URL` dans `settings.py` et ajouter cette directive dans votre `urls.py` global :

    from django.conf.urls.static import static
    from django.conf import settings
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

Cela étant fait, tous les fichiers consignés dans le répertoire configuré depuis `MEDIA_ROOT` (dans lequel Django déplace les fichiers enregistrés) seront accessibles depuis l'adresse telle qu'indiquée depuis `MEDIA_URL` (un exemple de `MEDIA_URL` serait simplement `"/media/"` ou `"media.monsite.fr/"` en production).

Cela étant fait, l'affichage d'une image est trivial. Si nous reprenons la liste des contacts enregistrés dans une vue simple :

    def voir_contacts(request):
        contacts = Contact.objects.all()
        return render(request, 'voir_contacts.html',{'contacts':contacts})

Côté template :

    <h1>Liste des contacts</h1>
    {% for contact in contacts %}
        <h2>{{ contact.nom }}</h2>
        Adresse : {{ contact.adresse|linebreaks }}<br/>
        <img src="{{ MEDIA_URL }}{{ contact.photo }}"/>
    {% endfor %}

Avant de s'attarder aux spécificités de l'affichage de l'image, une petite explication concernant le tag `linebreaks.` Par défaut, Django ne convertit pas les retours à la ligne d'une chaine de caractères (comme l'adresse ici) en un `<br/>` automatiquement, et cela pour des raisons de sécurité. Pour autoriser l'ajout de retours à la ligne en HTML, il faut utiliser ce tag, comme dans le code ci-dessus, sans quoi toute la chaîne sera sur la même ligne.

Revenons donc à l'adresse de l'image. Vous aurez déjà plus que probablement reconnu la variable `MEDIA_URL` de `settings.py`, qui fait son retour. Elle est accessible depuis le template grâce à un processeur de contexte inclus par défaut.

`contact.photo` renvoie simplement l'adresse relative vers le dossier et le nom du fichier associé. Afin de construire une adresse complète, il est impératif d'associer ces deux parties d'adresse, simplement en les juxtaposant.
Si par exemple `MEDIA_URL` vaut `"media.monsite.fr/"` et `contact.photo` vaut `"photos/mon_papa.jpg"`, l'adresse absolue concaténée sera donc `"media.monsite.fr/photos/mon_papa.jpg"`.

Le résultat est plutôt simple, comme vous pouvez le constater sur la figure suivante.

![L'adresse de Chuck Norris !](images/fichiers_chuck.png "L'adresse de Chuck Norris !")

Il est important de ne jamais renseigner en dur le lien vers l'endroit où est situé le dossier contenant les fichiers. Passer par `MEDIA_URL` est une méthode bien plus propre.

Avant de généraliser pour tous les types de fichiers, sachez qu'un `ImageField` non nul possède deux attributs supplémentaires : `width` et `height`. Ces deux attributs renseignent respectivement la largeur et la hauteur en pixels de l'image.

Encore plus loin
----------------

Heureusement, la gestion des fichiers ne s'arrête pas aux images. N'importe quel type de fichier peut être enregistré. La différence avec les images est plutôt maigre.

Au lieu d'utiliser `ImageField` dans les formulaires et modèles, il suffit tout simplement d'utiliser `FileField`. Que ce soit dans les formulaires ou les modèles, le champ s'assurera que ce qui lui est passé est bien un fichier, mais cela ne devra plus être nécessairement une image valide.

`FileField` retournera toujours un objet de `django.core.files.File`. Cette classe possède notamment les attributs suivants (l'exemple ici est réalisé avec un `ImageField`, mais les attributs sont également valides avec un `FileField` bien évidemment) :

    >>> from blog.models import Contact
    >>> c = Contact.objects.get(nom="Chuck Norris")
    >>> c.photo.name
    'photos/chuck_norris.jpg'  # Chemin relatif vers le fichier à partir de MEDIA_ROOT
    >>> c.photo.path
    '/home/mathx/crepes_bretonnes/media/photos/chuck_norris.jpg'  # Chemin absolu
    >>> c.photo.url
    'http://media.crepes-bretonnes.com/photos/chuck_norris.jpg'  # URL telle que construite à partir de MEDIA_URL
    >>> c.photo.size
    45300  # Taille du fichier en bytes

De plus, un objet `File` possède également des attributs `read` et `write`, comme un fichier (ouvert à partir d'`open()`) classique.

Dernière petite précision concernant le nom des fichiers côté serveur. Nous avons mentionné plus haut qu'il est possible de les renommer à notre guise, et de ne pas garder le nom que l'utilisateur avait sur son disque dur.

La méthode est plutôt simple : au lieu de passer une chaîne de caractères comme paramètre `upload_to` dans le modèle, il faut lui passer une fonction qui retournera le nouveau nom du fichier. Cette fonction prend deux arguments : l'instance du modèle où le `FileField` est défini, et le nom d'origine du fichier.

Un exemple de fonction serait donc simplement :

    def renommage(instance, nom):
       return instance.id+'.'+nom.split('.')[-1]  # Nous nous basons sur l'ID de l'entrée et nous gardons l'extension du fichier (en supposant ici que le fichier possède bien une extension)

Un exemple de modèle utilisant cette fonction serait alors :

    class Document(models.Model):
        nom = models.CharField(max_length=100)
        doc = models.FileField(upload_to=renommage, verbose_name="Document")

Désormais, vous devriez être en mesure de gérer correctement toute application nécessitant des fichiers !

En résumé
---------

- L'installation de la bibliothèque Pillow est nécessaire pour gérer les images dans Django. Cette bibliothèque permet de faire des traitements sur les images (vérification et redimensionnement notamment).
- Le stockage d'une image dans un objet en base se fait via un champ `models.ImageField`. Le stockage d'un fichier quelconque est similaire, avec `models.FileField`.
- Les fichiers uploadés seront stockés dans le répertoire fourni par `MEDIA_ROOT` dans votre `settings.py`.



Les utilisateurs
================

S'il y a bien une application très puissante et couramment utilisée que Django propose, c'est la gestion des utilisateurs. Le framework propose en effet une solution complète pour gérer le cas classique d'un accès membres, très flexible et indéniablement pratique. Nous expliquerons l'essentiel de cette application dans ce chapitre.

Commençons par la base
----------------------

Avant tout, il est nécessaire de s'assurer que vous avez bien ajouté l'application qui gère les utilisateurs, et ses dépendances. Elles sont ajoutées par défaut, néanmoins, vérifiez toujours que `'django.contrib.auth'` et `'django.contrib.contenttypes'` sont bien présents dans la variable `INSTALLED_APPS` de votre `settings.py`. De même, les middlewares suivants sont nécéssaires :
     
 - `'django.contrib.sessions.middleware.SessionMiddleware'` ;
 - `'django.contrib.auth.middleware.AuthenticationMiddleware'`.

### L'utilisateur

Tout le système utilisateur tourne autour du modèle `django.contrib.auth.models.User`. Celui-ci contient toutes les informations concernant vos utilisateurs, et quelques méthodes supplémentaires bien pratiques pour pouvoir les administrer. Voici les principaux attributs de `User` :

 - `username` : nom d'utilisateur, 30 caractères maximum (lettres, chiffres et les caractères spéciaux _, @, +, . et -) ;
 - `first_name` : prénom, optionnel, 30 caractères maximum ;
 - `last_name` : nom de famille, optionnel, 30 caractères maximum ;
 - `email` : adresse e-mail ;
 - `password` : un *hash* du mot de passe. Django n'enregistre pas les mots de passe en clair dans la base de données, nous y reviendrons plus tard ;
 - `is_staff` : booléen, permet d'indiquer si l'utilisateur a accès à l'administration de Django ;
 - `is_active` : booléen, par défaut mis à `True`. Si mis à `False`, l'utilisateur est considéré comme désactivé et ne peut plus se connecter. Au lieu de supprimer un utilisateur, il est conseillé de le désactiver afin de ne pas devoir supprimer d'éventuels modèles liés à l'utilisateur (avec une `ForeignKey` par exemple) ;
 - `is_superuser` : booléen, si mis à `True`, l'utilisateur obtient toutes les permissions (nous y reviendrons plus tard également) ;
 - `last_login` : datetime, représente la date et l'heure à laquelle l'utilisateur s'est connecté la dernière fois ;
 - `date_joined` : datetime, représente la date et l'heure à laquelle l'utilisateur s'est inscrit ;
 - `user_permissions` : une relation `ManyToMany` vers les permissions (introduites par la suite) ;
 - `groups` : une relation `ManyToMany` vers les groupes (introduits par la suite).

Vous ne vous servirez pas nécessairement de tous ces attributs, mais ils peuvent se révéler pratiques si l'occasion de les utiliser se présente. La première question qui devrait vous venir à l'esprit est la suivante : « Est-il possible d'ajouter des attributs ? La liste est plutôt limitée. » N'ayez crainte, nous aborderons cela bientôt.

La façon la plus simple de créer un utilisateur est d'utiliser la fonction `create_user` fournie avec le modèle. Elle prend trois arguments : le nom de l'utilisateur, son adresse e-mail et son mot de passe (les trois attributs obligatoires du modèle), et enregistre directement l'utilisateur dans la base de données :

```python
>>> from django.contrib.auth.models import User
>>> user = User.objects.create_user('Maxime', 'maxime@crepes-bretonnes.com', 'm0nsup3rm0td3p4ss3')
>>> user.id
2
```

Nous avons donc ici un nouvel utilisateur nommé « Maxime », avec l'adresse e-mail `maxime@crepes-bretonnes.com` et comme mot de passe « m0nsup3rm0td3p4ss3 ». Son ID dans la base de données (preuve que l'entrée a bien été sauvegardée) est 2. Bien entendu, nous pouvons désormais modifier les autres champs :

```python
>>> user.first_name, user.last_name = "Maxime","Lorant"
>>> user.is_staff = True
>>> user.save()
```

Et les modifications sont enregistrées. Tous les champs sont éditables classiquement, sauf un : `password`, qui possède ses propres fonctions. 

### Les mots de passe

En effet, les mots de passe sont quelque peu spéciaux. Il ne faut jamais enregistrer les mots de passe tels quels (en clair) dans la base de données. Si un jour une personne non autorisée accède à votre base de données, elle aura accès à tous les mots de passe de vos utilisateurs, ce qui serait plutôt embêtant du point de vue de la sécurité.

Pour éviter cela, il faut donner le mot de passe à une fonction de hachage qui va le transformer en une autre chaîne de caractères, ce que nous appelons une « empreinte » ou un *hash*. Cette fonction est à sens unique : le même mot de passe donnera toujours la même empreinte, en revanche, nous ne pouvons pas obtenir le mot de passe uniquement à partir de l'empreinte. En utilisant cette méthode, même si quelqu'un accède aux mots de passe enregistrés dans la base de données, il ne pourra rien en faire. Chaque fois qu'un utilisateur voudra se connecter, il suffira d'appliquer la même fonction de hachage au mot de passe fourni lors de la connexion, et de vérifier si celui-ci correspond bien à celui enregistré dans la base de données.

Quelle est la bonne nouvelle dans tout ça ? Django le fait automatiquement ! En effet, tout à l'heure nous avons renseigné le mot de passe « m0nsup3rm0td3p4ss3 » pour notre utilisateur. Regardons ce qui a réellement été enregistré :

```python
>>> user.password
'pbkdf2_sha256$10000$cRu9mKvGzMzW$DuQc3ZJ3cjT37g0TkiEYrfDRRj57LjuceDyapH/qjvQ='
```

Le résultat est plutôt inattendu. Tous les mots de passe sont enregistrés selon cette disposition : `algorithme$iterations$sel$empreinte`.

 - **Algorithme** : le nom de l'algorithme de la fonction de hachage utilisée pour le mot de passe (ici *pbkdf2_sha256*, la fonction de hachage par défaut de Django). Il est possible de changer d'algorithme par défaut, tout en gardant la validité des anciens mot de passe, de cette manière ;
 - **Itérations** : le nombre de fois que l'algorithme va être exécuté afin de ralentir le processus. Si le chiffrement est plus lent, alors cela ralenti le nombre d'essais possible à la seconde via le bruteforce. Rassurez-vous, cette lenteur est invisible à l'oeil nu sur un essai ; 
 - **Sel** : le sel est une chaîne de caractères insérée dans le mot de passe originel pour rendre son déchiffrage plus difficile (ici *cRu9mKvGzMzW*). Django s'en charge tout seul, inutile de s'y attarder ;
 - **Empreinte** : l'empreinte finale, résultat de la combinaison du mot de passe originel et du sel par la fonction de hachage après le nombre d'itérations précisé. Elle représente la majeure partie de `user.password`.

Maintenant que vous savez que le champ `password` ne s'utilise pas comme un champ classique, comment l'utiliser ? Django fournit quatre méthodes au modèle User pour la gestion des mots de passe :

 - `set_password(mot_de_passe)` : permet de modifier le mot de passe de l'utilisateur par celui donné en argument. Django va hacher ce dernier comme vu précédemment. Cette méthode ne sauvegarde pas l'entrée dans la base de données, il faut faire un `.save()` par la suite.
 - `check_password(mot_de_passe)` : vérifie si le mot de passe donné en argument correspond bien à l'empreinte enregistrée dans la base de données. Retourne `True` si les deux mots de passe correspondent, sinon `False`.
 - `set_unusable_password()` : permet d'indiquer que l'utilisateur n'a pas de mot de passe défini. Dans ce cas, `check_password` retournera toujours `False`.
 - `has_usable_password()` : retourne `True` si le compte utilisateur a un mot de passe valide, `False` si `set_unusable_password` a été utilisé.

Petit exemple pratique désormais, en reprenant notre utilisateur de tout à l'heure :

```python
>>> user = User.objects.get(username="Maxime")
>>> user.set_password("coucou")     # Nous changeons le mot de passe
>>> user.check_password("salut")    # Nous essayons un mot de passe invalide
False
>>> user.check_password("coucou")   # Avec le bon mot de passe, ça marche !
True
>>> user.set_unusable_password()    # Nous désactivons le mot de passe
>>> user.check_password("coucou")   # Comme prévu, le mot de passe précédent n'est plus bon
False
```

### Étendre le modèle User

Pour terminer ce sous-chapitre, abordons l'extension du modèle `User`. Nous avons vu plus tôt que les champs de User étaient assez limités, ce qui pourrait se révéler embêtant si nous souhaitons par exemple adjoindre un avatar à chaque utilisateur.

La solution est d'étendre le modèle avec un autre modèle contenant tous les champs que vous souhaitez ajouter à votre modèle utilisateur. Une fois ce modèle spécifié, il faut le lier au modèle `User` en ajoutant un `OneToOneField` vers ce dernier.

Imaginons que nous souhaitions donner la possibilité à un utilisateur d'avoir un avatar, une signature pour ses messages, un lien vers son site web et de pouvoir s'inscrire à la newsletter de notre site. Notre modèle ressemblerait à ceci :

```python
from django.contrib.auth.models import User

class Profil(models.Model):
    user = models.OneToOneField(User)  # La liaison OneToOne vers le modèle User
    site_web = models.URLField(blank=True)
    avatar = models.ImageField(null=True, blank=True, upload_to="avatars/")
    signature = models.TextField(blank=True)
    inscrit_newsletter = models.BooleanField(default=False)

    def __str__(self):
        return "Profil de {0}".format(self.user.username)
```

Les différents attributs que nous avons listés sont bel et bien repris dans notre modèle `Profil`, dont notamment la liaison vers le modèle `User`.

N'oubliez pas que vous pouvez accéder à l'instance `Profil` associée à une instance `User` depuis cette dernière en utilisant la relation inverse créée automatiquement par le `OneToOneField`. Pour illustrer le fonctionnement de cette relation inverse, voici un petit exemple :

```python
>>> from django.contrib.auth.models import User
>>> from blog.models import Profil
>>> user = User.objects.create_user('Mathieu', 'mathieu@crepes-bretonnes.com', 'sup3rp@ssw0rd')  # Nous créons un nouvel utilisateur
>>> profil = Profil(user=user, site_web="http://www.crepes-bretonnes.com",signature="Coucou ! C'est moi !")
>>> profil.save()
>>> profil
<Profil: Profil de Mathieu>
>>> user.profil
<Profil: Profil de Mathieu>
>>> user.profil.signature
"Coucou ! C'est moi !"
```

Comme vous pouvez le voir, la création de l'objet `Profil` n'est pas automatique (comme dans n'importe quelle relation OneToOne). Vous pouvez toutefois l'automatiser via un signal `post_save`, comme nous l'avons vu dans le chapitre sur les signaux.

Voilà ! Le modèle `User` est désormais correctement étendu avec les nouveaux attributs et méthodes de `Profil`.  
Sachez que si vous souhaitez avoir une méthode d'authentification personnelle (utiliser l'adresse e-mail comme identifiant ou passer par un serveur LDAP), c'est possible de redéfinir totalement le modèle `User`. Cependant, la décision de remplacer le modèle `User` doit être prise au début du projet, afin d'éviter de nombreux problèmes de dépendances au niveau de la base de données, avec les `ForeignKey` vers le modèle utilsiateur.

Passons aux vues
----------------

Maintenant que nous avons assimilé les bases, il est temps de passer aux vues permettant à un utilisateur de se connecter, déconnecter, etc. Nous n'aborderons pas ici l'enregistrement de nouveaux utilisateurs. En effet, nous avons déjà montré la fonction à utiliser, et le reste est tout à fait classique : un formulaire, une vue pour récupérer et enregistrer les informations, un template…

### La connexion

Nous avons désormais des utilisateurs, ils n'ont plus qu'à se connecter ! Pour ce faire, nous aurons besoin des éléments suivants :

 - Un formulaire pour récupérer le nom d'utilisateur et le mot de passe ;
 - Un template pour afficher ce formulaire ;
 - Une vue pour récupérer les données, les vérifier, et connecter l'utilisateur.

Commençons par le formulaire. Il ne nous faut que deux choses : le nom d'utilisateur et le mot de passe. Autrement dit, le formulaire est très simple. Nous le plaçons dans un fichier `forms.py` :

```python
class ConnexionForm(forms.Form):
    username = forms.CharField(label="Nom d'utilisateur", max_length=30)
    password = forms.CharField(label="Mot de passe", widget=forms.PasswordInput)
```

Rien de compliqué, si ce n'est le widget utilisé : `forms.PasswordInput` permet d'avoir une boîte de saisie dont les caractères seront masqués, afin d'éviter que le mot de passe ne soit affiché en clair lors de sa saisie.

Passons au template :

```html
<h1>Se connecter</h1>

{% if error %}
<p><strong>Utilisateur inconnu ou mauvais de mot de passe.</strong></p>
{% endif %}

{% if user.is_authenticated %}
Vous êtes connecté, {{ user.username }} !
{% else %}
<form method="post" action=".">
    {% csrf_token %}
    {{ form.as_p }}
    <input type="submit" value="Se connecter" />
</form>
{% endif %}
```

La nouveauté ici est la variable `user`, qui contient l'instance `User` de l'utilisateur s'il est connecté, ou une instance de la classe `AnonymousUser`. La classe `AnonymousUser` est utilisée pour indiquer que le visiteur n'est pas un utilisateur connecté. `User` et `AnonymousUser` partagent certaines méthodes comme `is_authenticated`, qui permet de définir si le visiteur est connecté ou non. Une instance `User` retournera toujours `True`, tandis qu'une instance `AnonymousUser` retournera toujours `False`.  
La variable `user` dans les templates est ajoutée par un processeur de contexte inclus par défaut. Le même objet est disponible via `request.user` dans les vues.

Notez l'affichage du message d'erreur si la combinaison utilisateur/mot de passe est incorrecte.

Pour terminer, passons à la partie intéressante : la vue. Récapitulons auparavant tout ce qu'elle doit faire :

 1. Afficher le formulaire ;
 2. Après la saisie par l'utilisateur, récupérer les données ;
 3. Vérifier si les données entrées correspondent bien à un utilisateur ;
 4. Si c'est le cas, le connecter et le rediriger vers une autre page ;
 5. Sinon, afficher un message d'erreur.

Vous savez d'ores et déjà comment réaliser les étapes 1, 2 et 5. Reste à savoir comment vérifier si les données sont correctes, et si c'est le cas connecter l'utilisateur. Pour cela, Django fournit deux fonctions, `authenticate` et `login`, toutes deux situées dans le module `django.contrib.auth`. Voici comment elles fonctionnent :

 - `authenticate(username=nom, password=mdp)` : si la combinaison utilisateur/mot de passe est correcte, `authenticate` renvoie l'entrée du modèle `User` correspondante. Si ce n'est pas le cas, la fonction renvoie `None`.
 - `login(request, user)` : permet de connecter l'utilisateur. La fonction prend l'objet `HttpRequest` passé à la vue par le framework, et l'instance de `User` de l'utilisateur à connecter.

**Attention !** Avant d'utiliser `login` avec un utilisateur, vous devez avant avoir utilisé `authenticate` avec le nom d'utilisateur et le mot de passe correspondant, sans quoi `login` n'acceptera pas la connexion. Il s'agit d'une mesure de sécurité.

Désormais, nous avons tout ce qu'il nous faut pour coder notre vue. Voici notre exemple :

```python
from django.contrib.auth import authenticate, login

def connexion(request):
    error = False

    if request.method == "POST":
        form = ConnexionForm(request.POST)
        if form.is_valid():
            username = form.cleaned_data["username"]
            password = form.cleaned_data["password"]
            user = authenticate(username=username, password=password)  # Nous vérifions si les données sont correctes
            if user:  # Si l'objet renvoyé n'est pas None
                login(request, user)  # nous connectons l'utilisateur
            else: # sinon une erreur sera affichée
                error = True
    else:
        form = ConnexionForm()

    return render(request, 'connexion.html', locals())
```

Et finalement la directive de routage :

```python
url(r'^connexion$', 'auth.views.connexion', name='connexion'),
```

Vous pouvez désormais essayer de vous connecter depuis l'adresse `/connexion`. Vous devrez soit créer un compte manuellement dans la console si cela n'a pas été fait auparavant grâce à la commande `manage.py createsuperuser`, soit renseigner le nom d'utilisateur et le mot de passe du compte super-utilisateur que vous avez déjà créé.

Si vous entrez une mauvaise combinaison, un message d'erreur sera affiché, sinon, vous serez connectés !

### La déconnexion

Heureusement, la déconnexion est beaucoup plus simple que la connexion. En effet, il suffit d'appeler la fonction `logout` de `django.contrib.auth`. Il n'y a même pas besoin de vérifier si le visiteur est connecté ou non (mais libre à vous de le faire si vous souhaitez ajouter un message d'erreur si ce n'est pas le cas par exemple).


```python
from django.contrib.auth import logout
from django.shortcuts import render  
from django.core.urlresolvers import reverse

def deconnexion(request):    
    logout(request)      
    return redirect(reverse(connexion))
```

Avec comme routage : 

```python
url(r'^deconnexion$', 'auth.views.deconnexion', name='deconnexion'),
```

### Intéragir avec le profil utilisateur

Comme nos utilisateurs peuvent désormais se connecter et se déconnecter, il ne reste plus qu'à pouvoir interagir avec eux. Nous avons vu précédemment qu'un processeur de contexte se chargeait d'ajouter une variable reprenant l'instance `User` de l'utilisateur dans les templates. Il en va de même pour les vues.

En effet, l'objet `HttpRequest` passé à la vue contient également un attribut `user` qui renvoie l'objet utilisateur du visiteur. Celui-ci peut, encore une fois, être une instance `User` s'il est connecté, ou `AnonymousUser` si ce n'est pas le cas. Exemple dans une vue :

```python
from django.http import HttpResponse

def dire_bonjour(request):
    if request.user.is_authenticated(): 
        return HttpResponse("Salut, {0} !".format(request.user.username)) 
    return HttpResponse("Salut, anonyme.")
```

Maintenant, imaginons que nous souhaitions autoriser l'accès de certaines vues _uniquement_ aux utilisateurs connectés. Nous pourrions vérifier si l'utilisateur est connecté, et si ce n'est pas le cas le rediriger vers une autre page, mais cela serait lourd et redondant. Pour éviter de se répéter, Django fournit un décorateur très pratique qui nous permet de nous assurer qu'uniquement des visiteurs authentifiés accèdent à la vue. Son nom est `login_required` et il se situe dans `django.contrib.auth.decorators`. En voici un exemple d'utilisation :

```python
from django.contrib.auth.decorators import login_required

@login_required
def ma_vue(request):
    # …
```

Si l'utilisateur n'est pas connecté, il sera redirigé vers l'URL de la vue de connexion. Cette URL est normalement définie depuis la variable `LOGIN_URL` dans votre `settings.py`. Si ce n'est pas fait, la valeur par défaut est `'/accounts/login/'`. Comme nous avons utilisé l'URL `'/connexion/'` tout à l'heure, réindiquons-la ici :

```python
LOGIN_URL = '/connexion/'
```

Il faut savoir que si l'utilisateur n'est pas connecté, non seulement il sera redirigé vers `'/connexion/'`, mais l'URL complète de la redirection sera `"/connexion/?next=/bonjour/"`. En effet, Django ajoute un paramètre GET appelé `next` qui contient l'URL d'où provient la redirection. Si vous le souhaitez, vous pouvez récupérer ce paramètre dans la vue gérant la connexion, et ensuite rediriger l'utilisateur vers l'URL fournie. Néanmoins, ce n'est pas obligatoire.

Sachez que vous pouvez également préciser le nom de ce paramètre (au lieu de `next` par défaut), via l'argument `redirect_field_name` du décorateur :

```python
from django.contrib.auth.decorators import login_required

@login_required(redirect_field_name='rediriger_vers')
def ma_vue(request):
    # …
```

Vous pouvez également spécifier une autre URL de redirection pour la connexion, au lieu de prendre `LOGIN_URL` dans le `settings.py` :


```python
from django.contrib.auth.decorators import login_required

@login_required(login_url='/connexion_pour_concours/')
def jeu_concours(request):
    # …
```

Enfin, comme pour les modèles, Django utilise des signaux pour certains événements utilisateurs : 


 - `user_logged_in` : Envoyé quand un utilisateur se connecte, avec `request` et `user` en argument ;
 - `user_logged_out` : Envoyé quand un utilisateur se déconnecte, avec `request` et `user` en argument ;
 - `user_login_failed` : Envoyé quand une tentative de connexion a échoué avec `credentials` en argument, contenant des informations sur la tentative.


Les vues génériques
-------------------

L'application `django.contrib.auth` contient certaines vues qui permettent de réaliser les tâches communes d'un système utilisateurs sans devoir écrire une seule vue : se connecter, se déconnecter, changer le mot de passe et récupérer un mot de passe perdu.

Pourquoi alors avoir expliqué comment gérer manuellement la (dé)connexion ? En réalité, ces vues répondent à un besoin basique. Si vous avez besoin d'implémenter des spécificités lors de la connexion, il est important de savoir comment procéder manuellement.

Pour utiliser ces vues, il suffit de leur assigner une URL et de passer les éventuels paramètres que vous souhaitez changer : 

    (r'^connexion$', 'django.contrib.auth.views.login', {'template_name': 'auth/connexion.html'})

Nous ne ferons ensuite que les lister, avec leurs paramètres et modes de fonctionnement.

### Se connecter

Vue : `django.contrib.auth.views.login`. Arguments optionnels :
 
 - `template_name` : le nom du template à utiliser (par défaut `registration/login.html`).

Contexte du template : 

 - `form` : le formulaire à afficher ;
 - `next` : l'URL vers laquelle l'utilisateur sera redirigé après la connexion.

Affiche le formulaire et se charge de vérifier si les données saisies correspondent à un utilisateur. Si c'est le cas, la vue redirige l'utilisateur vers l'URL indiquée dans `settings.LOGIN_REDIRECT_URL` ou vers l'URL passée par le paramètre GET `next` s'il y en a un, sinon il affiche le formulaire. Le template doit pouvoir afficher le formulaire et un bouton pour l'envoyer.


### Se déconnecter

Vue : `django.contrib.auth.views.logout`. Arguments optionnels (un seul à utiliser) : 
 
 - `next_page` : l'URL vers laquelle le visiteur sera redirigé après la déconnexion ;
 - `template_name` : le template à afficher en cas de déconnexion (par défaut `registration/logged_out.html`) ;
 - `redirect_field_name` : utilise pour la redirection l'URL du paramètre GET passé en argument.

Contexte du template :

 - `title` : chaîne de caractères contenant « Déconnecté ».

Déconnecte l'utilisateur et le redirige.


### Se déconnecter puis se connecter

Vue : `django.contrib.auth.views.logout_then_login`. Arguments optionnels :

 - `login_url` : l'URL de la page de connexion à utiliser (par défaut utilise `settings.LOGIN_URL`).

Contexte du template : aucun.

Déconnecte l'utilisateur puis le redirige vers l'URL contenant la page de connexion.

### Changer le mot de passe

Vue : `django.contrib.auth.views.password_change`. Arguments optionnels : 

 - `template_name` : le nom du template à utiliser (par défaut `registration/password_change_form.html`) ;
 - `post_change_redirect` : l'URL vers laquelle rediriger l'utilisateur après le changement du mot de passe ;
 - `password_change_form` : pour spécifier un autre formulaire que celui utilisé par défaut.

Contexte du template :

- `form` : le formulaire à afficher

Affiche un formulaire pour modifier le mot de passe de l'utilisateur, puis le redirige si le changement s'est correctement déroulé. Le template doit contenir ce formulaire et un bouton pour l'envoyer.


### Confirmation du changement de mot de passe

Vue : `django.contrib.auth.views.password_change_done`.
Arguments optionnels :


- `template_name` : le nom du template à utiliser (par défaut `registration/password_change_done.html`).

Contexte du template : aucun.

Vous pouvez vous servir de cette vue pour afficher un message de confirmation après le changement de mot de passe. Il suffit de faire pointer la redirection de `django.contrib.auth.views.password_change` sur cette vue.


### Demande de réinitialisation du mot de passe

Vue : `django.contrib.auth.views.password_reset`. Arguments optionnels :

 - `template_name` : le nom du template à utiliser (par défaut `registration/password_reset_form.html`) ;
 - `email_template_name` : le nom du template à utiliser pour générer l'e-mail qui sera envoyé à l'utilisateur avec le lien pour réinitialiser le mot de passe (par défaut `registration/password_reset_email.html`) ;
 - `html_email_template_name`: le nom du template HTML à utiliser pour l'e-mail. Par défaut, l'e-mail n'est envoyé qu'au format texte.
 - `subject_template_name` : le nom du template à utiliser pour générer le sujet de l'e-mail envoyé à l'utilisateur (par défaut `registration/password_reset_subject.txt`) ;
 - `password_reset_form` : pour spécifier un autre formulaire à utiliser que celui par défaut ;
 - `post_reset_direct` : l'URL vers laquelle rediriger le visiteur après la demande de réinitialisation ;
 - `from_email` : une adresse e-mail valide depuis laquelle sera envoyé l'e-mail (par défaut, Django utilise `settings.DEFAULT_FROM_EMAIL`).
 

Contexte du template :

 - `form` : le formulaire à afficher.

Contexte de l'e-mail et du sujet :

 - `user` : l'utilisateur concerné par la réinitialisation du mot de passe ;
 - `email` : un alias pour `user.email` ;
 - `domain` : le domaine du site web à utiliser pour construire l'URL (utilise `request.get_host()` pour obtenir la variable) ;
 - `protocol` : `http` ou `https` ;
 - `uid` : l'ID de l'utilisateur encodé en base 36 ;
 - `token` : le `token` unique de la demande de réinitialisation du mot de passe.

La vue affiche un formulaire permettant d'indiquer l'adresse e-mail du compte à récupérer. L'utilisateur recevra alors un e-mail (il est important de configurer l'envoi d'e-mails, référez-vous à l'annexe sur le déploiement en production pour davantage d'informations à ce sujet) avec un lien vers la vue de confirmation de réinitialisation du mot de passe.

Voici un exemple du template pour générer l'e-mail :

```jinja
Une demande de réinitialisation a été envoyée pour le compte {{ user.username }}. Veuillez suivre le lien ci-dessous :
{{ protocol }}://{{ domain }}{% url 'password_reset_confirm' uidb36=uid token=token %}
```

### Confirmation de demande de réinitialisation du mot de passe

Vue : `django.contrib.auth.views.password_reset_done`. Arguments optionnels :

- `template_name` : le nom du template à utiliser (par défaut `registration/password_reset_done.html`).

Contexte du template : vide.

Vous pouvez vous servir de cette vue pour afficher un message de confirmation après la demande de réinitalisation du mot de passe. Il suffit de faire pointer la redirection de `django.contrib.auth.views.password_reset` sur cette vue.

### Réinitialiser le mot de passe

Vue : `django.contrib.auth.views.password_reset_confirm`. Arguments optionnels :

- `template_name` : le nom du template à utiliser (par défaut `registration/password_reset_confirm.html`) ;
- `set_password_form` : pour spécifier un autre formulaire à utiliser que celui par défaut ;
- `post_reset_redirect` : l'URL vers laquelle sera redirigé l'utilisateur après la réinitialisation.

Contexte du template :

- `form` : le formulaire à afficher ;
- `validlink` : booléen, mis à `True` si l'URL actuelle représente bien une demande de réinitialisation valide.

Cette vue affichera le formulaire pour introduire un nouveau mot de passe, et se chargera de la mise à jour de ce dernier.

### Confirmation de la réinitialisation du mot de passe

Vue : `django.contrib.auth.views.password_reset_complete`. Arguments optionnels :

- `template_name` : le nom du template à utiliser (par défaut `registration/password_reset_complete.html`).

Contexte du template : aucun.

Vous pouvez vous servir de cette vue pour afficher un message de confirmation après la réinitialisation du mot de passe. Il suffit de faire pointer la redirection de `django.contrib.auth.views.password_reset_confirm` sur cette vue.

Les permissions et les groupes
------------------------------

Le système utilisateurs de Django fournit un système de permissions simple, permettant de déterminer si un utilisateur a le droit d'effectuer une certaine action ou non. Les groupes permettent quand à eux de définir des permissions communes à un ensemble d'utilisateurs.

### Les permissions

Une permission a la forme suivante :  `nom_application.nom_permission`. Django crée automatiquement trois permissions pour chaque modèle enregistré. Ces permissions sont notamment utilisées dans l'administration. Si nous reprenons par exemple le modèle `Article` de l'application `blog`, trois permissions sont créées par Django :

- `blog.add_article` : la permission pour créer un article ;
- `blog.change_article` : la permission pour modifier un article ;
- `blog.delete_article` : la permission pour supprimer un article.

Il est bien entendu possible de créer des permissions nous-mêmes. Chaque permission dépend d'un modèle et doit être renseignée dans sa sous-classe `Meta`. Petit exemple en reprenant notre modèle `Article` utilisé au début :

```python
class Article(models.Model):
    titre = models.CharField(max_length=100)
    auteur = models.CharField(max_length=42)
    contenu = models.TextField()
    date = models.DateTimeField(auto_now_add=True, auto_now=False, verbose_name="Date de parution")
    categorie = models.ForeignKey(Categorie)

    def __str__(self):
        return self.titre

    class Meta:
        permissions = (
            ("commenter_article", "Commenter un article"),
            ("marquer_article", "Marquer un article comme lu"),
        )
```

Pour ajouter de nouvelles permissions, il suffit de créer un tuple contenant les paires de vos permissions, avec à chaque fois le nom de la permission et sa description. Après avoir mis à jour la base de données (`createmigration` et `migrate`), il est ensuite possible d'assigner des permissions à un utilisateur dans l'administration (cela se fait depuis la fiche d'un utilisateur).

Par la suite, pour vérifier si un utilisateur possède ou non une permission, la classe `User` possède la méthode `has_perm` : `user.has_perm("blog.commenter_article")`. La méthode renvoie `True` ou `False`, selon si l'utilisateur dispose de la permission ou non. Les permissions de l'utilisateur courant sont également accessibles depuis les templates, encore grâce à un _context processor_, `perms` :

```jinja
{% if perms.blog.commenter_article %}
	<p><a href="/commenter/">Commenter</a></p>
{% endif %}
```

Le lien ici ne sera affiché que si l'utilisateur dispose de la permission pour commenter.

De même que pour le décorateur `login_required`, il existe un décorateur permettant de s'assurer que l'utilisateur qui souhaite accéder à la vue dispose bien de la permission nécessaire. Il s'agit de `django.contrib.auth.decorators.permission_required`.

```python
from django.contrib.auth.decorators import permission_required

@permission_required('blog.commenter_article')
def article_commenter(request, article):
    # …
```

Sachez qu'il est également possible de créer une permission dynamiquement. Pour cela, il faut importer le modèle `Permission`, situé dans `django.contrib.auth.models`. Ce modèle possède les attributs suivants :

 - `name` : le nom de la permission, 50 caractères maximum.
 - `content_type` : un `content_type` pour désigner le modèle concerné.
 - `codename` : le nom de code de la permission.

Donc, si nous souhaitons par exemple créer une permission « commenter un article » spécifique à chaque article, et ce à chaque fois que nous créons un nouvel article, voici comment procéder :

```python
from django.contrib.auth.models import Permission
from blog.models import Article
from django.contrib.contenttypes.models import ContentType

…  # Récupération des données
article.save()

content_type = ContentType.objects.get(app_label='blog', model='Article')
permission = Permission.objects.create(
    codename='commenter_article_{0}'.format(article.id),
    name='Commenter l\'article "{0}"'.format(article.titre),
    content_type=content_type)
```

Une fois que la permission est créée, il est possible de l'assigner à un utilisateur précis de cette façon :

```python
user.user_permissions.add(permission)
```

Pour rappel, `user_permissions` est une relation `ManyToMany` de l'utilisateur vers la table des permissions.

### Les groupes

Un groupe est un regroupement d'utilisateurs auquel nous pouvons assigner des permissions. Une fois qu'un groupe dispose d'une permission, tous ses utilisateurs en disposent automatiquement aussi. Il s'agit donc d'un modèle, `django.contrib.auth.models.Group`, qui dispose des champs suivants :

 - `name` : le nom du groupe (80 caractères maximum) ; 
 - `permissions` : une relation `ManyToMany` vers les permissions, comme `user_permissions` pour les utilisateurs.

Pour ajouter un utilisateur à un groupe, il faut utiliser la relation `ManyToMany` `groups` de `User` :

```python
>>> from django.contrib.auth.models import User, Group
>>> group = Group(name="Les gens géniaux")
>>> group.save()
>>> user = User.objects.get(username="Mathieu")
>>> user.groups.add(group)
```

Une fois cela fait, l'utilisateur « Mathieu » dispose de toutes les permissions attribuées au groupe « Les gens géniaux ». Il conserve également les permissions qui lui ont été attribué spécifiquement. La méthode `user.has_perm('app.nom_perm')` vérifie donc si l'utilisateur à cette permission ou s'il appartient à un groupe ayant la permission `app.nom_perm`.

Voilà ! Vous avez désormais vu de long en large le système utilisateurs que propose Django. Vous avez pu remarquer qu'il est tout aussi puissant que flexible. Inutile donc de réécrire tout un système utilisateurs lorsque le framework en propose déjà un plus que correct.


En résumé
---------

- Django propose une base de modèles qui permettent de décrire les utilisateurs et groupes d'utilisateurs au sein du projet. Ces modèles possèdent l'ensemble des fonctions nécessaires pour une gestion détaillée des objets : `make_password`, `check_password`…
- Il est possible d'étendre le modèle d'utilisateur de base, pour ajouter ses propres champs.
- Le framework dispose également de vues génériques pour la création d'utilisateurs, la connexion, l'inscription, la déconnexion, le changement de mot de passe… En cas de besoin plus spécifique, il peut être nécessaire de les réécrire soi-même.
- Il est possible de restreindre l'accès à une vue aux personnes connectées via `@login_required` ou à un groupe encore plus précis via les permissions.
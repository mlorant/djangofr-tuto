Les tests unitaires 
===================

Un test unitaire est une opération qui vérifie une certaine partie de votre code. Cela vous permet de vérifier certains comportements de fonctions : « est-ce que si nous appelons cette fonction avec ces arguments, nous obtenons bien ce résultat précis ? », ou de vérifier toute suite d'appels de vues de votre application en leur fournissant des données spécifiques. Ces tests pourront être exécutés automatiquement les uns à la suite des autres, chaque fois que vous le désirerez, afin de vérifier la stabilité de votre code.

Dans ce chapitre, nous allons donc apprendre à tester des fonctions ou méthodes spécifiques et des vues entières.

Nos premiers tests
------------------

Avant de commencer, **pourquoi faire des tests unitaires ?**

Ces tests ont plusieurs avantages. Tout d'abord, à chaque fois que vous modifierez ou ajouterez quelque chose à votre code, il vous suffira de lancer tous les tests afin de vous assurer que vous n'avez introduit aucune erreur lors de votre développement. Bien sûr, vous pourriez faire cela manuellement, mais lorsque que vous avez des dizaines de cas de figure à tester, la perte de temps devient considérable. Bien entendu, écrire des tests pour chaque cas prend un peu de temps, mais plus le développement s'intensifie, plus cela devient rentable. Au final, il y a un gain de temps et de productivité.

De plus, lorsqu'un bug s'introduit dans votre code, si vos tests sont bien conçus, ils vous indiqueront exactement où il se trouve, ce qui vous permettra de le résoudre encore plus rapidement.

Enfin, les tests sont considérés comme une preuve de sérieux. Si vous souhaitez que votre projet devienne un logiciel libre, sachez que de nombreux éventuels contributeurs risquent d'être rebutés si votre projet ne dispose pas de tests.

Cela étant dit, attaquons-nous au sujet. 

Reprenons notre modèle `Article` introduit précédemment dans l'application « blog ». Nous y avons adjoint une méthode `est_recent` qui renvoie `True` si la date de parution de l'article est comprise dans les 30 derniers jours, sinon elle renvoie `False` :

```python
class Article(models.Model):
    titre = models.CharField(max_length=100)
    auteur = models.CharField(max_length=42)
    contenu = models.TextField()
    date = models.DateTimeField(auto_now_add=True, auto_now=False, verbose_name="Date de parution")
    categorie = models.ForeignKey(Categorie)

    def est_recent(self):
        """ Retourne True si l'article a été publié dans
            les 30 derniers jours """
        from datetime import datetime
        return (datetime.now() - self.date).days < 30

    def __str__(self):
        return self.titre
```

Une erreur relativement discrète s'y est glissée : que se passe-t-il si la date de parution de l'article se situe dans le futur ? L'article ne peut pas être considéré comme récent, car il n'est pas encore sorti ! Pour détecter une telle anomalie, il nous faut écrire un test.

Les tests sont répartis par application. Chaque application possède en effet par défaut un fichier nommé `tests.py` dans lequel vous devez insérer vos tests. 

Voici notre `tests.py`, incluant le test pour vérifier si un article du futur est récent ou non, comme nous l'avons expliqué ci-dessus :

```python
from django.test import TestCase
from datetime import datetime, timedelta
from models import Article


class ArticleTests(TestCase):
    def test_est_recent_avec_futur_article(self):
        """
        Vérifie si la méthode est_recent d'un Article ne
        renvoie pas True si l'Article a sa date de publication
        dans le futur.
        """

        futur_article = Article(date=datetime.now() + timedelta(days=20))
        #Il n'y a pas besoin de remplir tous les champs ou de sauvegarder l'entrée
        self.assertEqual(futur_article.est_recent(), False)
```


Les tests d'une même catégorie (vérifiant par exemple toutes les méthodes d'un même modèle) sont regroupés dans une même classe, héritant de `django.test.TestCase`. Nous avons ici notre classe `ArticleTests` censée regrouper tous les tests concernant le modèle `Article`.

Dans cette classe, chaque méthode dont le nom commence par `test_` représente un test. Nous avons ajouté une méthode `test_est_recent_avec_futur_article` qui vérifie si un article publié dans le futur est considéré comme récent ou non. Cette fonction crée un article censé être publié dans 20 jours et vérifie si sa méthode `est_recent` renvoie `True` ou `False` (pour rappel, elle devrait renvoyer `False`, mais renvoie pour le moment `True`).

La vérification même se fait grâce à une méthode de `TestCase` nommée `assertEqual`. Cette méthode prend deux paramètres et vérifie s'ils sont identiques. Si ce n'est pas le cas, le test est reporté à Django comme ayant échoué, sinon, rien ne se passe et le test est considéré comme ayant réussi.

Ici, la méthode `est_recent` doit bien renvoyer `False`. Comme nous avons introduit une erreur dans notre modèle, elle est censée renvoyer `True` pour le moment, et donc faire échouer le test.

Afin d'exécuter nos tests, il faut utiliser la commande `python manage.py test` qui lance tous les tests répertoriés :

```console
$ python manage.py test
Creating test database for alias 'default'...
......................................................
======================================================================
FAIL: test_est_recent_avec_futur_article (blog.tests.ArticleTests)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/mathx/crepes_bretonnes/blog/tests.py", line 31, in test_est_recent_avec_futur_article
    self.assertEqual(futur_article.est_recent(), False)
AssertionError: True != False

----------------------------------------------------------------------
Ran 419 tests in 10.256s

FAILED (failures=1, skipped=1)
Destroying test database for alias 'default'...
```

Comme prévu, le test que nous venons de créer a échoué.

[[information]]
| 419 tests ont été lancés. C'est normal, car les applications par défaut de Django (utilisateurs, sessions, etc.) possèdent elles aussi des tests qui ont également été exécutés. Si vous souhaitez juste exécuter les tests de l'application « blog », vous pouvez indiquer `python manage.py test blog`, ou également `manage.py test blog.ArticleTests` pour lancer tous les tests de la classe `ArticleTests`, ou directement spécifier un seul test comme ceci : `python manage.py test blog.ArticleTests.test_est_recent_avec_futur_article`.


Modifions donc la méthode `est_recent` afin de corriger le bug :

```python
def est_recent(self):
    from datetime import datetime
    return (datetime.now() - self.date).days < 30 and self.date < datetime.now()
```

Relançons le test. Comme prévu, celui-ci fonctionne désormais !

```console
python manage.py test
Creating test database for alias 'default'...
..............................................
----------------------------------------------------------------------
Ran 419 tests in 10.165s

OK (skipped=1)
Destroying test database for alias 'default'...
```

Petite précision : vous pouvez préparer votre suite de tests en créant une méthode nommée `setUp` qui permet d'initialiser certaines variables à l'intérieur de votre classe, pour les utiliser dans vos tests par la suite :


```python
class UnTest(TestCase):
    def setUp(self):
        self.une_variable = "Salut !"

    def test_verification(self):
        self.assertEqual(self.une_variable, "Salut !")
```

Testons des vues
----------------

Outre les tests sur des méthodes, fonctions, etc., qui sont spécifiques, il est également possible de tester des vues. Cela se fait grâce à un serveur web intégré au système de test qui sera lancé tout seul lors de la vérification des tests.

Pour tester quelques vues, nous allons utiliser l'application `mini_url` créée lors du TP de la deuxième partie. N'hésitez pas à reprendre le code que nous avons donné en solution si ce n'est pas déjà fait, nous nous baserons sur celui-ci afin de construire nos tests.

Voici le début de notre `mini_url/tests.py`, incluant notre premier test :

```python
from django.test import TestCase
from django.core.urlresolvers import reverse
from models import MiniURL
from views import generer

def creer_url():
    mini = MiniURL(url="http://foo.bar",code=generer(6), pseudo="Maxime")
    mini.save()
    return mini

class MiniURLTests(TestCase):
    def test_liste(self):
        """ Vérifie si une URL sauvegardée est bien affichée """

        mini = creer_url()
        reponse = self.client.get(reverse('mini_url.views.liste'))

        self.assertEqual(reponse.status_code, 200)
        self.assertContains(reponse, mini.url)
        self.assertQuerysetEqual(reponse.context['minis'], [repr(mini)])
```

Nous avons créé une petite fonction nommée `creer_url` qui crée une redirection, l'enregistre et la retourne, afin de ne pas devoir dupliquer le code dans nos futurs tests.

Django créera à chaque séance de tests une nouvelle base de données vide et utilisera celle-ci, au lieu d'utiliser la base de données classique, afin d'éviter de compromettre cette dernière.

Intéressons-nous ensuite au test `test_liste` qui va s'assurer que lorsque nous créons une redirection dans la base de données celle-ci est bien affichée par la vue `views.liste`. Pour ce faire, nous créons tout d'abord une redirection grâce à la fonction `creer_url` et nous demandons ensuite au client intégré au système de test d'accéder à la vue `liste` grâce à la méthode `get` de `self.client`. Cette méthode prend une URL, c'est pourquoi nous utilisons la fonction `reverse` afin d'obtenir l'URL de la vue spécifiée.

`get` retourne un objet dont les principaux attributs sont `status_code`, un entier représentant le code HTTP de la réponse, `content`, une chaîne de caractères contenant le contenu de la réponse, et `context`, le dictionnaire de variables passé au template si un dictionnaire a été utilisé.

Nous pouvons donc vérifier si notre vue s'est bien exécutée en vérifiant le code HTTP de la réponse : `self.assertEqual(reponse.status_code, 200)`.

Pour rappel, 200 correspond à une requête correctement déroulée, 302 à une redirection, 404 à une page non trouvée et 500 à une erreur côté serveur, entre autres.

Deuxième vérification : est-ce que l'URL qui vient d'être créée est bien affichée sur la page ? Cela se fait grâce à la méthode `assertContains` qui prend comme premier argument une réponse comme celle donnée par `get`, et en deuxième argument une chaîne de caractères. La méthode renvoie une erreur si la chaîne de caractères n'est pas contenue dans la page.

Dernière et troisième vérification : est-ce que le `QuerySet` `minis` contenant toutes les redirections dans notre vue (celui que nous avons passé à notre template et qui est accessible depuis `reponse.context`) est égal au `QuerySet` indiqué en deuxième paramètre ? En réalité, le deuxième argument n'est pas un `QuerySet`, mais est censé correspondre à la représentation du premier argument grâce à la fonction `repr`. Autrement dit, il faut que `repr(premier_argument) == deuxieme_argument`. Voici ce à quoi ressemble le deuxième argument dans notre exemple : `['<MiniURL: [ALSWM0] http://foo.bar>']`.

Dans ce test, nous n'avons demandé qu'une simple page. Mais comment faire si nous souhaitons par exemple soumettre un formulaire ? Une telle opération se fait grâce à la méthode `post` de `self.client`, dont voici un exemple à partir de la vue `nouveau` de notre application, qui permet d'ajouter une redirection :

```python
def test_nouveau_redirection(self):
    """ Vérifie la redirection d'un ajout d'URL """
    data = {
        'url': 'http://www.djangoproject.com',
        'pseudo': 'Jean Dupont',
    }
    
    reponse = self.client.post(reverse('mini_url.views.nouveau'), data)
    self.assertEqual(reponse.status_code, 302)
    self.assertRedirects(reponse, reverse('mini_url.views.liste'))
```

La méthode `post` fonctionne comme `get`, si ce n'est qu'elle prend un deuxième argument, à savoir un dictionnaire contenant les informations du formulaire. De plus, nous avons utilisé ici une nouvelle méthode de vérification : `assertRedirects`, qui vérifie que la réponse est bien une redirection vers l'URL passée en paramètre. Autrement dit, si la requête s'effectue correctement, la vue `nouveau` doit rediriger l'utilisateur vers la vue `liste`.

Sachez que si vous gérez des redirections, vous pouvez forcer Django à suivre la redirection directement en indiquant `follow=True` à `get` ou `post`, ce qui fait que la réponse ne contiendra pas la redirection en elle-même, mais la page ciblée par la redirection, comme le montre l'exemple suivant.

```python
def test_nouveau_ajout(self):
    """
    Vérifie si après la redirection l'URL ajoutée est bien dans la liste
    """
    data = {
        'url': 'http://www.crepes-bretonnes.com',
        'pseudo': 'Amateur de crêpes',
    }

    reponse = self.client.post(reverse('mini_url.views.nouveau'), data, follow=True)
    self.assertEqual(reponse.status_code, 200)
    self.assertContains(reponse, data['url'])
```

Dernier cas de figure à aborder : imaginons que vous souhaitiez tester une vue pour laquelle il faut obligatoirement être connecté à partir d'un compte utilisateur, sachez que vous pouvez vous connecter et vous déconnecter de la façon suivante :

```python
c = Client()
c.login(username='utilisateur', password='mot_de_passe')
reponse = c.get('/une/url/')
c.logout()  # La déconnexion n'est pas obligatoire
```


En résumé
---------

 - Les tests unitaires permettent de s'assurer que vous n'avez introduit aucune erreur lors de votre développement, et assurent la robustesse de votre application au fil du temps. 
 - Les tests sont présentés comme une suite de fonctions à exécuter, testant plusieurs assertions. En cas d'échec d'une seule assertion, il est nécessaire de vérifier son code (ou le code du test), qui renvoie un comportement anormal.
 - Les tests sont réalisés sur une base de données différente de celles de développement ; il n'y a donc aucun souci de corruption de données lors de leur lancement.
 - Il est possible de tester le bon fonctionnement des modèles, mais aussi des vues. Ainsi, nous pouvons vérifier si une vue déclenche bien une redirection, une erreur, ou si l'enregistrement d'un objet a bien lieu.
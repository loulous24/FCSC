# Solution aux challenges Crypto
Par loulous24

Je n'explique pas comment j'ai discuté avec le serveur, je le fais dans ma solution des challenges Misc.

## Merry et Pippin

#### Merry

Ici, c'est une histoire de multiplication matricielle. On a A qui est générée aléatoirement et B = A*S_A + E_A. Il faut trouver S_A et E_A connaissant B et A. Vu comme ça, c'est un problème très difficile.

Maintenant, on va utiliser le fait que l'on a le doit de savoir si pour K = f(C- U * S_A) où C, U et K sont des matrices que l'on fournit. f est une fonction qui s'applique à chaque élément de la matrice et effectue un calcul modulaire puis ramène les valeurs trop grandes (la moitié du module) en dessous d'un certain seuil  et divise puis arrondi le tout. Or, on voit que K est une matrice avec une taille faible (4x4) donc on peut essayer de deviner f(C - U * S_A) et bruteforçant sur K. Le nombre d'opération à effectuer reste trop grand et on peut le faire intelligemment. D'abord, en prenant U une matrice nulle partout sauf en haut sur la première ligne. La colonne du coefficient non nul détermine la ligne de S_A que l'on essaie de déterminer.

On obtient seulement une matrice avec une ligne non nulle, réduisant le nombre de possibilités pour K. Prendre C = 0 suffit et on prend le coefficient non nul de U égal à 1024 pour passer la division et l'arrondi sans être ramené à 0. Les coefficients non nuls de K que l'on trouve sont forcément égaux à 2 et correspondent à 1 ou -1 dans la ligne de S_A. Pour savoir si c'est un 1 ou un -1, on prend C qui vaut 512 sur la première ligne là où on a des coefficients non nuls et U qui vaut -512 à la place du 1024. Ainsi, si le coefficient de S_A est égal à un, on a K qui aura un coefficient non nul et pour -1, il sera nul.

En déduire A avec cette méthode nécéssite 2 * 2^4 * 280 et en moyenne la moitié. 5000 interactions avec le serveur est quelque chose d'assez normal quand on voit combien il en faut pour afficher une page Web comme celle de Facebook de nos jours.

#### Pippin

Pippin est très similaire mais on n'a plus le droit qu'à 3000 tests pour deviner. Des optimisations liés à des calculs d'espérance peuvent être fait pour passer proche des 3000 opérations.

Cependant, en affichant les premières lignes de S_A qu'on obtient avec la méthode précédente, on se rend compte que une ligne contient un 1, un -1 et deux 0. En envoyant des requêtes permettant de tester seulement ces possibilités, on réduit à 1500 en ordre de grandeur le nombre moyen de requêtes à envoyer. Cela suffit pour trouver le deuxième flag.

## Macaron

Ici, on cherche à prouver qu'on possède un secret avec HMAC alors qu'on ne l'a pas et qu'on arrive seulement à voir les preuves du serveur.

Ce que fait le code est de prendre notre message, d'ajouter du padding pour qu'il ait une bonne taille et de travailler en faisant un XOR des HMAC que l'on obtient avec le message et un nonce. Sauf que le nonce pour valider, on le fournit au serveur donc on peut le modifier et faire ce qu'on veut. La seule chose qu'il faut, c'est prendre un mot qui ne soit pas un de ceux qu'on a déjà proposé.

Ici, on va jouer avec le fait que si on XOR deux fois quelque chose, cela revient à ne rien faire. On va donc ajouter des données qui vont s'annuler entre elles.

Le plus facile est de poser `s = "<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<"` (30 chevrons), de demander à taguer le message 2 * s dont le padding va être 4 * s. On récupère ce hash que l'on valide avec le message 4 * s qui est différent du précédent. Il va être paddé en 6 * s et on met le nonce `000000010002000300030003`. Ce dernier permet d'annuler les deux versions de s qu'on a ajoutées. En effet, c'est le même que précédemment sauf pour la fin où on a répété la fin du nonce initial pour obtenir exactement les mêmes données dans le HMAC qui vont se compenser.

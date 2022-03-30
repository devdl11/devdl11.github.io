---
title: Le CNED a fait ça ?!
date: 28-03-2022 13:30:00
tag:
  - article
  - hacking
slug: le-cned-fait-ca
---

<style>
    .center {
        text-align: center;
    }

</style>

## Le Cned
{: .center }

Le CNED, pour "Centre National d'Enseignement à Distance", est un [__établissement public à caractère administratif du ministère de l'Éducation nationale offrant des formations à distance__](https://fr.wikipedia.org/wiki/Centre_national_d%27enseignement_à_distance) (merci Wikipédia :b), a été et est toujours connu pour ses piètres services de sa "classe à la maison", qui ont dû être utilisés pendant les confinements et les cours alternés (présentiels + distanciels). 

Dans cet article, nous allons voir les différents problèmes rencontrés durant ces périodes ainsi que les éléments trahissant un manque de savoir faire ou de professionnalisme dans leurs conceptions.

> **Disclaimer**: Toutes les informations révélées dans cet article sont uniquement à but informatives et éducatives et ne doivent absolument pas être utilisées dans le but de dégrader d'une quelconque manière la plateforme. Je vous décourage ainsi de reproduire toute action de cet article et je ne saurais être tenue responsable des conséquences de vos actes !
>
> De plus, cet article est subjectif et traduit uniquement mon point de vue en tant qu'élève et passionné d'informatique, je ne suis **en aucun cas un professionnel** au moment où je 
> rédige ces lignes.
{: .prompt-warning } 

## Une plateforme instable: infrastructure ou application ?
{: .center }

Pour commencer, un petit [nmap](https://fr.wikipedia.org/wiki/Nmap) nous permet de voir que l'application web est hébergée sur [AWS](https://fr.wikipedia.org/wiki/Amazon_Web_Services) (Amazon Web Services) et expose les ports HTTP et HTTPS (80 et 443). Ainsi, au vu des crash incessant s'étalant sur toute une année, il est peu probable que cela vienne uniquement des services Amazon, mais plutôt de l'application elle-même (protection des serveurs, quotas, utilisation, etc.).

> Par application, je parle de la salle d'attente mise en place suite à des individus qui rejoignaient des cours en ligne pour y faire n'importe quoi (et insulter les professeurs… oui, il y a des personnes dans ce monde qui n'ont pas de vie …).
>
> En effet, les cours en eux-mêmes se tenaient sur un autre service en ligne, spécialisé dans le domaine, BlackBoard Collaborate (aka BBCollab), qui a étonnement plutôt bien tenu le coup.
{: .prompt-info }

De plus, rien que le code et la page web de l'application nous confortent dans cette hypothèse, et ce, pour plusieurs raisons: 
1. Un site relevant de l'amateurisme. Dès les premiers jours, le site arborait fièrement le titre `React App`, avec un code identique à celui du Getting Start de React. Attention, je ne condamne pas le fait d'utiliser React, bien au contraire ! C'est juste que lorsqu'un projet passe en production, ce genre de détail ne doit pas rester inchangé (Prenez exemple sur Netflix: il est difficile de voir qu'ils utilisent React !). 
2. Un site peu sécurisé. Mise à part la faille de sécurité assez importante que nous allons voir plus tard, il faut savoir que les professeurs ont accès à une instance de la salle d'attente permettant de gérer les sessions élèves (autoriser, refuser, révoquer, bannir). Le problème avec cette session, c'est qu'elle est directement accessible via son lien, sans besoin d'authentification supplémentaire. Ainsi, un seul petit manque d'attention de la part du professeur durant un partage d'écran et pouf toute la classe a accès à cette fenêtre (rip)
3. Un site instable. En heure de pointe, la salle d'attente devenait en partie voir totalement dysfonctionnel (peut-être un problème avec les serveurs ?). Cela se manifestait par des manques de synchronisation entre la salle d'attente professeur et celles des élèves (un élève pouvait être en SdA sans pour autant que le prof le voit, ou bien ne recevait pas le lien de connexion lorsqu'il se faisait accepter, nécessitant alors un rafraichissement de la page) et par des erreurs de chargements lorsque les serveurs étaient au plus bas.

Face à ces différents problèmes (et aux nombres importants d'heures de trou que cela induisait dans mon planning), je me suis posé la question: "Ne serait-il pas possible de bypass la salle d'attente ?". Etonnament, je ne fus pas prêt à découvrir ce qui suit...

## Des petits liens magiques...
{: .center }

Oui oui, c'est bien ce à quoi vous pensez malheureusement… Mais détaillons tout de même la démarche suivie ;-)

Pendant 2 jours consécutifs, j'ai enregistré à l'aide de [Charles](https://www.charlesproxy.com) (application proxy sympathique) le trafic de mon ordinateur durant mes cours en ligne (enregistrement que je possède encore à ma grande surprise !). Après ces 48 heures de collecte de données, j'ai épluché les logs à la recherche de paternes dans les différentes requêtes tout en notant les différentes étapes permettant d'arriver sur BBCollab.

Ainsi, voici ce que j'ai trouvé:
1. Après votre connexion sur `lycee.cned.fr`, vous êtes redirigés dans la salle d'attente.
2. Durant le chargement de la page, le site va tout d'abord récupérer vos informations au sein de la salle d'attente (nom, prénom, classe, si vous avez été accepté, etc.) puis lancer un service "event" permettant à la page de réagir selon les actions du professeur.  
![Infos Cned](/assets/img/blog/cnedarticle/infocned.png){: style="display: block; margin-left: auto; margin-right: auto;" }
3. Enfin, si vous êtes acceptez (`"canAccess": true`), alors la page va récupérer votre url session utilisateur BBCollab puis vous rediriger dessus.  
![Cned BBCollab Url](/assets/img/blog/cnedarticle/charlesbbcollaburl.png){: style="display:block; margin-left:auto; margin-right:auto;" }

Bon, je ne détaillerai pas plus que cela les différentes requêtes sinon cela prendrait trop de temps et n'apporterait pas plus d'informations utiles.
Maintenant que nous avons les requêtes élèves, jetons un coup d'oeil à la salle d'attente, pour voir s'il n'y pas d'autres informations par rapport à son fonctionnement. Pour cela, il suffit d'ouvrir un des liens enregistrés, de faire un petit F12 et de regarder les sources. 
![F12 cned](/assets/img/blog/cnedarticle/f12cned.png){: style="display:block; margin-left:auto; margin-right:auto;" }
Comme vous pouvez le voir, nous avons carrément accès au code js non minify grâce au source map disponible en production, ce qui simplifie grandement la compréhension du code... mais cela ne nous aide pas plus... Du coup, il nous faut un accès à la page professeur, même si non fonctionnel, car il y a fort a parier qu'il y ait aussi la source map. 

Il existe plusieurs façons de faire, personnellement j'ai opté pour une petite recherche Google : `"classevirtuelle.cned.fr/professor/"`. On défile un peu et voilà: 
![Recherche Google](/assets/img/blog/cnedarticle/cnedgoogle.png){: style="display:block; margin-left:auto; margin-right:auto;" }

On visite la page web et voici ce qu'on obtient: 
![Cned prof](/assets/img/blog/cnedarticle/cnedprof.png){: style="display:block; margin-left:auto; margin-right:auto;" }
... et avec le F12:
![F12 prof](/assets/img/blog/cnedarticle/cnedsourceprof.png){: style="display:block; margin-left:auto; margin-right:auto;" }
Ici j'ai encadré les **liens magiques** utilisés pour envoyer des commandes à la salle d'attente, liens que n'importe qui peut utiliser pour entre autres bypass la salle d'attente.
> Wait ! Il y a un `Bearer Token` qui nous va nous empêcher d'exécuter ces commandes ! Ca marche pas ton truc !  

Oui, vous avez raison, il y a un `Bearer Token`, mais le truc c'est que... Ils ne se sont pas compliqués la vie et le `uuid` du token correspond tout simplement à ce qui se trouve dans l'url après `/professor/` 😂 (à savoir que pendant les 6 premiers mois il n'y avait même pas besoin de renseigner ce token :b). 

Ainsi, si vous possèdez l'id d'une personne (ou bien votre id), vous pouvez faire exactement ce que le prof peut faire, soit l'accepter, etc. Mais ce n'est pas tout ! J'ai gardé le meilleur pour la fin !

Lorsqu'un élève arrive dans la salle d'attente, l'application est faite de telle sorte qu'il faut qu'il envoie ses informations au serveur (pire idée ever soit dit en passant). Comme vous l'avez sûrement deviné, cela se fait via une petite requête que voici:

![Cned registration](/assets/img/blog/cnedarticle/requestslogin.png){: style="display:block; margin-left:auto; margin-right:auto;" }
Et voici les infos envoyées:

![Cned registration infos](/assets/img/blog/cnedarticle/cnedrequestpost.png){: style="display:block; margin-left:auto; margin-right:auto;" }
Je vous donne dans le mille, il est possible de créer soit même des comptes élèves. Il suffit juste de changer les valeurs `name` et `surname` par ce que l'on veut et de mettre dans `email` une adresse mail qui n'a pas encore été utilisée (elle n'a même pas besoin d'être valide ! Par exemple `azdDAZ865@azd79653.cazefd` fonctionne parfaitement !).

## Mot de la fin
{: .center }

Pour des raisons évidentes, je ne publierai aucun script permettant d'utiliser ces failles, car mon but n'est pas de vous inciter à les utiliser (il s'agit tout de même du CNED !), mais vous laisse imaginer le champ des possibles si vous combinez le tout ;)

Cet article est publié uniquement à des fins éducatives, je ne saurais être tenu pour responsable d'une utilisation malveillante du contenu de cet article.

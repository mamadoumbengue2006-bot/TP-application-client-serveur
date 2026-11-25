# TP2 — Architectures 2-tiers et 3-tiers comparées

Questions et réponses transcrites à l'identique du compte rendu `TP2_Architectures_2tiers_et_3tiers_Comparées.pdf`.

---

## Q1

**Question.** Relevez dans le code les lignes qui relèvent de la présentation, de la logique métier et de l'accès aux données. Sur quelle machine s'exécutent-elles toutes ?

**Réponse.**

| Couche | Lignes de client_lourd.py |
|---|---|
| Présentation | Tous les print() et input() : menu et boucle de main() (l. 96 à 113), affichage de la liste (l. 53-54), du relevé (l. 63, 69, 73) et du classement (l. 89 à 93). |
| Logique métier | SEUIL_VALIDATION = 10.0 (l. 29) ; fonction moyenne(), moyenne pondérée par les crédits (l. 39-42) ; décision VALIDÉ ou NON VALIDÉ (l. 72) ; regroupement par étudiant, tri du classement et décompte des admis (l. 85 à 92). |
| Accès aux données | Paramètres de connexion (l. 22 à 26) ; connexion() (l. 33-36) ; requêtes cur.execute("SELECT ...") (l. 47, 52, 58, 64-66, 80-81) ; octets_recus() (l. 45-48). |

Les trois couches s'exécutent sur la même machine : le poste client (Windows ou VM2). Le serveur MariaDB ne fait que stocker les données et renvoyer les lignes demandées.

---

## Q2

**Question.** Quelles informations un étudiant qui récupère client_lourd.py obtient-il sur le serveur de base de données ? Que peut-il faire avec, en dehors du programme ?

**Réponse.** Le fichier contient en clair, aux lignes 22 à 26, l'adresse du serveur (192.168.1.185), le port (3306), le nom de la base (scolarite), l'utilisateur (etudiant) et son mot de passe. Avec ces informations, l'étudiant peut se connecter directement à MariaDB sans passer par le programme, par exemple avec mariadb -h 192.168.1.185 -u etudiant -p scolarite. Il peut alors exécuter n'importe quel SELECT sur toute la base (toutes les notes, tous les étudiants, toutes les tables), et pas seulement ce que l'application affiche. Il peut aussi réutiliser ce compte depuis un autre programme ou une autre machine du réseau.

---

## Q3

**Question.** Qu'aurait-il fallu installer sur les 300 postes d'une salle informatique pour déployer cette application ?

**Réponse.** Sur chaque poste, il faudrait installer Python, la bibliothèque pymysql et le programme client_lourd.py, soit 300 installations. Chaque nouvelle version exigerait aussi 300 mises à jour, avec le risque que certains postes restent sur l'ancienne version.

---

## Q4

**Question.** Les deux architectures donnent-elles la même moyenne pour ESP260001 et le même nombre d'admis ? Pourquoi est-ce indispensable pour cette expérience ?

**Réponse.** Oui. Pour ESP260001 (Rokhaya SY), le client lourd et l'application web affichent la même moyenne pondérée, 10.75 / 20, avec la même décision VALIDÉ. Le nombre d'admis est lui aussi identique : 18 sur 30 dans le client lourd et dans l'application web . Le classement est le même, de ESP260018 (15.77) à ESP260016 (13.22) pour les dix premiers.

Cette égalité est indispensable : l'expérience compare des architectures, donc seule l'architecture doit changer. Si les données ou les règles de calcul différaient, les écarts observés ensuite (volume transféré, déploiement d'une évolution, sécurité) ne pourraient pas être attribués à l'architecture. Avec les mêmes données et les mêmes règles, toute différence mesurée vient de l'endroit où s'exécute chaque couche.

---

## Q5

**Question.** Qu'a dû installer le poste Windows pour utiliser la version 3-tiers ? Comparez avec Q3.

**Réponse.** Rien : le navigateur, déjà présent, suffit. Il n'a fallu ni Python, ni pymysql, ni programme à copier. En 2-tiers (Q3), il fallait installer Python, pymysql et client_lourd.py sur chacun des 300 postes, et les mettre à jour à chaque version. En 3-tiers, l'installation et les mises à jour se font une seule fois, sur le serveur d'application, et les 300 postes en profitent immédiatement.

---

## Q6

**Question.** Que renvoie la requête vers config.php ? Expliquez pourquoi le mot de passe n'est jamais envoyé au navigateur.

**Réponse.** curl.exe -i http://192.168.1.182/scolarite/inc/config.php renvoie HTTP/1.1 200 OK avec Content-Length: 0 : le corps de la réponse est vide. Le fichier contient les paramètres de connexion à la base, dont le mot de passe, mais Apache ne le renvoie pas tel quel : il le confie à l'interpréteur PHP, qui l'exécute sur le serveur. Ce fichier ne fait que définir des paramètres et n'affiche rien, donc rien ne part vers le client. Le mot de passe reste sur le serveur d'application ; le navigateur ne reçoit que le résultat de l'exécution des pages, c'est-à-dire du HTML.

---

## Q7

**Question.** Le compte app_scolarite n'est autorisé que depuis une seule adresse. Combien de comptes MariaDB faut-il en 3-tiers, quel que soit le nombre d'utilisateurs ? Et en 2-tiers ?

**Réponse.** En 3-tiers, il faut un seul compte, app_scolarite, utilisé uniquement par le serveur d'application et autorisé seulement depuis son adresse 192.168.1.182. Que l'application serve 10 ou 10 000 utilisateurs, MariaDB ne voit qu'une machine et un compte, et les postes clients n'ont aucun droit sur la base. En 2-tiers, chaque poste se connecte lui-même à MariaDB : il faut soit un compte partagé dont le mot de passe est copié dans les 300 programmes et autorisé depuis tout le réseau (comme etudiant@192.168.1.%), soit un compte par utilisateur, dont le nombre augmente avec celui des utilisateurs.

---

## Q8

**Question.** Ouvrez le code source de la page dans le navigateur (Ctrl+U). Y trouvez-vous une trace de SQL ou de la règle de calcul ? Qu'est-ce que le navigateur reçoit exactement ?

**Réponse.** Non. Le code source ne contient que du HTML : l'en-tête, le menu, un tableau déjà rempli avec les notes, la ligne « Moyenne pondérée : 10.75 / 20 — VALIDÉ » et le pied de page. On n'y trouve ni requête SQL, ni formule de moyenne, ni seuil de validation, ni paramètre de connexion. Le navigateur reçoit uniquement la page finie, produite par le serveur d'application, ainsi que la feuille de style style.css référencée dans l'en-tête. La logique métier et l'accès aux données restent côté serveur.

---

## Q9

**Question.** Relevez la décision pour ESP260011 et le nombre d'admis sur chaque poste en 2-tiers. Que constatez-vous ? Quelles conséquences réelles cela aurait-il pour un service de scolarité ?

**Réponse.** Sur le poste Windows (version 1.0), ESP260011 est VALIDÉ et il y a 18 admis sur 30. Sur VM2 (version 1.1), le même étudiant est NON VALIDÉ et il y a 16 admis sur 30. Les deux postes interrogent la même base, avec les mêmes notes, mais ils appliquent des règles différentes parce que le programme n'a été mis à jour que sur l'un d'eux. Les deux étudiants concernés par l'écart sont ESP260011 et ESP260013, tous deux avec une moyenne supérieure à 10 et une note de MAT inférieure à 8.

Pour un service de scolarité, les conséquences seraient graves : la décision de validation d'un étudiant dépendrait du poste utilisé par l'agent. Deux agents pourraient éditer des listes d'admis et des procès-verbaux contradictoires, avec des étudiants validés ou refusés selon le hasard. Cela entraînerait des contestations et des litiges, des décisions difficiles à défendre, des corrections tardives, et une perte de confiance dans les résultats. Il serait aussi difficile de savoir quels postes sont à jour.

---

## Q10

**Question.** Combien de fichiers et de machines avez-vous modifiés en 3-tiers ? Combien d'utilisateurs ont bénéficié de la nouvelle règle, et à quel moment ?

**Réponse.** En 3-tiers, un seul fichier a été modifié (metier.php), sur une seule machine (le serveur d'application, VM2). Tous les utilisateurs, quel que soit leur poste ou leur nombre, ont bénéficié de la nouvelle règle dès l'enregistrement du fichier, au premier rechargement de leur page : la figure montre la version 1.1 et le résultat NON VALIDÉ sans rien changer sur Windows. En 2-tiers, il faut intervenir sur chaque poste : un fichier par poste, soit 300 fichiers pour 300 postes, et tant qu'un poste n'est pas mis à jour, il applique l'ancienne règle.

---

## Q11

**Question.** Dans metier.php, la moyenne est calculée par le SGBD (SUM(...)/SUM(...)), alors que client_lourd.py la calcule en Python. Où avez-vous dû intervenir pour la règle éliminatoire dans chaque cas ?

**Réponse.** En 2-tiers, toute la logique est dans le code Python du client : il a fallu modifier client_lourd.py à trois endroits (la constante, releve() et classement()), et refaire la même modification sur chaque copie du programme. Le client continue de rapatrier les 180 lignes de notes (4.2 Kios) pour calculer lui-même.

En 3-tiers, nous sommes intervenus uniquement côté serveur, dans metier.php : en PHP pour la décision du relevé, et dans la requête SQL pour le décompte des admis (note minimale par étudiant avec MIN(n.valeur)). Le calcul de la moyenne reste fait par le SGBD, et le classement ne renvoie toujours que le bilan au navigateur. La règle est donc modifiée à un seul endroit, là où se trouve la logique métier.

---

## Q12

**Question.** Complétez le tableau. D'où vient l'écart de volume ? Que deviendrait-il pour 300 postes reliés par une liaison à 10 Mbit/s, ou pour un client sur une connexion mobile ?

**Réponse.** Le tableau est complété dans la section 6.4.

| Mesure | 2-tiers | 3-tiers |
|---|---|---|
| Lignes ou octets reçus par le poste client | 120 180 lignes, 2 407,3 Kio (environ 2,46 Mo) | 1 345 octets (une page HTML : 10 lignes et le bilan) |
| Temps de calcul du classement | 728 ms (Windows), 2 044 ms (VM2) pour rapatrier les notes | 894 ms affichés par la page (figure 30) ; 10 226 ms lors d'une première mesure instable (figure 24) |
| Machine qui a effectué le calcul | Le poste client (Windows ou VM2) | Le serveur de base de données (SGBD, 192.168.1.185) |

Remarque sur le nombre d'admis : Le client lourd de Windows affiche 9 407 admis sur 20 030 (figure 22), alors que VM2 et l'application web en affichent 16 (figures 23 et 24). Cet écart ne vient pas de l'architecture : Windows est resté en version 1.0, sans la règle éliminatoire, alors que VM2 et le web sont en version 1.1. Les notes fictives sont aléatoires entre 0 et 20, donc presque tous les étudiants fictifs ont au moins une note inférieure à 8 et sont éliminés en version 1.1.

L'écart de volume vient de l'endroit où le calcul est fait. En 2-tiers, le client lourd ne sait pas calculer dans la base : il rapatrie toutes les notes (120 180 lignes, 2 407,3 Kio) puis calcule les moyennes et trie lui-même. En 3-tiers, le SGBD calcule les moyennes (GROUP BY), trie et ne garde que les 10 premières lignes ; le serveur d'application en fait une page HTML, et le navigateur ne reçoit que 1 345 octets. Le volume est environ 1 800 fois plus petit. De plus, ce volume augmente avec le nombre de notes en 2-tiers, alors qu'il reste presque constant en 3-tiers (10 lignes).

Pour 300 postes qui lancent le classement en même temps sur une liaison partagée de 10 Mbit/s, le 2-tiers transférerait environ 300 × 2 407 Kio, soit près de 700 Mio (environ 5,9 milliards de bits), ce qui occuperait la liaison pendant environ 10 minutes. Le 3-tiers transférerait environ 300 × 1,3 Kio, soit 400 Kio, transmis en environ 0,3 s. Même pour un seul poste, 2,4 Mo prennent environ 2 s à 10 Mbit/s, contre une milliseconde pour 1 345 octets. Sur une connexion mobile, l'écart est encore plus pénalisant : consommation du forfait de données, débit variable et latence plus élevée, coupures qui interrompent un transfert de plusieurs mégaoctets, et batterie sollicitée. Le 3-tiers reste rapide et peu coûteux dans les deux cas. Ces valeurs sont des calculs à partir de nos mesures, pas des mesures réseau.

---

## Q13

**Question.** Le client lourd lit la valeur Bytes_sent du serveur avant et après la requête : que mesure exactement cette valeur ? Pourquoi est-ce une mesure honnête du trafic ?

**Réponse.** Bytes_sent est un compteur de statut de MariaDB. Le client lit sa valeur de session (SHOW SESSION STATUS LIKE 'Bytes_sent'), c'est-à-dire le nombre total d'octets que le serveur a envoyés sur cette connexion depuis son ouverture. En lisant le compteur avant et après la requête, la différence donne le nombre d'octets envoyés par le serveur pour cette seule requête. C'est une mesure honnête parce qu'elle est faite par le serveur lui-même, et non estimée par le client à partir de la taille des données affichées : elle compte ce qui a réellement été envoyé, avec le protocole MariaDB (en-têtes de paquets, description des colonnes, codage des valeurs). Comme elle porte sur la session, elle n'inclut pas le trafic des autres clients connectés au même moment. Elle ne compte pas les en-têtes TCP/IP : le trafic réseau réel est donc légèrement supérieur, mais l'ordre de grandeur et la comparaison restent valables.

---

## Q14

**Question.** Que se passe-t-il pour chaque architecture ? Pourquoi ne peut-on pas appliquer cette protection en 2-tiers ?

**Réponse.** En 2-tiers, le client lourd ne peut plus se connecter : le serveur ne répond pas sur le port 3306 et la connexion se termine par un délai dépassé. L'application est inutilisable depuis Windows.

En 3-tiers, l'application web continue de fonctionner normalement, parce que c'est le serveur d'application (VM2) qui interroge la base, et que VM2 est la seule machine autorisée par la règle du pare-feu.

Cette protection est impossible en 2-tiers parce que chaque poste client est lui-même un client de MariaDB : il doit joindre directement le port 3306. Pour que l'application fonctionne, il faudrait donc ouvrir ce port à tous les postes, soit 300 adresses à autoriser une par une, ou tout le réseau, sans compter les postes qui changent d'adresse ou qui se connectent de l'extérieur. La base reste alors exposée. En 3-tiers, un seul client légitime accède à la base, le serveur d'application, et une seule règle suffit.

---

## Q15

**Question.** Citez deux autres risques de sécurité propres au client lourd, et la façon dont le 3-tiers les supprime ou les réduit.

**Réponse.**

1. Les identifiants de la base sont dans le programme. Le mot de passe du compte etudiant est écrit dans client_lourd.py (Q2) : quiconque récupère le programme peut se connecter à la base sans lui et lire toutes les tables. En 3-tiers, les identifiants restent dans config.php sur le serveur d'application, qui n'est jamais envoyé au navigateur (Q6), et le compte de l'application n'est utilisable que depuis l'adresse de ce serveur.

2. Le code et les règles s'exécutent sur le poste de l'utilisateur. Un utilisateur peut modifier sa copie du programme (changer le seuil de validation, par exemple), ou continuer à utiliser une ancienne version non mise à jour, comme le poste Windows en version 1.0 (Partie 3). En 3-tiers, il n'existe qu'une copie du code, sur le serveur, contrôlée par l'administrateur, et une correction ou une nouvelle règle s'applique à tous immédiatement.

Le 3-tiers réduit ces risques sans les supprimer : le serveur d'application reste une cible, ses requêtes doivent être protégées contre les injections SQL (requêtes préparées), et la page circule en HTTP non chiffré dans ce TP, ce qu'un déploiement réel remplacerait par HTTPS.

---

## Q16

**Question.** Le 2-tiers reste utilisé : logiciel de caisse d'une boutique, poste de saisie d'un laboratoire, outil d'administration d'une base. Donnez les conditions qui le rendent acceptable, puis justifiez le choix du 3-tiers pour un service de scolarité utilisé par toute l'université.

**Réponse.** Quand le 2-tiers est acceptable. Il convient lorsque les postes sont peu nombreux et sous le contrôle de la même équipe (une caisse, un laboratoire), ce qui rend les installations et les mises à jour faciles à maintenir. Le réseau doit être local, fermé et fiable, les utilisateurs de confiance (l'administrateur d'une base, par exemple), et la base de taille modeste, de sorte que le volume échangé reste raisonnable. Les règles de gestion changent peu, et les données ne sont pas assez sensibles pour que leur exposition aux postes pose problème. Dans ces conditions, sa simplicité et son faible coût l'emportent.

Pourquoi le 3-tiers pour la scolarité. Un service de scolarité utilisé par toute l'université cumule les contraintes que nos mesures montrent difficiles en 2-tiers :

• Nombre et diversité des utilisateurs. Des milliers d'étudiants, d'enseignants et d'agents, sur des postes variés, parfois hors du campus ou sur mobile : un navigateur suffit en 3-tiers, alors que le 2-tiers exigerait une installation et des mises à jour sur chaque poste (Q3, Q5).

• Évolution des règles et cohérence des résultats. Une règle comme la note éliminatoire doit s'appliquer à tous en même temps : un seul fichier modifié en 3-tiers, alors que le 2-tiers a produit deux décisions différentes pour le même étudiant (Q9, Q10).

• Sécurité des données. Les notes sont sensibles : en 3-tiers, la base n'est joignable que par le serveur d'application et les mots de passe restent sur le serveur, alors qu'en 2-tiers le port de la base doit rester ouvert et le mot de passe est distribué avec le programme (Q2, Q6, Q14, Q15).

• Volume de données et réseau. La base grossit chaque année : le 2-tiers transfère toutes les notes à chaque poste (2 407,3 Kio pour 120 180 notes), le 3-tiers quelques centaines d'octets, ce qui reste supportable sur un réseau chargé ou mobile (Q12).

Le 3-tiers a un coût : deux serveurs à installer, à superviser et à dimensionner, et un point de défaillance supplémentaire (si le serveur d'application s'arrête, tous les utilisateurs sont touchés). Ce coût est faible face aux avantages de déploiement, de cohérence et de sécurité pour une application utilisée par toute l'université.

---

## Synthèse (tableau du compte rendu)

| Critère | 2-tiers (client lourd) | 3-tiers (application web) |
|---|---|---|
| Logiciel à installer sur chaque poste | Python, pymysql et client_lourd.py sur chaque poste (Q3). 300 installations pour 300 postes. | Un navigateur, déjà présent (Q5). Tout est installé une seule fois, sur le serveur d'application. |
| Déploiement d'une nouvelle règle | Un fichier à modifier par poste. Un poste non mis à jour garde l'ancienne règle (Q9, Q11). | Un fichier sur une machine. Tous les utilisateurs en profitent dès le rechargement de la page (Q10). |
| Cohérence entre utilisateurs | Risque de résultats différents : ESP260011 VALIDÉ sur Windows (v1.0), NON VALIDÉ sur VM2 (v1.1), 18 admis contre 16 (Q9). | Une seule copie de la règle : tous les utilisateurs obtiennent les mêmes résultats (Q4, Q10). |
| Trafic réseau côté client | 120 180 lignes, 2 407,3 Kio pour un classement. Le volume croît avec la base (Q12). | 1 345 octets, quasiment constant : seules 10 lignes sont renvoyées (Q12). |
| Exposition du SGBD et des mots de passe | La base doit être joignable par tous les postes. Le mot de passe est dans le programme. Le pare-feu ne peut pas fermer le port 3306 (Q2, Q14, Q15). | Seul le serveur d'application joint la base. Le mot de passe reste sur le serveur. Le port 3306 est fermé aux autres machines (Q6, Q7, Q14). |
| Charge des serveurs | Le calcul est réparti sur les postes. Le serveur de données lit et envoie toutes les lignes à chaque client : beaucoup de trafic. | Le calcul est centralisé sur le SGBD et le serveur d'application, qui doivent être dimensionnés. Les échanges sont réduits. |
| Simplicité de développement | Un seul programme et une seule configuration : simple pour un petit projet. | Plus complexe : deux serveurs à installer, couches à séparer, comptes et droits à définir. |

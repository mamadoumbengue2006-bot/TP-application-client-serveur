# TP 1 — Le modèle client-serveur
## Informations générales

| Élément | Valeur reportée dans le compte rendu |
| --- | --- |
| Titre | Compte rendu — TP 1 : Le modèle client-serveur |
| Encadreur | Dr Keba Gueye |
| Étudiants | Mamadou Mbengue, Fatima Gueye |
| Année | 2026/2027 |
| Compte rendu original | [`rapport_TP1.pdf`](./rapport_TP1.pdf) |

## Contexte et objectifs

D’après le compte rendu, le TP s’articule en cinq parties :

1. préparer le serveur MariaDB sur la VM1 ;
2. ouvrir le service au réseau ;
3. connecter plusieurs clients à un seul serveur ;
4. ajouter un deuxième service, le serveur web Apache ;
5. observer les limites du modèle client-serveur.

Le document se termine par une synthèse comparative MariaDB / Apache et par la définition du modèle client-serveur, avec ses avantages et ses inconvénients.

## Architecture

| Machine | Adresse | Rôle |
| --- | --- | --- |
| Hôte Windows | `192.168.109.1` | Client 1 : navigateur, HeidiSQL / `mariadb` |
| VM1 (Linux) | `192.168.109.139` | Serveur : MariaDB (`3306`) et Apache (`80`) |
| VM2 (Linux) | `192.168.109.140` | Client 2 : `mariadb`, `curl` |

## Outils et technologies

| Catégorie | Éléments mentionnés dans le compte rendu |
| --- | --- |
| Serveurs | MariaDB (`mariadbd`), Apache `2.4.58` |
| Clients | `mariadb`, HeidiSQL, navigateur (`Mozilla/5.0 (Windows NT 10.0...)`), `curl/8.5.0` |
| Systèmes | Ubuntu (VM1, VM2), hôte Windows |
| Outils de diagnostic | `ss`, `tcpdump`, `systemctl`, `ufw`, `curl -v` |
| Fichiers et configurations cités | `/etc/mysql/mariadb.conf.d/50-server.cnf`, `tp1/scolarite.sql`, `tp1/index.html`, `/var/www/html/index.html`, journal `access.log` d’Apache |

## Déroulé, commandes et questions/réponses

### Partie 1 — Préparer le serveur (VM1)

```bash
sudo mariadb < tp1/scolarite.sql
sudo mariadb
USE scolarite;
SHOW TABLES;
SELECT COUNT(*) FROM etudiant;
```

#### Q1. Contenu de la base et schéma

Résultats relevés avec `SELECT COUNT(*) FROM etudiant;`, puis les requêtes équivalentes sur `ec` et `note` :

| Table | Nombre de lignes |
| --- | --- |
| `etudiant` | 30 |
| `ec` | 6 |
| `note` | 100 |

Schéma relationnel (colonnes déduites des requêtes de l’énoncé ; vérifiez avec `DESCRIBE <table>;` et ajoutez la 4e table affichée par `SHOW TABLES;`) :

```text
etudiant (matricule PK, prenom, nom, filiere, email)
ec       (code PK, intitule, credit)
note     (id PK, matricule FK, ec FK, valeur)
filiere  (code PK, libelle)
```

#### Q2. Client et serveur sont-ils le même programme ?

Non. Ce sont deux processus distincts, avec deux exécutables différents : le serveur `mariadbd` (`/usr/sbin/mariadbd`), démon lancé par systemd, tourne en permanence et appartient à l’utilisateur `mysql`, et le client `mariadb` est lancé par l’utilisateur `root`.

### Partie 2 — Ouvrir le service au réseau

Après modification dans `/etc/mysql/mariadb.conf.d/50-server.cnf` :

Reconnexion depuis Windows avec le compte `etudiant` (`[mot de passe masqué]`) :

```sql
SELECT USER(), CURRENT_USER();
```

Résultat relevé :

```text
+---------------------------+-------------------------+
| USER()                    | CURRENT_USER()          |
+---------------------------+-------------------------+
| etudiant1@192.168.109.1   | etudiant1@192.168.109.% |
+---------------------------+-------------------------+
```

#### Q3. Que signifie l’erreur 10061 ?

`10061` est le code Windows `WSAECONNREFUSED` : connexion refusée. Le SYN envoyé par Windows a été reçu par VM1, dont la pile TCP a répondu par un RST parce qu’aucun processus n’écoutait sur `192.168.109.139:3306`.

- VM1 était joignable : une réponse (le RST) est revenue. Si la machine avait été éteinte, injoignable ou filtrée par un pare-feu qui ignore les paquets, on aurait eu un délai d’attente dépassé (`10060`), pas un refus immédiat.
- Le serveur MariaDB fonctionnait : la sortie `ss` le prouve, `mariadbd` écoutait bien, mais uniquement sur `127.0.0.1`. Il était donc injoignable depuis le réseau tout en étant en bon état de marche.

#### Q4. 127.0.0.1 ou 0.0.0.0 ?

- `127.0.0.1` est l’adresse de bouclage (loopback) : le service n’accepte que les connexions venant de la machine elle-même ; aucun paquet venant du réseau n’est accepté.
- `0.0.0.0` signifie « toutes les adresses IPv4 de la machine » : le service accepte les connexions arrivant par n’importe quelle interface (ici `192.168.109.139`, mais aussi la carte NAT).

Ubuntu choisit `127.0.0.1` par défaut pour des raisons de sécurité : une base de données contient des données sensibles et la plupart des installations n’ont qu’une application locale qui l’utilise. On réduit ainsi la surface d’attaque, et l’administrateur doit ouvrir l’accès réseau de façon explicite et consciente.

#### Q5. USER() et CURRENT_USER()

- `USER()` : le compte tel que le client s’est présenté (nom donné + machine d’où il vient), ici `etudiant@192.168.109.1`.
- `CURRENT_USER()` : le compte enregistré sur le serveur qui a été retenu pour l’authentification et qui détermine les privilèges, ici `etudiant@192.168.109.%` (la règle correspondant à l’hôte).

Les deux peuvent différer car le serveur associe la connexion entrante à la ligne de la table des comptes qui correspond (nom + motif d’hôte), pas forcément à l’adresse exacte du client.

#### Q6. Connexion en root depuis le réseau

Aucun compte `root` n’est utilisable depuis Windows : sur Ubuntu, `root@localhost` s’authentifie par la socket Unix (`unix_socket`), c’est-à-dire par l’identité du système, et aucun `root@192.168.109.%` n’existe.

C’est une bonne chose : `root` a tous les droits (création de comptes, lecture et suppression de toutes les bases). Exposer ce compte sur le réseau le rendrait la cible n° 1 de toute attaque par force brute ou de tout vol de mot de passe. Les accès distants passent par des comptes limités, aux droits minimaux.

### Partie 3 — Plusieurs clients, un seul serveur

Commande lancée en même temps sur Windows (compte `etudiant`) et sur VM2 (compte `secretariat`, `[mot de passe masqué]`) :

```sql
SELECT SLEEP(20);
```

Observation sur VM1 pendant ces 20 secondes :

```bash
sudo ss -tn state established '( sport = :3306 )'
```

Le compte rendu signale une sortie d’erreur pour cette commande.

#### Q7. Adresses IP et ports sources

| Client | IP source | Port source (exemple) |
| --- | --- | --- |
| Windows (`etudiant`) | `192.168.109.1` | `55974` |
| VM2 (`secretariat`) | `192.168.109.140` | `41284` |

Les ports sources sont choisis par le système d’exploitation de chaque client (ports éphémères : `49152-65535` sous Windows, `32768-60999` par défaut sous Linux), librement parmi ceux qui sont libres. Ils diffèrent de `3306` parce que `3306` est le port d’écoute du serveur, fixe et connu à l’avance ; un client ne peut pas l’utiliser comme port source côté serveur. Les ports sources diffèrent d’un client à l’autre et d’une connexion à l’autre : le serveur peut ainsi distinguer chaque connexion grâce au quadruplet (IP source, port source, IP destination, port destination), et on voit bien un seul serveur sur un seul port, mais autant de connexions que de clients.

Test des droits :

```sql
-- Windows, compte etudiant1
UPDATE note SET valeur = 20 WHERE matricule = 'ESP260001' AND ec = 'ACSW';

-- VM2, compte secretariat
UPDATE note SET valeur = 12 WHERE matricule = 'ESP260001' AND ec = 'ACSW';
DELETE FROM note WHERE matricule = 'ESP260001';

-- Windows, compte etudiant1
SELECT valeur FROM note WHERE matricule = 'ESP260001' AND ec = 'ACSW';
```

#### Q8. Messages d’erreur et droits

Les erreurs relevées sont de type `ERROR 1142 (42000)` : « command denied to user ». Les droits sont enregistrés côté serveur, dans les tables système de la base `mysql` (`mysql.user`, `mysql.db`, `mysql.tables_priv`...), et c’est le serveur qui les vérifie à chaque requête, pas le client.

Conclusion pour la sécurité : on ne peut pas faire confiance au client. Un client modifié, ou n’importe quel autre programme parlant le protocole, reste soumis aux mêmes règles puisqu’elles sont appliquées au seul endroit contrôlé par l’administrateur. La sécurité doit donc être définie et appliquée au niveau du serveur et non dans l’interface cliente.

#### Q9. Cohérence des données

Oui, le client Windows voit immédiatement la valeur `12` écrite depuis VM2. Les clients ne possèdent aucune copie des données : chaque `SELECT` interroge le serveur, qui est la source unique de vérité. Dès que le `UPDATE` du secrétariat est validé, toute lecture ultérieure, depuis n’importe quel client, retourne la valeur à jour. Il n’y a pas de synchronisation à faire entre clients, puisqu’ils ne communiquent jamais entre eux ; le serveur sérialise les accès concurrents.

### Partie 4 — Un deuxième service : le serveur web

Installation sur VM1 :

```bash
sudo apt install apache2
```

Puis copie de `tp1/index.html` dans `/var/www/html/`, et ouverture du port `80` si `ufw` est actif.

Accès depuis Windows : `http://192.168.109.139/`.  
Accès depuis VM2 : `curl -v`.  
Puis lecture du journal.

#### Q10. Requête et réponse dans curl -v

Requête du client (lignes précédées de `>`) :

- 1re ligne, la ligne de requête : `GET / HTTP/1.1` = méthode (ce que l’on veut faire), ressource demandée (`/`, la racine du site) et version du protocole.
- En-têtes : `Host: 192.168.109.139` (serveur visé, indispensable en HTTP/1.1 pour les sites virtuels) et `User-Agent: curl/8.5.0` (logiciel client) ; curl ajoute aussi `Accept: */*`.

Réponse du serveur (lignes précédées de `<`) :

- 1re ligne, la ligne d’état : `HTTP/1.1 200 OK` = version, code de statut (`200` = succès) et message.
- En-têtes : `Server: Apache/2.4.58` et `Content-Type: text/html` ; on trouve en général aussi `Date`, `Content-Length`, `Last-Modified`, `ETag`.
- Après une ligne vide vient le corps : le contenu de `index.html`.

#### Q11. Informations enregistrées dans access.log

Pour chaque requête, Apache enregistre : l’adresse IP du client, la date et l’heure, la requête complète, le code de statut (`200`, `404`...), la taille de la réponse en octets, le référent et l’identification du client (User-Agent).

On distingue ainsi nos deux clients : `192.168.109.1` avec un User-Agent de navigateur (`Mozilla/5.0 (Windows NT 10.0...)`), accompagné d’une requête automatique `/favicon.ico`, et `192.168.109.140` avec `curl/8.5.0`. Le serveur ne connaît le client que par ce que celui-ci lui révèle : IP et en-têtes.

#### Q12. Deux serveurs sur VM1 : qui reçoit quoi ?

Le système distingue les programmes grâce au numéro de port de destination du segment TCP. Chaque serveur a demandé au noyau d’écouter sur un port (socket + `bind` + `listen`) : le noyau tient une table des sockets à l’écoute et remet le segment au processus associé : port `80` → `apache2`, port `3306` → `mariadbd`. Pour les connexions établies, le noyau utilise le quadruplet (IP source, port source, IP destination, port destination).

Un client doit connaître à l’avance l’adresse ET le port parce que le serveur ne vient jamais le chercher : l’adresse IP sert à acheminer les paquets jusqu’à la bonne machine et le port à désigner le bon service sur cette machine. Il n’existe pas de découverte automatique ; c’est une convention (ports « bien connus » : `80` pour HTTP, `3306` pour MariaDB).

#### Q13. Connexion persistante ou requêtes indépendantes

| Mode | Avantage | Inconvénient |
| --- | --- | --- |
| MariaDB (connexion persistante) | On s’authentifie une seule fois ; le serveur garde l’état de la session (base courante, transaction en cours, variables) ; pas de coût de connexion répété. | Chaque connexion ouverte consomme des ressources serveur (mémoire, thread) même inactive ; le nombre de clients est limité (`max_connections`) ; une coupure réseau casse la session. |
| HTTP (requête-réponse) | Le serveur est sans état : il libère ses ressources après chaque réponse, il peut servir beaucoup de clients et il est facile de répartir la charge. | Le serveur ne reconnaît pas l’utilisateur d’une requête à l’autre ; il faut le ré-identifier à chaque fois ; surcoût d’en-têtes et de connexions (atténué par keep-alive). |

Comment un site « se souvient » d’un utilisateur ? Après l’authentification, le serveur crée une session et envoie au navigateur un identifiant (cookie de session, ou jeton type JWT). Le navigateur le renvoie dans chaque requête suivante (en-tête `Cookie`), et le serveur retrouve les données de la session. L’état est donc ajouté au-dessus d’un protocole sans état.

### Partie 5 — Les limites du modèle

#### 5.1 Arrêt du serveur

```bash
sudo systemctl stop mariadb
```

#### 5.2 Capture du trafic

```bash
sudo tcpdump -i enp0s8 -A -s0 'tcp port 3306'
```

Requête lancée depuis Windows pendant la capture :

```sql
SELECT nom, valeur FROM note JOIN etudiant USING(matricule) WHERE matricule = 'ESP260011';
```

#### Q14. Serveur arrêté

Chaque client reçoit le même type de message : `ERROR 2002 (HY000): Can't connect to server on '192.168.109.139'` (code `10061` sous Windows, `111` sous Linux : connexion refusée). Tous les clients (2 sur 2) sont affectés simultanément, et un client déjà connecté verrait sa session coupée (`ERROR 2013 : Lost connection` ou `2006 : server has gone away`). Le serveur est la seule source de données : s’il tombe, plus aucun client ne peut travailler, même si les clients et le réseau sont en parfait état.

Ce défaut d’architecture s’appelle le point unique de défaillance (Single Point of Failure, SPOF). On le réduit par la redondance : réplication maître/esclave, cluster (par exemple Galera), adresse IP virtuelle avec basculement automatique, sauvegardes.

#### Q15. Ce qui circule sur le réseau

Dans la capture on retrouve en clair :

- le nom du compte (`etudiant1`) et le nom de la base (`scolarite`) ;
- le texte exact de la requête SQL ;
- les données de la réponse (noms, notes).

Le mot de passe n’apparaît pas en clair : MariaDB utilise un échange défi-réponse (`mysql_native_password`) où le client envoie une valeur calculée à partir du mot de passe et d’un défi aléatoire du serveur. Mais ce n’est pas une vraie protection : tout le reste est lisible, et cette valeur peut être rejouée ou attaquée hors ligne par force brute.

Solution : chiffrer les échanges avec TLS. Sur le serveur, activer SSL/TLS dans `50-server.cnf` (`ssl-cert`, `ssl-key`, `ssl-ca`), forcer le chiffrement (`require_secure_transport = ON` ou `REQUIRE SSL` sur les comptes) et connecter les clients avec `--ssl`. Alternatives : tunnel SSH (`ssh -L 3306:localhost:3306`) ou VPN. Pour le web, l’équivalent est HTTPS à la place de HTTP.

### Synthèse MariaDB / Apache

| Caractéristique | MariaDB | Apache (HTTP) |
| --- | --- | --- |
| Machine et port du serveur | VM1 (`192.168.109.139`), port `3306/TCP` | VM1 (`192.168.109.139`), port `80/TCP` |
| Logiciel client utilisé | `mariadb` (Windows et VM2), HeidiSQL | Navigateur (Windows), `curl` (VM2) |
| Qui prend l’initiative de la connexion ? | Le client | Le client |
| Durée de vie d’une connexion | Persistante : toute la session | Courte : une requête-réponse (ou quelques-unes avec keep-alive) |
| Où sont vérifiés les droits ? | Côté serveur (comptes et privilèges dans la base `mysql`) | Côté serveur (configuration Apache, droits sur les fichiers) |
| Effet d’un arrêt du serveur | Tous les clients perdent l’accès : erreur `2002` / sessions coupées | Tous les clients perdent l’accès : connexion refusée ; MariaDB continue de fonctionner (services indépendants) |

#### Q16. Définition du modèle client-serveur

Le modèle client-serveur est une architecture de communication à deux rôles distincts. Le serveur est un programme qui s’exécute en permanence sur une machine connue, écoute sur un port connu (`3306` pour MariaDB, `80` pour Apache) et attend des demandes. Les clients (`mariadb`, navigateur, `curl`) prennent l’initiative : ils ouvrent une connexion vers l’adresse et le port du serveur, envoient une requête puis attendent la réponse. Nos observations montrent que le serveur détient et centralise les données (la modification faite depuis VM2 est vue aussitôt par Windows) et décide des droits (les erreurs `1142` viennent du serveur, pas du client). Il peut servir plusieurs clients simultanés (deux connexions dans `PROCESSLIST`), chacun identifié par son couple IP/port source, tandis que les clients ne communiquent jamais entre eux. Plusieurs serveurs peuvent cohabiter sur une même machine grâce aux numéros de port. Les protocoles diffèrent selon les besoins : session persistante pour MariaDB, requêtes indépendantes sans état pour HTTP.

Deux avantages :

1. Centralisation des données, de la sécurité et de l’administration : une seule copie à jour et des droits appliqués au même endroit, donc cohérence et contrôle.
2. Partage et évolutivité : de nombreux clients, de types différents (Windows, Linux, navigateur), utilisent le même service ; on peut ajouter des clients sans toucher au serveur.

Deux inconvénients :

1. Point unique de défaillance : si le serveur s’arrête (partie 5.1), tous les clients sont bloqués ; il devient aussi un goulot d’étranglement quand la charge augmente.
2. Trafic en clair et exposition : les échanges peuvent être lus sur le réseau et le serveur, joignable par tous, est une cible ; il faut chiffrer (TLS) et filtrer.

## Résultats et tests documentés

- Import de `scolarite.sql` et comptage : `etudiant` = 30, `ec` = 6, `note` = 100 ; une quatrième table `filiere` est affichée par `SHOW TABLES;`.
- Connexion distante depuis Windows après ouverture du service, avec relevé de `USER()` / `CURRENT_USER()`.
- Deux clients simultanés (`SELECT SLEEP(20);`) observés depuis VM1 ; la commande `ss` est documentée avec une sortie d’erreur signalée.
- Tests de droits : `UPDATE` / `DELETE` selon les comptes, erreurs `ERROR 1142 (42000)`, puis lecture de la valeur `12` depuis Windows.
- Service web Apache : requête `GET / HTTP/1.1`, réponse `HTTP/1.1 200 OK`, et distinction des clients dans `access.log`.
- Arrêt de MariaDB : erreurs `2002` / `10061` / `111` et mise en évidence du SPOF.
- Capture `tcpdump` : requêtes SQL, données de réponse et compte visibles en clair ; le mot de passe n’apparaît pas en clair.



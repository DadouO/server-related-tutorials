# Monter un compose Nginx, PHP, SQL & Admin

```bash
> cd ~/../c/Users/Patolash/Documents/_dev/server-related-tutorials/01-docker/04-my-tests/03-compose-nginx-php-sql
> docker-compose up
```

Liens: [Front sur 8080](http://localhost:8080/), [Adminer sur 8081](http://localhost:8081/) avec Serveur: *db_tartopaum*, Utilisateur: *bob*, Pass: *bobspw*

Sources :

- *Fichier docker-compose.yml*
- *Dossier sources /src*
- *Dossier configs /config*
  - Contient également les configurations par défaut des deux images

On repart du compose nginx + php :)

## Principales commandes

Lancement en tant que service via [swarm](https://docs.docker.com/get-started/part4/)

```bash
> docker stack deploy -c docker-compose.yml swarm-php-sql
```

Arrêt du service

```bash
> docker stack rm swarm-php-sql
```

Notes

- Possibilité de lancer via `docker-compose up` pour avoir de meilleurs logs des différents containers.

---

Rentrer dans un container monté pour vérifier contenu ou conf :

```bash
# Récupérer le nom du container (Selec + clic droit (copier), puis clic droit (coller))
> docker container ls

# php-fpm
> docker exec -it NOM_CONTAINER bash

# **alpine ne possède pas bash (nginx)**
> docker exec -it NOM_CONTAINER /bin/ash
```

## Choix de la DB et de l'admin

Basiquement le choix se fait entre [MySQL](https://hub.docker.com/_/mysql) et [MariaDB](https://hub.docker.com/_/mariadb).

Après avoir lu [quelques comparatifs](https://www.eversql.com/mariadb-vs-mysql/) et [benchmarks](http://dimitrik.free.fr/blog/archives/2018/04/mysql-performance-80-and-utf8-impact.html), les différences restent mineures.

Je vais partir sur MySQL parce que cela semble légèrement plus performant, que j'en ai plus l'habitude & qu'il peut y avoir de légères variations de syntaxe via mariaDB ?

Concernant le choix de l'admin, les deux recommandent [adminer](https://hub.docker.com/_/adminer) (ex: phpmyadmin), je vais partir la dessus pour mes tests.

## Mise en place de MySQL

Lecture de la doc et récupération des bonnes pratiques :

1. Analyse de la configuration de l'image
2. Initialisation / Récupération de la base de données
   - Variables d'environnements + volumes
3. Sécurité : Utilisation de [docker secrets](https://docs.docker.com/compose/compose-file/#secrets)

### Ajout de MySQL au docker-compose

```yaml
  db_tartopaum:
    # Sert à faire le lien avec adminer sans network
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: pizza
    # latest when writing
    image: 'mysql:8.0.18'
    restart: always
```

Ok, mais quelques warnings de configuration à régler

## Ajout de adminer

... pour pouvoir mettre les mains dedans.

```yaml
adminer:
    # latest when writing
    image: 'adminer:4.7.5'
    ports:
      # Running on port 8081 as 8080 is already used by web
      - 8081:8080
    restart: always
```

Ok, test sur [http://localhost:8081/](http://localhost:8081/), on peut se connecter via root/pizza.

**Attention !**, sur l'interface d'administration d'adminer, le champ serveur doit être renseigné avec l'identifiant du conteneur indiqué dans le docker-compose (ici `db_tartopaum`). C'est ce qui permet d'assurer la connexion sans réseau, ce qui me parait bizarre may bon plugin ✨..

---

Essai adminer via fast-cgi > echec

Mais tourne sur son propre serveur. La doc recommande de le passer sur le php fast-cgi.

// Multiples essais [1](https://github.com/TimWolla/docker-adminer/issues/35), [2](https://forum.vestacp.com/viewtopic.php?t=14386&start=10) > KO, doc incomplète

Retour sur l'image adminer de base, je pense que adminer fast-cgi dispose de son propre serveur, je n'arrive pas a lui dire d'utiliser l'autre ou de le faire tourner sans conflit...

Essai de l'ouvrir sur `/adminer` plutôt que sur `localhost:8081` via un [proxy](https://gist.github.com/soheilhy/8b94347ff8336d971ad0) et son propre fichier de conf > KO

Bref, on avance..

---

### Changement du thème

Ajout de variable d'environnement pour choisir parmis la [liste des thèmes](https://github.com/vrana/adminer/tree/master/designs)

```yaml
  adminer:
    environment:
      ADMINER_DESIGN: 'pappu687'
```

Liste des thèmes bieng :

- Clairs : esterka, lucas-sandery, mvt, nette, ng9, pappu687, pepa-linha
- Sombres : galkaev, mancave

## Configuration MySQL

On rentre dans le conteneur mysql et on suit la doc

```bash
> docker exec -it NOM_CONTAINER_MYSQL bash
```

Copie des 3 fichiers présents dans `HOST/config/mysql8.0.18/_defaults/etc/mysql/` mais rien d'extravagant.

Possibilité de rajouter/surcharger dans `/etc/mysql/conf.d/wtv.cnf`.

### Ajout de l'encodage par défaut

Utilisation de [l'encodage recommandé](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) en suivant les recos [SO](https://stackoverflow.com/a/3513812/12026487).

Création d'un fichier de conf personnalisé `my-conf.cnf` et surcharge de la configuration dans le docker-compose (bind).

Test > Ok :)

## Base de données

### Initialisation

Utilisation des variables d'environnement dans le docker-compose, ajout de la bdd

```yaml
  db_tartopaum:
    environment:
      MYSQL_DATABASE: my_awesome_db
```

Ok, la DB est créée par défaut lors de la première connexion.

Ajout d'un utilisateur

```yaml
  db_tartopaum:
    environment:
      MYSQL_USER: bob
      MYSQL_PASSWORD: bobspw
```

Accès ok, privilèges uniquement pour la db créé à la volée.

### Pré-remplissage de la base

Possibilité de fournir du sql éxécuté automatiquement depuis `/docker-entrypoint-initdb.d` (cf. doc) : création d'un script `HOST/init/mysql/docker-entrypoint-initdb.d/my_awesome_table.sql`

```sql
/* Create table with 2 fields */
CREATE TABLE `my_awesome_table` (
  `id_my_awesome_table` int(10) unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `test_my_awesome_table` varchar(100) NOT NULL
);

/* Add one glorious entry */
INSERT INTO `my_awesome_table` (`test_my_awesome_table`)
VALUES ('hey je viens de la bdd');
```

..  et ajout via docker-compose > bind.

- Test au montage > Ok.
- Test ajout nouvel élément en bdd, démontage et remontage (voir si pas d'erreur de persistance..)
  - Persistance, mais d'après la doc, MySQL se crée automatiquement un volume docker.

### Persistance des données (volume)

Création d'un volume **nommé**, c'est plus propre

```yaml
  db_tartopaum:
    # [...]
    volumes:
      # [...]
      - type: volume
        source: nginx-php-sql_dbdata
        target: /var/lib/mysql

# [...]

volumes:
  nginx-php-sql_dbdata:
```

**Note** : En cas de tests répétés, il se peut que certains conteneurs soient encore en place avec des volumes anonymes monté sur `/var/lib/mysql`, et que docker-compose refuse le montage vers le volume nommé "Duplicate mount point".

~~Afin de corriger cela, on peut utiliser `docker-compose down -v` afin de cleaner les compositions en place [cf. SO](https://github.com/docker-library/mysql/issues/414#issuecomment-429129244).~~

**^ Attention ! Cela détruit les données ! Développement uniquement** / Plus tard > back up & restaurations de données, cf. doc mysql

- Bien attendre que le docker-compose soit stoppé/kill (gracefull), et a priori plus de problèmes
  - Ou utiliser plutôt `docker container prune` qui ne touche pas aux volumes

## Sécurité / Docker secrets / KO AVEC PLUGIN DE MERDE mysql connect

Bonnes pratiques + s'entrainer un peu, un [exemple](https://github.com/docker-library/mysql/issues/414).

Problèmes de cache bizarres via [docker-compose & les variables d'environnement](https://github.com/docker/compose/issues/6889) (je suis en version 1.22.0)

---

Maj docker-compose

- Verif version `pip show docker-compose` // 1.25.0
- Ne veut pas s'update, mais genre pas du tout `pip install --user docker-compose` / `pip install --upgrade etc.` /
- Essai sans cache PM
- Maj package manager : `pip install -U pip`
  - KO > [wtv](https://github.com/pypa/pip/issues/5221#issuecomment-382069604)
  - de retour
- Essai [sans cache](https://stackoverflow.com/questions/9510474/removing-pips-cache) `pip install --user --no-cache-dir docker-compose`
  - Tout re-dl mais nope
- Essai sans `--user`
  - Ok dans les logs, mais `docker-compose version` OSEF
- Désinstaller puis reinstaller
  - `pip uninstall docker-compose`

```bash
# Successfully uninstalled docker-compose-1.25.0
❯ docker-compose version
# docker-compose version 1.22.0, build f46880fe
```

wat

Récupération de la [méthode d'installation](https://stackoverflow.com/questions/36685980/docker-is-installed-but-docker-compose-is-not-why/36689427#36689427) > curl & mv -_-

[Dernière version à date](https://github.com/docker/compose/releases)

```bash
> sudo curl -L "https://github.com/docker/compose/releases/download/1.25.1-rc1/docker-compose-$(uname -s)-$(uname -m)"  -o /usr/local/bin/docker-compose
> sudo mv /usr/local/bin/docker-compose /usr/bin/docker-compose
> sudo chmod +x /usr/bin/docker-compose
```

OK

---

Forcément c'étay pas ça. A l'instinct cette ligne alakon `https://mariadb.com/kb/en/authentication-plugin-mysql_native_password/` > Le problème semble venir de la.

FUCK THIS, 2h de perdues, le pass change quand ça le chante

EDIT : essai avec swarm (`docker stack...`) ok une fois, mais en fait non, ça pue bien la merde.

// TODO #cancer

### Essai résolutions warnings

#### Warning symbolic links

[github issue](https://github.com/docker-library/mysql/issues/591) > reco changement conf de base

Surcharge de `my.cnf` via bind > commenter le bousin.

#### Warning CA certificate ca.pem is self signed

[wtv](https://i-mscp.net/thread/16232-ca-certificate-ca-pem-is-self-signed-reported-by-mysql/)

Osef

#### Insecure configuration for --pid-file

`Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.`

Rien sur le net, pas le courage

## Requête PHP alaakon

Réutilisation d'un vieux script PDO > could not find driver.

Besoin d'installer PDO > [github issue](https://github.com/docker-library/docs/issues/723#issuecomment-405196521) & [Another one](https://github.com/docker-library/php/issues/62)

Création d'une image dédiée `DockerfilePhpWithPdo`, remplacement dans docker-compose.

Attention ! Lors de la connexion à la bdd via PHP

- Bien spécifier l'hôte de la connexion PDO comme docker : `$host = "db_tartopaum";`
  - (Pas oublier login/pass.. Et adapter nom bdd, table, champ)
- PHP et SQL doivent faire partie du même network
  - **Un network est créé par défaut entre MySQL & adminer, et ce dernier se fait oblitérer si définition d'un réseau explicite pour mysql**
    - Solutions :
      - ~~Mettre tlm sur le même réseau~~ // sécu
      - Créer un réseau explicite dédié entre sql & adminer

Yay, tout good

## docker stack deploy Fix

docker-compose.yml > build est KO avec docker stack (swarm) qui n'accepte que les images.

Solution : build l'image de PHP avec PDO, la [push sur docker hub](https://hub.docker.com/repository/docker/youpiwaza/php-with-pdo) & la réutiliser.

1. [Dockerhub](https://docs.docker.com/docker-hub/repos/) > Log in > Création d'un repo
2. `docker build -t <hub-user>/<repo-name>[:<tag>]`
3. [CLI login](https://docs.docker.com/engine/reference/commandline/login/) / `docker login login -u=<hub-user>`
4. `docker push <hub-user>/<repo-name>[:<tag>]`

```bash
# Build
# /!\ Ne pas oublier le dernier '.' pour le contexte
> docker build \
  -f DockerfilePhpWithPdo \
  -t youpiwaza/php-with-pdo:1.0.0 \
.

# Push
> ~~docker push youpiwaza/php-with-pdo:1.0.0~~
# denied: requested access to the resource is denied

# Login
> docker login -u=youpiwaza
# Password ? > yay

# Push
> docker push youpiwaza/php-with-pdo:1.0.0

```

Maj du docker-compose.yml

```yaml
  phpfpm:
    image: 'youpiwaza/php-with-pdo:1.0.0'
```

Test > *Purrrfect*

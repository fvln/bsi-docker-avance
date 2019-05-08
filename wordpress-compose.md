# Démarrage automatisé des services avec Docker compose

## Pré-requis

Docker-compose est un programme distribué à part sur le site de Docker. Se reporter à la procédure décrite ici : https://docs.docker.com/compose/install/

## Fonctionnement

Docker-compose permet d'exécuter plusieurs services conjointement (sous forme de conteneurs), tout en gérant entre autres :
* la configuration des réseaux
* les montages de volumes
* les dépendances entre services
* les redémarrages automatiques en cas de crash
* la centralisation des logs

En d'autres termes, nous pouvons décrire en un seul fichier au format YAML (https://fr.wikipedia.org/wiki/YAML) toutes les commandes docker permettant de démarrer le service Wordpress. Ce fichier se nomme par défaut `docker-compose.yml`.

## Rédaction du fichier docker-compose.yml

Nous allons construire ce fichier de manière itérative pour faire fonctionner tous les conteneurs ensemble.

### Service de base de données

Si l'on se réfère [aux travaux précédents](https://github.com/fvln/bsi-docker-avance/blob/master/wordpress-docker.md#utilisation-dune-image-mariadb), ce service : 
* est basé sur une image publique _mariadb_
* ne dépend d'aucun autre service
* s'appuie sur des variabbles d'environnement 

Notre fichier `docker-compose.yml` s'écrit donc ainsi ([manuel de référence](https://docs.docker.com/compose/compose-file/)):

```yaml
version: '3.7'
services:
  mariadb:
    image: "mariadb"
    environment:
      MYSQL_ROOT_PASSWORD: my-secret-pw
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    networks:
      - webserver

networks:
  webserver:
    driver: bridge
    internal: true
```

On peut démarrer facilement ce premier service avec son réseau (éventuellement en ajoutant l'option `-d` pour détacher l'affichage des logs de la console) :

```bash
user@debian:~/BSI$ sudo docker-compose up

Creating network "bsi_webserver" with driver "bridge"
Creating bsi_mariadb_1 ... done
Attaching to bsi_mariadb_1
mariadb_1  | Initializing database
mariadb_1  | 
mariadb_1  | 
mariadb_1  | PLEASE REMEMBER TO SET A PASSWORD FOR THE MariaDB root USER !
(...)
```

On voit que l'image mariadb a bien été instanciée sous la forme d'un conteneur nommé `bsi` (nom du répertoire) `_mariadb` (nom de l'image) `_1` (numéro de l'instance) :

```bash
user@debian:~/BSI$ sudo docker ps
[sudo] Mot de passe de user : 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
a6cd758cedfe        mariadb             "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes                            bsi_mariadb_1
```
 
Idem pour le réseau `bsi_webserver`, auquel ce conteneur est attaché :

```bash
user@debian:~/BSI$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b28501f5b59a        bridge              bridge              local
f7025f09e910        bsi_webserver       bridge              local
b9e97f0ebc32        host                host                local
556cbf1ffcfd        none                null                local
```

### Service php-fpm

Cette fois, nous allons utiliser une image non pas disponible sur docker-hub mais construite localement grâce
[à ce fichier](https://github.com/fvln/bsi-docker-avance/blob/master/bsi-php/Dockerfile). Voici le nouveau service
ajouté au fichier `docker-compose.yml` :

```yaml
  php:
    build:
      context: ./bsi-php
    volumes:
      - type: bind
        source: ./wordpress
        target: /var/www/html
    networks:
      - webserver
```

Docker-compose déclenche automatiquement le `build` de cette image au démarrage, si c'est nécessaire :

```bash
user@debian:~/BSI$ sudo docker-compose up
Creating network "bsi_webserver" with driver "bridge"
Building php
Step 1/6 : FROM debian:stretch-slim
 ---> c08899734c03
Step 2/6 : RUN apt update && apt install -y php-fpm php-mysql
 ---> Using cache
 ---> a3ea7f4e27b3
Step 3/6 : RUN sed -i -e 's#listen = /run/php/php7.0-fpm.sock#listen = 9000#' /etc/php/7.0/fpm/pool.d/www.conf
 ---> Using cache
 ---> e37d4c1f5ac0
Step 4/6 : RUN echo 'catch_workers_output = yes' >> /etc/php/7.0/fpm/pool.d/www.conf
 ---> Using cache
 ---> 8f7e09ad1dec
Step 5/6 : RUN mkdir /run/php
 ---> Using cache
 ---> cea85e235809
Step 6/6 : CMD /usr/sbin/php-fpm7.0 --nodaemonize
 ---> Using cache
 ---> 4da12abccb0a

Successfully built 4da12abccb0a
Successfully tagged bsi_php:latest
WARNING: Image for service php was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating bsi_mariadb_1 ... done
Creating bsi_php_1     ... done
```

### Service nginx

De manière similaire, on ajoute un nouveau service pour démarrer nginx, en exposant le port 80 du container sur le 8000 de l'hôte :

```yaml
  nginx:
    build:
      context: ./bsi-nginx
    volumes:
      - type: bind
        source: ./wordpress
        target: /var/www/html
    networks:
      - webserver
      - public
    ports:
      - "8000:80"

networks:
  webserver:
    driver: bridge
    internal: true
  public: ### Nouveau réseau connecté à l'hôte
    driver: bridge
```

On voit que le service nginx est à la fois connecté au réseau interne _webserver_ et au réseau _public_ qui permet d'exposer son port
80 via le port 8000 de l'hôte.




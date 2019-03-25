---
title: "Usine logicielle avec Gitlab runner, Nexus, Sonar - Partie 1"
tags: [ "gitlab-runner", "ci/cd", "docker", "nexus", "sonar" ]
date: 2019-02-23T22:27:24+02:00
draft: false
---

# Introduction {#intro}

La vérification de la qualité de votre code, l'exécution de toutes sortes de tests et le maintien du contrôle de version peut devenir une tâche difficile à gérer.

Et nous, les développeurs, en tant que créatures paresseuses, avons tendance à automatiser toutes ces tâches à répétition.

Un service d'intégration continue (CI) une fois configuré avec notre environnement d'application peut lancer toutes ces tâches automatiquement chaque fois que le code est modifié dans nos référentiels.

Mieux encore, nous verrons comment faire de la livraison continue !

Cet article donne les détails d'une mise en place d'un ensemble d'outils permettant d'avoir un registre privé d'artefacts (Java, npm ou encore docker), de l'intégration continue, de la livraison continue et du contrôle de qualité automatisé.

# BigPicture {#big-picture}

Voici l'ensemble des outils qui seront utilisés :

- Gitlab (ici dans sa version cloud, https://gitlab.com) mais une version on-premise peut être utilisé
- [Gitlab-runner](https://docs.gitlab.com/runner/)
- [Nexus Repository OSS](https://fr.sonatype.com/nexus-repository-oss)
- [SonarQube](https://www.sonarqube.org/)
- [Docker](https://www.docker.com)
- [Traefik](https://traefik.io/)
- [Let's encrypt](https://letsencrypt.org/)
- [Vuejs](https://vuejs.org/), [Buffalo](https://gobuffalo.io/fr) et [PostgreSQL](https://www.postgresql.org/) pour l'application d'exemple
- De plusieurs VMs ou machine physique (ici quelques machines chez [Scaleway](https://www.scaleway.com/))

Petit schéma qui explique rapidement la cinématique de notre automatisation :

![Cycle](/images/GitlabRunner-Page-1.png)

> Icons made by [Freepik](http://www.freepik.com/) from [www.flaticon.com](https://www.flaticon.com/) is licensed by [CC 3.0 BY](http://creativecommons.org/licenses/by/3.0/)


Donc dans les grandes lignes :

- Les développeurs poussent leur code dans un référentiel de code, ici en l'occurrence [GitLab](https://gitlab.com)
- Suivant la branche utilisée, une pipeline est déclenchée. Une pipeline est un ensemble d'étapes (*stages*) et de tâches (*jobs*) définies dans un fichier **.gitlab-ci.yml**
- Les dépendances sont installées, les tests exécutés, au besoin le livrable est stocké dans un référentiel (ici Nexus)
- Si tout c'est bien déroulé, on exécute une livraison sur le serveur applicatif.

Voici un schéma de l'infrastructure qui sera mis en place :

![infrastructure](/images/GitlabRunner-Page-2.png)

> Icons made by [Freepik](http://www.freepik.com/) from [www.flaticon.com](https://www.flaticon.com/) is licensed by [CC 3.0 BY](http://creativecommons.org/licenses/by/3.0/)

# Installation de notre machine "DevTools"

Entrons dans le vif du sujet maintenant !

## VM DevTools

Tout d'abord il nous faut donc une machine pour préparer notre usine logicielle !

Pour notre tuto du jour, j'ai fait le choix d'une VM de type **START1-L (8 vCpu, 8Go Ram, 50Go LSSD)** chez [Scaleway](https://www.scaleway.com/pricing/#anchor_starter).
C'est pas cher (moins de 20 € TTC/mois), c'est français et ça marche bien, mais libre à vous de choisir un autre hébergeur.

Nexus et Sonarqube sont plutôt gourmand en mémoire.

Prenez une Ubuntu Bionic, juste parce que c'est une distribution bien répandu, bien suivie et vous trouverez de [l'aide facilement](https://stackoverflow.com/questions/tagged/ubuntu).

Je vous conseille fortement de créer un groupe de sécurité et de l'affecter à votre nouveau serveur :

![Security Group Scaleway](/images/SecurityGroupScaleway1.png)

Par défaut je **refuse** tout le trafic entrant et **accepte** tout le trafic sortant.

Puis j'autorise uniquement le trafic entrant sur le port 80 et 443 (HTTP et HTTPS) provenant de n'importe quel IP.

Enfin j'autorise le trafic entrant sur tout les ports mais uniquement sur l'IP correspondant à ma connexion internet.

## Installation de docker et de docker-compose

J'ai choisi d'utiliser docker car c'est pratique pour le déploiement et on trouve les images officiels pour Nexus et SonarQube.

J'ai aussi fait le choix d'utiliser docker-compose pour ne pas à avoir à gérer la complexité d'un orchestrateur de type Swarm ou Kubernetes par exemple.

Voici les commandes shell à lancer sur votre VM toute fraîche :

{{< highlight console >}}
apt update && apt upgrade
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
{{< / highlight >}}

Nous avons donc maintenant docker qui est installé et opérationnel.

## Installation de Traefik

Maintenant nous allons installer Traefik qui nous permettra d'avoir un reverse proxy et la gestion automatique pour nos certificat SSL via Let's encrypt.

Référence https://docs.traefik.io/user-guide/docker-and-lets-encrypt/

{{< highlight console >}}
mkdir -p /opt/traefik
touch /opt/traefik/docker-compose.yml
touch /opt/traefik/acme.json && chmod 600 /opt/traefik/acme.json
touch /opt/traefik/traefik.toml
{{< / highlight >}}


**/opt/traefik/docker-compose.yml :**
{{< highlight dockerfile >}}
version: '2'

services:
  traefik:
    image: traefik:1.5.4
    restart: always
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    networks:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/traefik/traefik.toml:/traefik.toml
      - /opt/traefik/acme.json:/acme.json
    container_name: traefik

networks:
  web:
    external: true
{{< / highlight >}}

Ici le docker-compose est très simple, on fait le lien avec les fichiers de configuration sur notre machine hôte.
Enfin se fichier permet de créer un réseau nommé *web* qui nous permettra de faire sortir notre trafic web.

**/opt/traefik/traefik.toml :**
{{< highlight toml >}}
debug = false

logLevel = "ERROR"
defaultEntryPoints = ["https","http"]

# WEB interface of Traefik - it will show web page with overview of frontend and backend configurations
[api]
  entryPoint = "traefik"
  dashboard = true
  address = ":8080"

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
  [entryPoints.https.tls]

[retry]

[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "scw-par1-devtools.IP_VM_SCALEWAY.nip.io"
watch = true
exposedByDefault = false

[acme]
email = "email@example.com"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
[acme.httpChallenge]
entryPoint = "http"
{{< / highlight >}}

Le fichier de configuration est simple aussi :

- on redirige tout trafic entrant http vers https
- on définie un domaine principal via la variable *domain*. Vous pouvez utiliser un domaine que vous avez fait pointer sur votre IP publique scaleway ou alors utiliser le service wildcard DNS de http://nip.io/
- n'oubliez pas renseigner votre adresse *email* dans la variable prévue à cet effet

Allez c'est partie, on lance le docker traefik ! :

{{< highlight console >}}
cd /opt/traefik/
docker network create web
docker-compose up -d
{{< / highlight >}}

Si tout se passe bien, le containeur est UP :

{{< highlight console >}}
root@scw-par1-devtools:/opt/traefik# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                                              NAMES
ecd1c9413f2f        traefik:1.5.4       "/traefik"          23 hours ago        Up 23 hours         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:8080->8080/tcp   traefik
{{< / highlight >}}

Allez faire un tour sur le domaine que vous avez configuré, vous devriez avoir un joli **404 page not found** provenant de traefik.
Mais le plus intéressant sera de jeter un oeil sur le port 8080 (qui n'est ouvert que sur les IPs autorisées du groupe de sécurité bien-sur !).

## Installation de Nexus

Faite comme pour traefik, créer un répertoire /opt/nexus3/ et y mettre le fichier docker-compose.yml suivant :

**/opt/nexus3/docker-compose.yml**
{{< highlight dockerfile >}}
version: "3"

services:
  nexus:
    image: sonatype/nexus3
    restart: always
    volumes:
      - "nexus-data:/nexus-data"
    networks:
      - web
      - default
    expose:
      - "8081"
    labels:
      - "traefik.backend=nexus3"
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:nexus3.scw-par1-devtools.IP_VM_SCALEWAY.nip.io"
      - "traefik.enable=true"
      - "traefik.port=8081"

volumes:
  nexus-data: {}

networks:
  web:
    external: true
{{< / highlight >}}

Pensez à remplacer la valeur de *traefik.frontend.rule* avec votre domaine ou IP scaleway.

{{< highlight console >}}
cd /opt/nexus3/
docker-compose up -d
{{< / highlight >}}

A ce stade, vous devez avoir accès à une interface web via l'url précisé dans la variable *traefik.frontend.rule*.
Par défaut, le login est **admin** et le mot de passe est **admin123**.
Évidement, je vous conseils de le changer :

![Nexus-Admin-Account](/images/Nexus-Admin-Account.png)

Vous pouvez aussi refuser la consultation de votre registre aux utilisateurs non identifiés dans la configuration en décochant l'option *Allow anonymous users to access the server* :

![Nexus-Anonymous](/images/Nexus-Anonymous.png)

Passons a la configuration d'un repository npm pour les dépendances javascript de votre équipe.

La première étape consiste à configurer un 'blob storage' dans Nexus.
Il s'agit d'un espace de stockage où Nexus enregistre vos fichiers (dans notre cas lors d'un `npm publish`).
Vous pouvez configurer un blob de type local (sur le disque de votre machine) ou un bucket S3.
Dans mon cas, j'ai choisi de créer un bucket S3 sur Scaleway.
L'avantage est que je n'ai pas à me soucier de la sauvegarde des objets publiés. De plus si pour un besoin quelconque j'ai besoin de recréer/déménager ma VM, le changement sera facile.

Dans votre console scaleway, créez un bucket via le menu `object storage` :

![Scaleway-Create-Bucket](/images/Scaleway-Create-Bucket.png)

Ensuite vous aurez besoin d'un `Access key` et d'un `Secret key`, pour cela il faut les générer via https://console.scaleway.com/account/credentials

![Scaleway-Create-APIKEY](/images/Scaleway-Create-APIKEY.png)

Normalement la création d'un blob store dans nexus se fait facilement via l'interface web.
Malheureusement le formulaire de création possède une limitation bien gênante : la région est une liste prédéfinie avec les valeurs uniquement valable pour AWS.
J'ai réussi à contourner le problème en utilisant curl :

{{< highlight console >}}
curl 'https://nexus3.scw-par1-devtools.IP_VM_SCALEWAY.nip.io/service/extdirect' -H 'content-type: application/json' -H 'Accept: */*' --user admin:XXXXXXXXXX --data-binary '{"action":"coreui_Blobstore","method":"create","data":[{"type":"S3","name":"bucket-devtools","isQuotaEnabled":false,"attributes":{"s3":{"bucket":"bucket-devtools-darma","prefix":"nexus3/npm","accessKeyId":"SCWXXXXXXXXXXXXXXXXX","secretAccessKey":"abcdefgh-ijkl-mnop-qrst-uvwxyz","sessionToken":"","assumeRole":"","region":"nl-ams","endpoint":"https://s3.nl-ams.scw.cloud/","expiration":"-1","signertype":"DEFAULT","forcepathstyle":"false"}}}],"type":"rpc","tid":15}' --compressed
{{< / highlight >}}

Évidemment, remplacez les valeurs qui correspondent à votre domaine, nom de bucket, login/mot de passe nexus et access/secret de votre object storage.
La valeur de prefix, permet d'avoir un répertoire racine dans votre bucket (utile pour mutualiser le bucket ;-)).

Maintenant nous allons créer 3 repositories :

- un privé pour nos package perso :

![Nexus-Admin-Repo-NPM-Private](/images/Nexus-Admin-Repo-NPM-Private.png)

Le paramètre `Deployment policy` permet d'autoriser ou non la republication d'une même version.

- un proxy vers le registry npmjs :

![Nexus-Admin-Repo-NPM-Proxy](/images/Nexus-Admin-Repo-NPM-Proxy.png)

- un group qui permet de réunir les deux en un seul endpoint

![Nexus-Admin-Repo-NPM-Group](/images/Nexus-Admin-Repo-NPM-Group.png)

Il faut maintenant activer l'authentification NPM dans Nexus :

![Nexus-Admin-Realms](/images/Nexus-Admin-Realms.png)

Enfin ! Nous allons pouvoir faire un essai depuis notre machine de dév.

De manière générale, on utilise le repo npm-group :

{{< highlight console >}}
npm config set registry https://nexus3.scw-par1-devtools.IP_VM_SCALEWAY.nip.io/repository/npm-group/
npm login
{{< / highlight >}}

Dans ce cas, pour publier vos projets vous devrez paramétrer vos projet npm avec :

{{< highlight json >}}
{
  ...

  "publishConfig": {
    "registry": "https://nexus3.scw-par1-devtools.IP_VM_SCALEWAY.nip.io/repository/npm-private/"
  }
}
{{< / highlight >}}

Ou alors, on utilise uniquement le repo npm-private via les [scopes](https://docs.npmjs.com/about-scopes) npm :

{{< highlight console >}}
npm config set @moah:registry https://nexus3.scw-par1-devtools.IP_VM_SCALEWAY.nip.io/repository/npm-private/
npm login --scope=@moah
{{< / highlight >}}

source : https://blog.sonatype.com/using-nexus-3-as-your-repository-part-2-npm-packages

## Installation de SonarQube

Aller, on continue ! Créez un répertoire /opt/sonarqube/ et y mettre le fichier docker-compose.yml suivant :

**/opt/sonarqube/docker-compose.yml**
{{< highlight dockerfile >}}
version: "3"

services:
  sonarqube:
    image: sonarqube
    restart: always
    expose:
      - "9000"
    networks:
      - web
      - sonarnet
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://sonar-db:5432/sonar
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
    labels:
      - "traefik.backend=sonarqube"
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:sonarqube.scw-par1-devtools.IP_VM_SCALEWAY.nip.io"
      - "traefik.enable=true"
      - "traefik.port=9000"

  sonar-db:
    image: postgres:alpine
    networks:
      - sonarnet
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    volumes:
      - postgresql:/var/lib/postgresql
      # This needs explicit mapping due to https://github.com/docker-library/postgres/blob/4e48e3228a30763913ece952c611e5e9b95c8759/Dockerfile.template#L52
      - postgresql_data:/var/lib/postgresql/data

networks:
  web:
    external: true
  sonarnet:
    driver: bridge

volumes:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_bundled-plugins:
  postgresql:
  postgresql_data:
{{< / highlight >}}

{{< highlight console >}}
cd /opt/sonarqube/
docker-compose up -d
{{< / highlight >}}

A ce stade, vous devez avoir accès à une interface web via l'url précisé dans la variable *traefik.frontend.rule*.
Par défaut, le login est **admin** et le mot de passe est **admin**.
Évidement, je vous conseille de le changer :

![SonarQube-Admin-Account](/images/SonarQube-Admin-Account.png)

Administration > Configuration > Security
Activer **Force user authentication**

![SonarQube-ForceUserAuth](/images/SonarQube-ForceUserAuth.png)

## Installation d'un runner GitLab

Créez un répertoire pour stocker la configuration ainsi que nos clefs SSH pour le déploiement.
Récupérez le token d'enregistrement dans l'interface de GitLab dans le menu `Settings > CI / CD`.
Utilisez le générateur de configuration :

{{< highlight console >}}
$ mkdir -p /opt/gitlab-runner/config
$ mkdir -p /opt/gitlab-runner/ssh-keys
$ docker run --rm -t -i -v /opt/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
https://gitlab.com/
Please enter the gitlab-ci token for this runner:
ZVXnXXXWxfyGYAYYYoYv
Please enter the gitlab-ci description for this runner:
[8aab126cc600]: runner DevTools
Please enter the gitlab-ci tags for this runner (comma separated):
docker,devtools
Registering runner... succeeded                     runner=ZVXnXXXW
Please enter the executor: ssh, docker+machine, kubernetes, docker, docker-ssh, parallels, shell, virtualbox, docker-ssh+machine:
docker
Please enter the default Docker image (e.g. ruby:2.1):
alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
{{< / highlight >}}

Modifiez le fichier de configuration qui vient d'être généré :

{{< highlight toml >}}
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "runner DevTools ams1"
  url = "https://gitlab.com/"
  token = "XXXXXXXXXXXX"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache","/data_docker/gitlab-runner/ssh-keys:/ssh-keys"]
    shm_size = 0
  [runners.cache]
    Type = "s3"
    Path = "runners-cache"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.nl-ams.scw.cloud"
      AccessKey = "S3_ACCESS_KEY"
      SecretKey = "S3_SECRET_KEY"
      BucketName = "bucket-devtools-herschel"
      BucketLocation = "nl-ams"
      Insecure = false
    [runners.cache.gcs]
{{< / highlight >}}

J'ai rajouté dans `volumes` un montage de `ssh-keys` par défaut. On y stockera nos clef ssh pour le déploiement notamment. Je fait ça pour ne pas les mettre dans des variables d’environnements dans la version cloud de GitLab.
J'ai aussi ajouté la configuration pour stocker dans notre bucket S3 les fichiers de caches.
Exécutez le runner via la commande suivante :

{{< highlight console >}}
docker run -d --name gitlab-runner-devtools1 --restart always \
   -v /opt/gitlab-runner/config:/etc/gitlab-runner \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest
{{< / highlight >}}

source : https://docs.gitlab.com/runner/install/docker.html et https://docs.gitlab.com/runner/register/

Voilà, Voilà, vous êtes arrivés jusque ici ? Bravo ! 😎
La prochaine fois je le refais avec [Terraform](https://www.terraform.io/), ou alors sur quelques raspberry pi 🤯...

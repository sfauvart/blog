---
title: "MongoDb 3.4 sur Raspberry Pi 3 64Bits avec Docker (Hypriot)"
tags: [ "mongodb", "raspberrypi", "docker", "hypriot" ]
date: 2017-10-21T23:27:24+02:00
draft: false
---

Récemment je me suis lancé un petit défi : Utiliser un Raspberry Pi 3 pour auto-héberger un petit projet.

Le projet en question utilise :

- VueJS pour le Front
- GoLang pour le Back (API Rest)
- MongoDb pour le stockage des données

Voilà pour la stack applicative.

Pour héberger tout ça sur le Raspberrypi, j'ai utilisé un OS "spécial" pour le Raspberrypi 3 : [HypriotOS](https://blog.hypriot.com/)

Cet OS est une version 64Bits de Raspbian avec Docker pré-installé

Vous trouverez l'image à écrire sur votre carte SD là : https://github.com/DieterReuter/image-builder-rpi64/releases

(Un petit `sudo dd if=hypriotos-rpi64-v20171005-093459.img of=/dev/sda bs=1M` devrait faire l'affaire pour l'écriture vers la carte SD si */dev/sda* correspond bien à votre carte SD)

Par défaut votre raspberrypi sera accessible via un nom de machine défini à `black-pearl`.
Le nom est modifiable dans le fichier *device-init.yaml* situé dans le répertoire */dev/* de la carte SD.

{{< highlight console >}}
$ ssh pirate@black-pearl.local
{{< / highlight >}}

Le mot de passe par défaut est `hypriot`.

Ensuite j'ai utilisé la release officiel arm64 fourni par MongoDb pour réaliser mon Dockerfile : https://www.mongodb.com/download-center#community

Voici mon Dockerfile :

{{< highlight docker >}}
FROM arm64v8/debian:stretch

RUN groupadd -r mongodb && useradd -r -g mongodb mongodb

RUN apt-get update -y && apt-get -y install curl && \
   curl -O https://fastdl.mongodb.org/linux/mongodb-linux-arm64-ubuntu1604-3.4.9.tgz && \
   tar -zxvf mongodb-linux-arm64-ubuntu1604-3.4.9.tgz && \
   mv mongodb-linux-aarch64-ubuntu1604-3.4.9/bin/* /usr/bin/ && \
   rm mongodb-linux-arm64-ubuntu1604-3.4.9.tgz && \
   curl -O http://ftp.fr.debian.org/debian/pool/main/o/openssl/libssl1.0.0_1.0.1t-1+deb8u6_arm64.deb && \
   dpkg -i libssl1.0.0_1.0.1t-1+deb8u6_arm64.deb && \
   apt-get remove --purge -y curl && \
   rm -rf /tmp/* /var/tmp/* && \
   apt-get clean && \
   rm -rf /var/lib/apt/lists/*

RUN mkdir -p /data/db && chown -R mongodb:mongodb /data/db

USER mongodb

# Define mountable directories.
VOLUME ["/data/db"]

# Define working directory.
WORKDIR /data

# Define default command.
CMD ["mongod"]

# Expose ports.
#   - 27017: process
#   - 28017: http
EXPOSE 27017
EXPOSE 28017
{{< / highlight >}}

Ensuite, un petit `docker build -t sebf/arm64v8-mongodb .` permet la génération de l'image docker.
Enfin la commande `docker run -d -p 27017:27017 -p 28017:28017 sebf/arm64v8-mongodb:latest` démarrera un container avec MongoDb 3.4.9 !

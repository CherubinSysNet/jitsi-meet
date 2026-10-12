<p align="center">
  <img src="https://img.shields.io/badge/Ubuntu-22.04%20%7C%2024.04%20%7C%2026.04-E95420?logo=ubuntu&logoColor=white" alt="Ubuntu 22.04 | 24.04">
  <img src="https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white" alt="Docker Compose">
  <img src="https://img.shields.io/badge/Jitsi-Meet-blue?logo=jitsi&logoColor=white" alt="Jitsi Meet">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License MIT">
  <img src="https://img.shields.io/badge/Status-Operational-success" alt="Status Operational">
</p>

# Déploiement de Jitsi Meet en réseau local (LAN)

> Mise en place d'une plateforme de visioconférence open source,
> auto-hébergée et accessible depuis le réseau local, à l'aide de Docker
> Compose.

## Présentation

Ce projet documente l'installation et la configuration de **Jitsi Meet**
sur Ubuntu, afin de fournir un service de visioconférence interne sans
dépendre d'une plateforme cloud tierce.

L'objectif est de permettre à plusieurs postes connectés au même réseau
local de créer ou rejoindre une salle de réunion, avec audio, vidéo,
chat et partage d'écran.

## Objectifs du projet

-   Déployer une solution de visioconférence auto-hébergée.
-   Rendre la plateforme accessible depuis le LAN.
-   Déployer les services avec Docker Compose.
-   Assurer la persistance de la configuration.
-   Diagnostiquer et résoudre des incidents de ports, de permissions et
    de connectivité entre participants.
-   Documenter les tests et les procédures d'exploitation.

## Environnement technique

  Élément                  Configuration
  ------------------------ -------------------------------
  Système d'exploitation   Ubuntu 22.04 / 22.04 / 26.04
  Conteneurisation         Docker et Docker Compose
  Solution                 Jitsi Meet
  Adresse IP du serveur    `192.168.88.137`
  URL d'accès              `https://192.168.88.137:8443`
  Réseau cible             Réseau local (LAN)
  Fuseau horaire           `Africa/Lubumbashi`

> **Important :** l'adresse IP et les paramètres présentés correspondent
> à l'environnement de laboratoire utilisé. Adaptez-les à votre réseau
> avant le déploiement.

## Architecture

La plateforme repose sur les principaux composants suivants :

  Composant                     Rôle
  ----------------------------- ---------------------------------------------
  **Web (Nginx)**               Interface web utilisée par les participants
  **Prosody**                   Serveur XMPP pour la signalisation
  **Jicofo**                    Gestion et orchestration des conférences
  **Jitsi Videobridge (JVB)**   Acheminement des flux audio et vidéo

### Ports utilisés

  -----------------------------------------------------------------------
  Port                    Protocole               Usage
  ----------------------- ----------------------- -----------------------
  `8000`                  TCP                     Accès HTTP

  `8443`                  TCP                     Accès HTTPS à
                                                  l'interface

  `10000`                 UDP                     Flux audio/vidéo via
                                                  JVB

  `8082`                  TCP, local              Port hôte utilisé pour
                                                  COLIBRI/JVB

  `5222`, `5269`, `5280`, TCP                     Communications XMPP et
  `5347`                                          internes selon la
                                                  configuration
  -----------------------------------------------------------------------

Les ports internes ne doivent pas nécessairement être exposés au réseau.
N'ouvrez que les ports nécessaires à votre architecture.

## Prérequis

-   Une machine Ubuntu 22.04 ou 24.04.
-   Au moins **4 Go de RAM** (8 Go recommandés) et 2 vCPU minimum pour
    ce laboratoire.
-   Un compte disposant des droits `sudo`.
-   Docker Engine et le plugin Docker Compose.
-   Une adresse IP stable et une connectivité entre les postes du LAN.

## Installation

Les commandes ci-dessous résument le déploiement. Consultez les fichiers
de configuration officiels de Jitsi Docker Meet pour les détails propres
à la version choisie.

### 1. Installer Docker

``` bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Installez ensuite Docker Engine et le plugin Compose depuis le dépôt
officiel Docker pour Ubuntu, puis vérifiez :

``` bash
docker --version
docker compose version
```

Si vous ajoutez votre compte au groupe `docker`, déconnectez-vous puis
reconnectez-vous. Le groupe `docker` accorde des privilèges élevés sur
la machine.

### 2. Récupérer le projet Jitsi Meet Docker

Téléchargez une version stable du dépôt officiel
`jitsi/docker-jitsi-meet`, puis entrez dans le répertoire extrait.

### 3. Préparer la configuration

``` bash
cp env.example .env
./gen-passwords.sh
```

Adaptez ensuite les variables nécessaires dans `.env`, notamment :

``` dotenv
PUBLIC_URL=https://192.168.88.137:8443
ENABLE_LETSENCRYPT=0
HTTP_PORT=8000
HTTPS_PORT=8443
JVB_PORT=10000
JVB_ADVERTISE_IPS=192.168.88.137
JVB_COLIBRI_PORT=8082
TZ=Africa/Kinshasa
```

Générez des mots de passe uniques et conservez les secrets hors du dépôt
Git. **Ne publiez jamais le fichier `.env` contenant des secrets.**

### 4. Créer les répertoires de configuration persistante

``` bash
mkdir -p ~/.jitsi-meet-cfg/{web,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb}
```

Les permissions doivent correspondre aux utilisateurs utilisés par les
conteneurs. Vérifiez les besoins de la version déployée avant
d'appliquer un `chown` récursif.

### 5. Configurer le pare-feu

Exemple à adapter à votre politique réseau :

``` bash
sudo ufw allow 22/tcp
sudo ufw allow 8000/tcp
sudo ufw allow 8443/tcp
sudo ufw allow 10000/udp
sudo ufw status
```

N'activez UFW qu'après avoir vérifié les règles nécessaires à votre
accès distant, notamment SSH.

### 6. Démarrer les services

Dans le répertoire du projet :

``` bash
docker compose up -d
docker compose ps
```

## Accéder à la plateforme

Depuis un poste connecté au même LAN, ouvrez :

**https://192.168.88.137:8443**

Dans ce laboratoire, HTTPS utilise un certificat auto-signé. Le
navigateur peut afficher un avertissement de sécurité. Pour un
déploiement durable, configurez un certificat de confiance et un nom DNS
adapté.

Pour rejoindre une salle appelée `reunion-test`, utilisez :

``` text
https://192.168.88.137:8443/reunion-test
```

Les participants doivent pouvoir joindre le serveur sur les ports
requis, en particulier le port UDP `10000` pour les flux audio/vidéo.

## Incidents rencontrés et résolutions

### 1. Conflit sur le port 8080

**Symptôme :** Docker ne peut pas réserver le port hôte `8080`, car il
est déjà utilisé.

**Résolution appliquée :**

``` dotenv
JVB_COLIBRI_PORT=8082
```

Puis redémarrage des services :

``` bash
docker compose down
docker compose up -d
```

### 2. Le conteneur Web redémarre en boucle

**Symptôme :** le conteneur ne peut pas écrire dans le répertoire
`/storage`.

**Cause identifiée :** permissions incompatibles sur le volume de
configuration.

**Résolution dans le laboratoire :**

``` bash
sudo chown -R 1000:1000 ~/.jitsi-meet-cfg/
docker compose down
docker compose up -d
```

> Vérifiez l'UID réellement utilisé par les conteneurs avant de
> reproduire cette commande dans un autre environnement.

### 3. Le deuxième participant ne peut pas rejoindre la conférence

**Pistes de diagnostic :** - vérifier que Jicofo détecte le pont vidéo
; - examiner les journaux de JVB, Jicofo et Prosody ; - confirmer la
valeur annoncée par `JVB_ADVERTISE_IPS` ; - vérifier l'accessibilité du
port UDP `10000` depuis le LAN.

Commandes utiles :

``` bash
docker compose logs jvb --tail=100
docker compose logs jicofo --tail=100
docker compose logs prosody --tail=100
curl -s http://localhost:8888/stats
```

Dans le laboratoire, un redémarrage ordonné de Prosody, Jicofo puis JVB,
suivi de la validation réseau, a permis de résoudre le problème.

## Tests de validation

  Test                                  Résultat observé
  ------------------------------------- ------------------------
  Docker et Docker Compose installés    Validé
  Conteneurs démarrés                   Validé
  Réponse HTTPS locale                  `HTTP 200 OK`
  Accès depuis un poste du LAN          Validé
  Port UDP `10000` exposé et autorisé   Validé
  Réunion entre deux postes             Audio et vidéo validés

Ces résultats décrivent les tests réalisés dans l'environnement du
projet ; ils ne garantissent pas le même résultat sans adaptation sur un
autre réseau.

## Exploitation et maintenance

### Démarrer

``` bash
docker compose up -d
```

### Arrêter

``` bash
docker compose down
```

### Redémarrer

``` bash
docker compose restart
```

### Consulter les journaux

``` bash
docker compose logs --tail=100
docker compose logs jvb --tail=100
docker compose logs jicofo --tail=100
docker compose logs web --tail=100
```

### Mettre à jour

``` bash
docker compose pull
docker compose up -d
```

Avant une mise à jour, consultez les notes de version et sauvegardez la
configuration.

### Sauvegarder la configuration

Depuis le répertoire du projet, en adaptant les chemins à votre
installation :

``` bash
tar -czf jitsi-backup-$(date +%F).tar.gz ~/.jitsi-meet-cfg/ .env
```

**Attention :** cette archive peut contenir des secrets. Stockez-la dans
un emplacement protégé et ne la publiez pas sur GitHub.

## Améliorations envisagées

-   Configurer un DNS local, par exemple `meet.local`.
-   Déployer un certificat TLS de confiance pour les clients.
-   Mettre en place une authentification et contrôler la création des
    salles.
-   Automatiser les sauvegardes et tester la restauration.
-   Documenter une procédure de reprise après incident.
-   Ajouter une supervision de la disponibilité des conteneurs et des
    ports réseau.

## Compétences mobilisées

-   Administration Linux (Ubuntu).
-   Déploiement de services avec Docker Compose.
-   Réseaux TCP/IP, règles de pare-feu et diagnostic UDP.
-   Configuration de services de communication.
-   Analyse des journaux et résolution d'incidents.
-   Documentation technique et validation de service.

## Auteur

**Cherubin Tshienda** --- *MANDRIVA*\
Administration systèmes et réseaux

------------------------------------------------------------------------

*Projet de laboratoire réalisé et documenté le 11 octobre 2026.*

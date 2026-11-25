# Exercice guidé — Construire une image Debian avec Apache HTTP Server

## Objectif

Vous devez écrire vous-même un `Dockerfile` pour construire une image qui :

- utilise Debian comme base ;
- installe Apache HTTP Server ;
- intègre le fichier `index.html` fourni dans le dépôt ;
- démarre automatiquement le serveur HTTP au lancement du conteneur ;
- sert la page sur le port interne `80`.

Ce guide indique la démarche et les commandes de vérification. Il ne fournit ni Dockerfile complet, ni squelette à compléter : la rédaction du Dockerfile fait partie de l’évaluation.

> Cet exercice Debian/Apache est distinct du portail Nginx de l’infrastructure Compose.

## 1. Cloner le dépôt

```bash
git clone https://github.com/nxcourses/portail_docker.git
cd portail_docker
ls -la
```

Repérez le fichier `index.html` fourni. Vous allez créer votre `Dockerfile` dans ce répertoire.

## 2. Comprendre le terme « httpd » sous Debian

Dans ce sujet, « httpd » désigne Apache HTTP Server. Sur Debian, le paquet à installer est `apache2`, et le serveur se démarre avec `apache2ctl`.

N’utilisez ni l’image officielle `httpd`, ni une image Nginx : vous devez installer Apache vous-même dans une image Debian.

## 3. Préparer le répertoire de travail

Créez avec votre éditeur un fichier nommé exactement `Dockerfile`, sans extension, au même niveau que `index.html`.

```text
portail_docker/
├── Dockerfile       # À rédiger par le candidat
└── index.html       # Fichier fourni dans le dépôt
```

Si `index.html` est dans un sous-répertoire, adaptez le chemin source de votre instruction `COPY`. N’utilisez pas de fichier `.env`.

## 4. Choisir l’image de base

Sélectionnez une image officielle Debian avec un tag explicite ; une variante `slim` est acceptée.

Instruction à rechercher : `FROM`. Vous devez pouvoir justifier votre choix.

## 5. Installer Apache pendant la construction

Pendant le build, votre Dockerfile doit :

1. mettre à jour l’index des paquets avec `apt-get update` ;
2. installer le paquet `apache2` sans interaction ;
3. éviter les paquets recommandés inutiles ;
4. supprimer les listes de paquets de `/var/lib/apt/lists/`.

Instruction à rechercher : `RUN`.

Regroupez ces opérations dans une même instruction et arrêtez la construction si une étape échoue. N’installez pas Apache dans un conteneur lancé pour utiliser ensuite `docker commit` : le build doit être reproductible.

## 6. Copier la page dans le bon répertoire

Le répertoire web par défaut d’Apache sous Debian est `/var/www/html/`. Le fichier `index.html` du dépôt doit se retrouver ici dans l’image :

```text
/var/www/html/index.html
```

Instruction à rechercher : `COPY`.

La source est le fichier du dépôt cloné ; la destination est le chemin ci-dessus. La page doit remplacer la page d’accueil Apache par défaut. Elle doit être intégrée à l’image, pas simplement montée au lancement.

## 7. Documenter le port HTTP

Déclarez le port `80` dans le Dockerfile avec l’instruction `EXPOSE`. Cette déclaration ne publie pas le port sur l’hôte : la publication se fait au lancement.

## 8. Définir le démarrage du serveur

Apache doit démarrer automatiquement et rester au premier plan comme processus principal du conteneur :

```bash
apache2ctl -D FOREGROUND
```

Instructions à rechercher : `CMD` et `ENTRYPOINT`. Choisissez une stratégie et privilégiez la forme exec (tableau JSON).

N’utilisez pas `systemctl`, un démarrage en arrière-plan suivi de `tail -f`, ni un conteneur maintenu artificiellement en vie.

## 9. Relire votre Dockerfile

Vérifiez qu’il décrit bien : la base Debian, l’installation non interactive d’Apache, le nettoyage des listes de paquets, la copie vers `/var/www/html/index.html`, le port `80` et le démarrage au premier plan.

## 10. Construire l’image

Depuis le répertoire contenant le Dockerfile :

```bash
docker build -t debian-apache-lab:1.0 .
docker image ls debian-apache-lab
```

`-t` nomme et tague l’image ; le point final désigne le contexte de construction, c’est-à-dire le répertoire courant. `index.html` doit être dans ce contexte et ne pas être exclu par un `.dockerignore`.

## 11. Lancer un conteneur de test

N’utilisez pas les ports `8080`, `8081` et `8082`, réservés à l’infrastructure :

```bash
docker run -d --name apache-test -p 8090:80 debian-apache-lab:1.0
docker ps --filter name=apache-test
docker logs apache-test
curl -I http://localhost:8090/
```

Depuis un navigateur, ouvrez `http://192.168.56.101:8090/`. Vous devez voir votre page personnalisée, pas la page Apache par défaut.

## 12. Vérifier le contenu de l’image

```bash
docker exec apache-test sh -c 'test -s /var/www/html/index.html && ls -l /var/www/html/index.html'
docker exec apache-test apache2ctl -t
```

Le fichier doit exister et la configuration Apache doit être valide.

> Les liens de la page utilisent les noms `Wordpress`, `Phpmyadmin` et `Registry`. Ce test valide l’affichage, pas l’accès aux applications : le conteneur de test n’est pas rattaché à `sw1`, et un navigateur externe ne bénéficie pas automatiquement du DNS interne Docker.

## 13. Modifier la page et reconstruire

Modifiez un texte visible dans `index.html`, puis :

```bash
docker build -t debian-apache-lab:1.1 .
docker rm -f apache-test
docker run -d --name apache-test -p 8090:80 debian-apache-lab:1.1
```

Rechargez la page : la modification doit apparaître. Modifier le fichier local ne change pas une image déjà construite.

## 14. Livrables

- le Dockerfile rédigé ;
- la commande de construction et la preuve de réussite du build ;
- une capture de la page servie sur le port `8090` ;
- une courte explication du démarrage au premier plan et de la différence entre port exposé et port publié.

## 15. Nettoyage

```bash
docker rm -f apache-test
docker image rm debian-apache-lab:1.0 debian-apache-lab:1.1
```

## Documentation de référence

- Dockerfile : https://docs.docker.com/build/concepts/dockerfile/
- Contexte de construction : https://docs.docker.com/build/concepts/context/
- Construction d’une image : https://docs.docker.com/reference/cli/docker/image/build/
- Apache sous Debian : https://manpages.debian.org/trixie/apache2-bin/apache2.8.en.html

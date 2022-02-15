# TP 01 - Docker

## Lancement du container psql

---

### Container postgres

`docker pull postgres:11.6-alpine`

creation du Dockerfile

```
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```

`docker build -t psql .`

`docker run -p 5432:5432 --network app-network --name postgres psql -d -it`

### Acces avec adminer

On créer un network pour que les container puissent communiquer entre eux :

`docker network create app-network`

Relancer le container psql dans le network qu'on vient de créer :

`docker run -p 5432:5432 --network app-network --name postgres psql -d -it`

Installation et build d'adminer :

`docker pull adminer`

`docker build -t adminer adminer`

`docker run -p 8080:8080 --name adminer --network app-network adminer`

On accède à l'interface adminer sur localhost:8080

### Ajout des scripts sql

Création des scripts sql fournis dans le répertoire du Dockerfile

Ajouter les lignes suivantes au Dockerfile :

```
COPY 01-CreateSchema.sql /docker-entrypoint-initdb.d/

COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```

Rebuild le container

#### persistance des données

Relancer le container Docker avec cette commande

`docker run -p 5432:5432 --network app-network -v ~/psqldata/:/var/lib/postgresql/data --name postgres psql -d -it`

```
Why should we run the container with a flag -e to give the environment variables ?

On utilise l'option -e pour ne pas écrire les identifiants dans la base de données
```

```
Why do we need a volume to be attached to our postgres container ?

Nous utilisons un volume pour stocker la base de données pour éviter que la base et donc toutes les données de l'application soient détruites à chaque redémarrage du container ou a sa suppression
```

## Lancement du container Java simple api

---

Le DockerFile :

```
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn dependency:go-offline
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

`docker build -t openjdk .`

`docker run -p 8080:8080 --network app-network --name simple-api openjdk -d -it`

## Lancement du container Apache reverse proxy

---

Le DockerFile :

```
FROM httpd
COPY ./public/ /usr/local/apache2/htdocs/
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

Les modifications au fichier httpd.conf :

```

LoadModule proxy_module modules/mod_proxy.so # <-- décommenté
...
LoadModule proxy_http_module modules/mod_proxy_http.so # <-- décommenté
...
ServerName localhost
<VirtualHost *:8080>
    ProxyPreserveHost On
    ProxyPass / http://simple-api:8080/
    ProxyPassReverse / http://simple-api:8080/
</VirtualHost>
```

On ajoute un fichier html dans le répertoire public :

```
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Accueil</title>
</head>

<body>
    <p>Hello world !</p>

</body>

</html>
```

`docker build -t my_apache .`

`docker run -dit --network app-network --name my-apache-server -p 80:80 my-apache`

On a maintenant accès à l'api sur localhost:8080

## Docker compose

---

```
version: '3.7'
services:
  backend:
    container_name:
      simple-api
    build:
      ./simple-api/
    networks:
      - my-app-network
    depends_on:
      - database
  frontend:
    container_name:
      devops-front
    build:
      ./devops-front/
    networks:
      - my-app-network
    depends_on:
      - database
      - backend
  database:
    container_name:
      postgres
    build:
      ./psql
    networks:
      - my-app-network
    volumes:
      - ~/psqldata/:/var/lib/postgresql/data
  httpd:
    container_name:
      my-apache-server
    build:
      ./apache
    ports:
      - "80:80"
      - "8080:8080"
    networks:
      - my-app-network
    depends_on:
      - backend
      - database
      - frontend
networks:
  my-app-network:
```

Pour lancer les container avec docker-compose :

`docker-compose up -d`

Relancer un seul service :

`docker-compose restart backend-api`

Arrêter tout les services :

`docker-compose stop`

```
Why do we put our images into an online repository ?

Pour pouvoir faire du continuous deployment
```

# TP 2 - GitHub Actions

## CI

---

Pour lancer les tests de l'application sur la machine :

`mvn clean verify`

```
mvn clean verify what is it supposed to do ?

Cette commande permet de clean le build précédent et de charger toutes les dépendances, elle joue aussi les tests
```

```
Unit tests ? Component test ?

Les tests unitaires visent à tester des composantes individuelles du code et vérifier leurs fonctionnements
Les tests d'intégrations sont utilisés pour tester les interactions entre les différents modules (après regroupement)
```

```
What are testcontainers?

Des container qui servent à jouer les tests JUnit
```

On créer un répertoire git :

```
git init
git add .
git commit -m "init commit"
git branch -M main
git remote add origin git@github.com:eloibrd/cpe-devops.git
git push -u origin main
```

On créer à la racine du répertoire les dossiers .github/workflows/

On créer dans le dossier workflows le fichier main.yml :

```
name: CI/CD devops 2022 CPE EBD
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      # checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3

      # do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'

      # cache maven
      - uses: actions/cache@v1
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-
      # build and run maven tests
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=eloibrd_cpe-devops -Dsonar.organization=eloibrd -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONARCLOUD_TOKEN }} --file ./pom.xml
        working-directory: simple-api
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONARCLOUD_TOKEN }}
```

## CD

---

```
Secured variables, why ?

Pour pouvoir utiliser des données sensibles comme des identifiants et mots de passe ou tokens personnels dans les pipeline sans les exposer en dur
```

On lance les jobs de déploiement des images docker en parallèle en ajoutant ces instructions dans le fichier main.yml, dans les jobs :

```
# build and deploy docker images

  build_and_deploy_back_image:
    needs: test-backend
      # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
      # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_USR }} -p ${{secrets.DOCKER_TOKEN}}

      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          context: ./simple-api
          tags: ${{secrets.DOCKER_USR}}/simple-api
          push: ${{ github.ref == 'refs/heads/main' }}

  build_and_deploy_front_image:
    needs: test-backend
      # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
      # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_USR }} -p ${{secrets.DOCKER_TOKEN}}

      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build image and push frontend
        uses: docker/build-push-action@v2
        with:
          context: ./devops-front
          tags: ${{secrets.DOCKER_USR}}/devops-front
          push: ${{ github.ref == 'refs/heads/main' }}

  build_and_deploy_database_image:
    needs: test-backend
      # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
      # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_USR }} -p ${{secrets.DOCKER_TOKEN}}

      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./psql
          tags: ${{secrets.DOCKER_USR}}/psql
          push: ${{ github.ref == 'refs/heads/main' }}

  build_and_deploy_proxy_image:
    needs: test-backend
      # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
      # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_USR }} -p ${{secrets.DOCKER_TOKEN}}

      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./apache
          tags: ${{secrets.DOCKER_USR}}/my-apache-server
          push: ${{ github.ref == 'refs/heads/main' }}
```

```
Why did we put needs: build-and-test-backend on this job?

Pour build l'application afin de pouvoir placer les fichiers dans le container api
```

## Quality Gate

---

Modifications effectuées dans le pipeline pour que le Sonar se lance à chaque push :

```
    # build and run maven tests
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=eloibrd_cpe-devops -Dsonar.organization=eloibrd -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONARCLOUD_TOKEN }} --file ./pom.xml
        working-directory: simple-api
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONARCLOUD_TOKEN }}
```

# TP 3 - Ansible

## Accès à distance au serveur

---

### Accès à distance au serveur

On test l'accès à distance après avoir récupéré notre clé privée :

`ssh -i ~/.ssh/takima_key_private centos@eloi.bernard.takima.cloud`

### Connexion avec Ansible

Après l'installation d'Ansible on doit rajouter notre serveur à la liste des hôtes qui se situe dans /etc/ansible/hosts :

`centos@eloi.bernard.takima.cloud`

On essaye de ping :
`ansible all -m ping`

Erreur : il faut fournir la clé ssh pour réaliser des opérations

`ansible all -m ping --private-key=~/.ssh/takima_key_private -u centos`

```

centos@eloi.bernard.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}

```

## Inventaire Ansible

On créer un fichier inventaire Ansible setup.yml dans le répertoire ansible/inventories/

```

all:
vars:
ansible_user: centos
ansible_ssh_private_key_file: ~/.ssh/takima_key_private
children:
prod:
hosts: eloi.bernard.takima.cloud

```

On test la configuration avec la comande suivante :

`sudo ansible all -i ansible/inventories/setup.yml -m ping`

## Playbooks

---

### Playbook pour ping les hôtes

```

-   hosts: all
    gather_facts: false
    become: yes
    tasks:
    -   name: Test connection
        ping:

```

Ce playbook permet de ping tout les hôtes présents dans l'inventaire

`ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml`

### Installation de Docker sur le serveur

On change les tâches du playbook comme ceci afin d'installer docker :

```
tasks:
    - name: Clean packages
      command:
        cmd: dnf clean -y packages
    - name: Install device-mapper-persistent-data
      dnf:
        name: device-mapper-persistent-data
        state: latest
    - name: Install lvm2
      dnf:
        name: lvm2
        state: latest
    - name: add repo docker
      command:
        cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    - name: Install Docker
      dnf:
        name: docker-ce
        state: present
    - name: install python3
      dnf:
        name: python3
    - name: Pip install
      pip:
        name: docker
    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker

```

## Installation par rôles

---

On va créer un rôle pour installer docker :

`ansible-galaxy init roles/docker`

On déplace les tâches du playbook dans ansible/roles/docker/tasks/main.yml :

```

# tasks file for roles/docker

-   name: Clean packages
    command:
    cmd: dnf clean -y packages

-   name: Install device-mapper-persistent-data
    dnf:
    name: device-mapper-persistent-data
    state: latest

-   name: Install lvm2
    dnf:
    name: lvm2
    state: latest

-   name: add repo docker
    command:
    cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

-   name: Install Docker
    dnf:
    name: docker-ce
    state: present

-   name: install python3
    dnf:
    name: python3

-   name: Pip install
    pip:
    name: docker

-   name: Make sure Docker is running
    service: name=docker state=started
    tags: docker

```

On modifie le playbook.yml pour qu'il prenne en compte le role docker :

```

-   hosts: all
    roles:
    -   docker
        gather_facts: false
        become: yes

```

On créé les rôles suivants à l'aide de ansible-galaxy :

-   network
-   database
-   app
-   proxy

### Rôle network

ansible/roles/network/tasks/main.yml

```

# tasks file for roles/network

-   name: create network
    docker_network:
    name: app-network

```

### Rôle database

ansible/roles/database/tasks/main.yml

```

# tasks file for roles/database

-   name: Run database container
    docker_container:
    name: postgres
    image: eloibrd/psql
    networks: - name: app-network

```

### Rôle app

ansible/roles/app/tasks/main.yml

```

# tasks file for roles/app

-   name: Run backend app container
    docker_container:
    pull: true
    name: simple-api
    image: eloibrd/simple-api
    # ports:
    # "8080:8080"
    networks: - name: app-network

```

### Rôle proxy

ansible/roles/proxy/tasks/main.yml

```

# tasks file for roles/proxy

-   name: Run proxy container
    docker_container:
    name: my-apache-server
    image: eloibrd/my-apache-server
    ports: - "8080:8080" # on expose l'api
    networks: - name: app-network
    pull: true

```

On met à jour le playbook pour qu'il prenne en compte tout les rôles :

```

-   hosts: all
    roles:
    -   docker
    -   network
    -   database
    -   app
    -   proxy
        gather_facts: false
        become: yes

```

## Ajout du Front-End

Une fois le front récupéré, on modifie le rôle proxy pour qu'il expose le front sur le port http :

ansible/roles/proxy/tasks/main.yml

```

# tasks file for roles/proxy

-   name: Run proxy container
    docker_container:
    name: my-apache-server
    image: eloibrd/my-apache-server
    ports: - "8080:8080" # on expose l'api - "80:80" # on expose le front
    networks: - name: app-network
    pull: true

```

On modifie le rôle app pour qu'il fasse tourner également le front :

ansible/roles/app/tasks/main.yml

```

# tasks file for roles/app

-   name: Run backend app container
    docker_container:
    pull: true
    name: simple-api
    image: eloibrd/simple-api
    # ports:
    # "8080:8080"
    networks: - name: app-network
-   name: Run frontend app container
    docker_container:
    pull: true
    name: devops-front
    image: eloibrd/devops-front
    # ports:
    # "80:80"
    networks: - name: app-network

```

Enfin il faut modifier la configuration du proxy apache :

```

ServerName localhost
<VirtualHost _:80>
    ProxyPreserveHost On
    ProxyPass / http://devops-front:80/
    ProxyPassReverse / http://devops-front:80/
</VirtualHost>
<VirtualHost _:8080>
    ProxyPreserveHost On
    ProxyPass / http://simple-api:8080/
    ProxyPassReverse / http://simple-api:8080/
</VirtualHost>

```

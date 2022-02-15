# Compte-Rendu TP01

# Database

## Basics

- Lancer la base de données :
```bash
sudo docker run -d -p 3306:3306 --name postegre --network app-network bastou/mysecondapp
```
- Lancer adminer :
```bash
sudo docker run -p 8080:8080 --network=app-network adminer
```

L'avantage de mettre un -e pour les variables d'environnement est que les informations critique ne sont pas en clairs dans un fichier, comme par exemple les mots de passe.

Ces données seront données directement lorsqu'on exécute la commande.

## Init DB

Voici les modifications apportés au fichier Dockerfile pour exécuter les 2 scripts SQL :
```docker
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd

COPY /CreateScheme.sql /docker-entrypoint-initdb.d 
COPY /InsertData.sql /docker-entrypoint-initdb.d 

```

## Persistance

Pour mettre en place la persistance, il faut d'abord créer un dossier où sera stocké les données en local.

Commande à lancer pour démarrer le container PostegreSQL :
```bash
 sudo docker run -d --rm -p 3306:3306 --name postgre -v /home/bastou/TP01/VolPostgre:/var/lib/postgresql/data --network app-network bastou/mysecondapp
 ```

Nous avons besoin d'une persistance pour une base de données car nous avons besoin d'enregistrer les modifications faites dessus. Si nous ne mettons pas en place la persistance, les données modifiées ne seront pas enregistrés et nous les perdrons lorsque le container sera redémarré.

# Backend API

## Basics

>En admettant que nous sommes dans le dossier /TP01/Java

- Composition du Dockerfile :
```docker
FROM openjdk:11
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
CMD ["java", "Main"]
```
- Commande à exécuter pour build :
```bash
sudo docker build -t bastou/java .
```
- Commande à exécuter pour run le container :
```bash 
sudo docker run -it --rm --name java bastou/java
```
- Voici le retour du run :
```bash
Hello World!
```

- Lancer API
```
 sudo docker run -d --name backend --network app-network -p 8081:8080 bastou/java
```



## Multistage Build

- Dockerfile
```docker
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY simple-api/pom.xml .
COPY simple-api/src ./src
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

- build 
```bash
sudo docker build -t bastou/java .
```

- run
```bash
sudo docker run -d --name backend -p 8080:8080 bastou/java
```
**Why do we need  a multistage build ? And explain each steps of this dockerfile ?**

> L'utilisation de plusieurs instances permet d'abord d'exécuter les packages lourds puis les packages moins lourd. Ici, on exécute d'abord  le package maven (jdk) puis ensuite le package .jar (jre).


## HTTPD

Dockerfile

```
FROM httpd:2.4
COPY public-html/index.html /usr/local/apache2/htdocs/
```

Index.html

```
Hello World !
```

Commande :

```bash
sudo docker build -t bastou/httpd .
sudo docker run -d -p 8080:80 --name httpd bastou/httpd
```

- Modification du httpd.conf, décommenter et rajouter les lignes dans le httpd.conf :
```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
[...]
ServerName localhost
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

> Rebuild et exécuter

## Docker-Compose

Fichier docker-compose.yml :
```
version: '3.7'
services:

  postgre:
    build: ./Postgre
    networks:
    - app-network

  backend:
    build: ./Java
    networks:
    - app-network
    depends_on:
    - postgre

  httpd:
    build: ./httpd
    ports:
    - "80:80"
    networks:
    - app-network
    depends_on:
    - postgre
    - backend
    
networks:
  app-network: 
```

> Exécution de la commande "_docker-compuse up -d_" pour démarrer le docker-compose.


## Publish

Mise en place des tags sur les versions que nous allons publier sur Docker-Hub. Puis push sur la plateforme avec le précision de la version. Cela est important pour se retrouver dans le versionning des images que l'on publie.
> Mise en place des tags

```bash
sudo docker tag tp01_httpd bastienzanardocpe/httpd:1.0
sudo docker tag tp01_backend bastienzanardocpe/backend:1.0
sudo docker tag tp01_postgre bastienzanardocpe/postgre:1.0
```

> Publish
```
sudo docker push bastienzanardocpe/httpd:1.0
sudo docker push bastienzanardocpe/backend:1.0
sudo docker push bastienzanardocpe/postgre:1.0
```

**Why is docker-compose so important ?**

> Docker-Compose permet de démarrer ou d'éteindre plusieurs docker en même temps. De plus, il permet d'avoir toutes la configuration de tous les dockers réuni sur un seul et même fichier. Il permet notamment de pouvoir faire en sorte de démarrer tous les dockers au démarrage du serveur.


_________________________

# Compte-Rendu TP02

# Setup Github Actions

## First steps into the CI world

> Ok, what is it supposed to do ?

L'utilisation de *sudo mvn clean verify* permet de vérifier la qualité du code.

> Unit tests ? Component test ?

Test unitaire permet de tester qu'une partie du code sans que tous le reste du code influe sur le résultat.

Un compenent test est un test sur une partie du code qui est fait independemment des autres parties, sans que cela soit intégré.

> What are testcontainers?

Testcontainers est une librairie Java qui permet de tester des modules du code qui sont défini à l'avance.

- Fichier de configuration .main.yml :
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: master
  pull_request:
jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3

      #do the same with another action (actions/setup-java@v2) thatenable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'
          java-version: '11'
        #TODO
      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file ./Java/simple-api/pom.xml
```

# First step into the CD World

- Quality gate configuration :
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: master
  pull_request:

env:
  GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}


jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3

      #do the same with another action (actions/setup-java@v2) thatenable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'
          java-version: '11'
        #TODO
      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=bastienzanardocpe_TP01DevOps -Dsonar.organization=bastienzanardocpe -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONAR_TOKEN }} --file ./Java/simple-api/pom.xml


  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to DockerHub
        run: docker login -u ${{secrets.USER_DOCKERHUB}} -p ${{secrets.TOKEN_DOCKERHUB}}
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Java
          # Note: tags has to be all lower-case
          tags: ${{secrets.USER_DOCKERHUB}}/tp-devops-cpe:Java
          push: ${{ github.ref == 'refs/heads/master'}}
  
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Postgre
          # Note: tags has to be all lower-case
          tags: ${{secrets.USER_DOCKERHUB}}/tp-devops-cpe:Postgre
          push: ${{ github.ref == 'refs/heads/master'}}
  
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./httpd
          # Note: tags has to be all lower-case
          tags: ${{secrets.USER_DOCKERHUB}}/tp-devops-cpe:httpd
          push: ${{ github.ref == 'refs/heads/master'}}

      - name: Build image and push front
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./front
          # Note: tags has to be all lower-case
          tags: ${{secrets.USER_DOCKERHUB}}/tp-devops-cpe:front
          push: ${{ github.ref == 'refs/heads/master'}}
```

__________________________

# TP part 03 - Ansible

# Intro 

## Inventories

> Configuration du fichier setup.yml :
```yml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: /home/bastou/TP03/id_rsa
  children:
    prod:
      hosts: bastien.zanardo.takima.cloud
```

Création de l'architecture /home/bastou/TP03/ansible/inventories.

>Toutes les commandes suivantes sont lancés dans le dossier /home/bastou/TP03/ansible/inventories
```bash
ansible all -i setup.yml -m ping
```
On a bien une réponse 'Pong'

## Facts

> Nous sommes toujours dans le dossier mentionné ci-dessus
```bash
ansible all -i setup.yml -m setup -a "filter=ansible_distribution*"
```
Grâce à cette commande, nous apprenons que nous sommes sur une machine CentOS version 8.

Pour remove les packages httpd (pour apache), nous utilisons la commande suivante :
```bash
ansible all -i setup.yml -m yum -a "name=httpd state=absent" --become
```

# Playbooks

## First playbook

> Fichier de configuration playbook.yml :
```yml
- hosts: all
  gather_facts: false
  become: yes

  tasks: 
    - name: Test connection
      ping:
    
```

## Advanced playbook

> Configuration du fichier playbook_docker.yml :
```yml
- hosts: all
  gather_facts: false
  become: yes
  
  # Install Docker
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

## Ansible-galaxy

> Création du groupe docker

Dans le fichier *playbook_docker.yml*, ajout des lignes suivantes :
```yml
roles: 
    - docker
```
Puis mettre les tasks dans le fichier main.yml, se trouvant dans le dossier /roles/docker/main.yml

# Deploy your app

>playbook_docker.yml 
```yml
- hosts: all
  gather_facts: false
  become: yes

  roles: 
    - docker
    - network
    - database
    - app
    - proxy
    - front


```

- Contenu du fichier main.yml pour le groupe *docker* :


```yml
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

- Création du groupe *network* et contenu du fichier main.yml du groupe *network* :
```yml
- name: Create a network
  docker_network:
     name: app-network
```

- Création du groupe *database* et contenu du fichier main.yml du groupe *database* :
```yml
- name: Run database
  docker_container:
    name: postgre
    image: bastienzanardocpe/postgre:1.0
    volumes:
      - /VolPostgre
    networks:
      - name: app-network
```

- Création du groupe *app* et contenu du fichier main.yml du groupe *app* :
> *Ce groupe correspond à l'application Backen API*
```yml
- name: Run API
  docker_container:
    name: backend
    image: bastienzanardocpe/backend:1.0
    networks:
      - name: app-network

```

- Création du groupe *proxy* et contenu du fichier main.yml du groupe *proxy* :
> *Ce groupe permet de faire la liaison entre le front et le backend API*
```yml
- name: Run httpd
  docker_container:
    name: httpd
    image: bastienzanardocpe/httpd:1.0
    ports:
      - "80:80"
    networks:
      - name: app-network

```

- Création du groupe *front* et contenu du fichier main.yml du groupe *front :
> *Ce groupe correspond à la page que nous verrons quand on appellera le serveur*
```yml
- name: Run front
  docker_container:
    name: front
    image: bastienzanardocpe/tp-devops-cpe:front
    networks:
      - name: app-network
    env:
      VUE_APP_API_URL: bastien.zanardo.takima.cloud
```

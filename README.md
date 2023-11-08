# TP part 01 - Docker - Juliette CRESPO
## Base de données
Nous créons un fichier Dockerfile. Ici, nous faisons fonctionner un simple serveur postgres :
```
FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
- L'instruction FROM définit l'image de base à utiliser pour construire notre image. Ici, nous utilisons l'image officielle de PostgreSQL dans sa version 14.1 mais avec la variante allégée alpine (cela permet d'avoir une empreinte mémoire plus réduite par rapport à des images classiques).
- L'instruction ENV permet de définir des variables d'environnement. Elles sont utilisées par l'image PostgreSQL pour créer une base de données avec un nom spécifié (db), un nom d'utilisateur (usr), et un mot de passe (pwd).

Nous créons un network : ```docker network create app-network```.
Ce network permet aux conteneurs Docker qui y sont attachés de communiquer entre eux, tout en les isolants des autres réseaux.

Nous redémarrons l'administrateur avec la commande suivante (cela nous permettra d'accéder à l'interface Adminer depuis l'hôte via le navigateur web):
```
docker run \
-p "8090:8080" \
--net=app-network \
--name=adminer \
-d \
adminer
```
Nous pouvons maintenant construire notre image : ```docker build -t jcrespota/db```.

Enfin, nous exécutons notre conteneur db sur le network creer plutot : ```docker run -p 8888:5000 --name db —network app-network jcrespota/db```.
#
Maintenant, nous voulons initiliser la base de données pour cela créons de deux scripts SQL.
Nous créons les deux fichiers ci-dessous et les plaçons dans un dossier db :

**01-CreateScheme.sql :**
```
CREATE TABLE public.departments
(
id SERIAL PRIMARY KEY,
name VARCHAR(20) NOT NULL
);
CREATE TABLE public.students
(
id SERIAL PRIMARY KEY,
department_id INT NOT NULL REFERENCES departments (id),
first_name VARCHAR(20) NOT NULL,
last_name VARCHAR(20) NOT NULL
);
```

**02-InsertData.sql :**
```
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');
INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```

Nous modifions le Dockerfile pour qu'il prenne en compte nos fichiers SQL :
```
FROM postgres:14.1-alpine
COPY /sql /docker-entrypoint-initdb.d
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
- Cette commande COPY est utilisée pour copier des fichiers depuis l'hôte vers l'image Docker. 

L'image a donc été modifier nous voulons ainsi la build de nouveau : ```docker build -t jcrespota/db```

Pour finir pouvons run notre conteneur :```docker run -p 8888:5000 --name db —network app-network jcrespota/db```

#
Si notre conteneur de base de données vient d'être détruit, toutes nos données sont réinitialisées. Une base de données se doit de conserver les données de manière durable.
Pour cela, nous utilisons des volumes pour conserver les données :
```
docker run -p 8888:5000 --name db -v /my/own/datadir:/var/lib/postgresql/data jcrespota/db
```
- Les volumes sont des mécanismes permettant de gérer et de stocker des données générées et utilisées par des conteneurs. Ils peuvent être utilisés pour partager des données entre des conteneurs ou pour stocker des données persistantes même si le conteneur est supprimé.

## API Backend

Pour commencer, nous exécuterons simplement une classe Java hello-world dans nos conteneurs. 
Nous créons le Dockerfile suivant :
```
FROM eclipse-temurin:8-jdk-alpine
COPY Main.java .
RUN javac Main.java
CMD ["java", “Main"]
```
Nous pouvons run et build notre image :
```
docker build -t jcrespota/backend
docker run -p 8091:8080 --name backend --network app-network jcrespota/backend
```
#
Nous déploierons une application Springboot fournissant une API simple avec un seul point de terminaison d'accueil. Nous créons ainsi une application Springboot sur : Spring Initializer.

Nous utilisons la configuration suivante :
<br>Projet : Maven<br>Langue : Java 17<br>Botte à ressort : 2.7.5<br>Emballage : Pot<br>Dépendances : Spring Web

Nous ajoutons ce nouveau Dockerfile à la racine du projet :
```
# Build / Étape de construction
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
# Utilise l'image de base Maven avec Amazon Corretto 17 pour construire l'application
ENV MYAPP_HOME /opt/myapp
# Définit la variable d'environnement MYAPP_HOME pointant vers le répertoire de l'application
WORKDIR $MYAPP_HOME
# Configure le répertoire de travail de l'image à $MYAPP_HOME
COPY pom.xml .
# Copie le fichier pom.xml dans le répertoire de travail
COPY src ./src
# Copie le répertoire src contenant le code source dans le répertoire de travail.
RUN mvn dependency:go-offline
# Télécharge les dépendances du projet et les rendre disponibles localement.
RUN mvn package -DskipTests
# Construit l'application, en sautant l'exécution des tests.

# Run / Exécution de l'application
FROM amazoncorretto:17
# Utilisz l'image Amazon Corretto 17 comme base pour exécuter l'application
ENV MYAPP_HOME /opt/myapp
# Redéfinit la variable d'environnement MYAPP_HOME
WORKDIR $MYAPP_HOME
# Configure le répertoire de travail de l'image à $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Copie le fichier JAR (résultat de la construction de l'étape précédente) vers le répertoire de travail de l'image d'exécution
ENTRYPOINT java -jar myapp.jar
# Définit la commande qui sera exécutée lorsque le conteneur sera démarré
```

Nous utilisons les constructions multi-étapes (multistage builds) pour optimiser les images Docker en séparant l'environnement de construction de l'image de production finale. Cela réduit la taille de l'image finale et élimine les dépendances inutiles.
#
Nous voulons maintenant construire et exécuté l'API d'arrière-plan connectée à la base de données. Nous téléchargeons le dossier simple api-student.

Nous complétons le fichier application.yml :
```
datasource:
url: jdbc:postgresql://mydatabase:5432/db
username: usr
password: pwd
driver-class-name: org.postgresql.Driver
```

Nous pouvons run et build notre image :
```
docker build -t jcrespota/backend
docker run -p 8091:8080 --name backend --network app-network jcrespota/backend
```

## Link application

**Docker-composer les commandes les plus importantes :**

- ```docker compose up``` : Lance tous les services définis dans le fichier docker-compose.yml. `-d` permet exécuter en arrière-plan.
- ```docker compose down```: Arrête et supprime les services définis dans le fichier docker-compose.yml.
- ```docker compose start```: Démarre les services.
- ```docker compose stop```: Arrête les services.
- ```docker compose ps```: Affiche l'état des services. `-a` permet d'afficher se qui ne sont pas actuellement en cours d'exécution.
- ```docker compose build```: Construit les images des services.
#
**docker-compose.yml**
```
version: '3.7'

services:
# Création du conteneur de la base de donnée en récupérent les variables secrétes du .env, puis connexion au network et au volume
  api:
    build:
      context: ./simpleapi
    container_name: api
    ports:
      - 8080:8080
    networks:
      - app-network
    depends_on:
      - db

# Création du conteneur de l'api (connecté au network) uniquement si la base de données a démarré 
  db:
    build:
      context: ./db
    container_name: db
    networks:
      - app-network

# Création du proxy (en indiquant le port et le network) uniquement si l'api a démarré
  frontend:
    build:
      context: ./frontend
    container_name: frontend
    ports:
      - 8070:80
    networks:
      - app-network
    depends_on:
      - api

# Création d'un network pour permettre aux conteneurs de communiquer 
networks:
  app-network:
```

## Publish

Nous allons publier nos images afin qu'elles puissent être utilisées par d'autres membres de l'équipe ou sur d'autres machines.

Dans un premier temps nous nous connectons au compte DockerHub : ```docker login```

Nous taggeons l’image : ```docker tag tp1-httpd jcrespota/tp1-frontend:1.0```

Nous publions l’image sur DockerHub : ```docker push jcrespota/tp1-frontend:1.0```

Nous fessons la même chose pour les deux autres images.

Nous pouvons visualiser les images sur le compte DockerHub :

<img width="934" alt="Capture d’écran 2023-11-08 à 14 56 58" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/82294e8d-9d2c-4fec-b4c5-ad7f3b1f811e">

# TP part 02 - Docker

**Testcontainers :**
Ce sont des bibliothèques qui facilitent l'utilisation de conteneurs Docker dans des test.
#


Dans un premier temps, il faut créer un repository GitHub :
```
git init
git remote add origin https://github.com/JulietteCrespo/tp_ci_cd.git
git push --set-upstream origin main
```
Nous ajoutons les dossiers suivant `.github/workflows` avec le fichier ci-dessus :
**main.yml**
```
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - 'main'
      - 'develop'
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 17

      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify -f ./simpleapi/pom.xml
```
Aprés avoir `git commit` et `git push`, nous pouvons voir que la pipeline a fonctionné :

<img width="1048" alt="Capture d’écran 2023-11-08 à 15 51 11" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/852364d1-b480-4be4-8c37-fc7bad477ed8">

#
Maintenant nous voulons construire une docker image dans la GitHub Actions pipeline. Pour cela on modifie le main.yml :
```
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - 'main'
      - 'develop'
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 17

      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify -f ./simpleapi/pom.xml
  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./simpleapi
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-01:latest

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./db
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-01:latest

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./frontend
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
```
Aprés avoir `git commit` et `git push`, nous pouvons voir que la pipeline a fonctionné :

<img width="1048" alt="Capture d’écran 2023-11-08 à 16 00 45" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/75fc0f43-8213-438a-84ef-f823bca192ba">

#

Maintenant nous voulons push nos images pour cela, nous modifions de nouveau le fichier main.yml:

```

name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - 'main'
      - 'develop'
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 17

      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify -f ./simpleapi/pom.xml
      # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ vars.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./simpleapi
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./db
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-db:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./frontend
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-frontend:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```


Aprés avoir `git commit` et `git push`, nous pouvons voir que la pipeline a fonctionné :

<img width="1048" alt="Capture d’écran 2023-11-08 à 16 05 25" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/ea2c95d2-9451-443f-827b-2310ee575056">

Nous avons ajouté nos identifiants de DockerHub au variables de GitHub Action ainsi notre image est publier sur DockerHub :

<img width="778" alt="Capture d’écran 2023-11-08 à 16 07 13" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/e8804861-54ce-4148-8933-5cbfe2cb6616">

<img width="930" alt="Capture d’écran 2023-11-08 à 16 11 37" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/76918a4f-c347-48fa-96fd-ffa3eb4d7069">

#

Nous voulons utiliser SonarCloud. Pour cela, nous créons un compte Sonar Cloud.

Puis nous avons récupéré la project key & organization key et nous avons ajouté aux variables de GitHub Action :

<img width="779" alt="Capture d’écran 2023-11-08 à 16 20 16" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/256f799e-c072-41ff-8b44-d98ce4cf431a">


Nous modifions de nouveau le fichier main.yml:
```
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - 'main'
      - 'develop'
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: ./simpleapi
    steps:
      #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 17

      #finally build your app with the latest command
      - name: Build and test with Maven
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=JulietteCrespo_tp_ci_cd  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ vars.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./simpleapi
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./db
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-db:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./frontend
          # Note: tags has to be all lower-case
          tags: ${{vars.DOCKERHUB_USERNAME}}/tp-01-frontend:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

Maintenant nous pouvons utiliser SonarCloud :

<img width="1386" alt="Capture d’écran 2023-11-08 à 16 24 46" src="https://github.com/JulietteCrespo/tp_ci_cd/assets/73819396/8776cab3-0d41-41d0-aae2-fa930c94341c">


# TP part 03 - Ansible

Nous créons les dossiers suivant `.ansible/inventories` avec le fichier ci-dessus :

**setup.yml**
```
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ../../id_rsa
 children:
   prod:
     hosts: marion.chineaud.takima.cloud
```

Nous utilisons `ansible all -i inventories/setup.yml -m ping` pour tester la connectivité

Nous utilise le module "setup" pour collecter des informations sur les hôtes, en filtrant spécifiquement les informations liées à la distribution du système :
`ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` 

#

Nous créons par la suite un fichier playbooks.yml dans le dossier .ansible/inventories. Il vas permettre d'utiliser différent roles :

**playbooks.yml**
```
- hosts: all # definit les hôtes
  gather_facts: false
  become: yes # désactive la collecte des facts
  roles: # rôles Ansible à exécuter dans un ordre précis
    - docker
    - networks
    - database
    - api
    - proxy
```

On exécute le playbook : 
`ansible-playbook -i inventories/setup.yml playbook.yml`

#
Maintenant, nous déployons notre application avec les rôles suivants :

- installer docker :
  **docker/main.yml**
  ```
      - name: Clean packages
        command:
          cmd: yum clean -y packages
      
      - name: Install device-mapper-persistent-data
        yum:
          name: device-mapper-persistent-data
          state: latest
      
      - name: Install lvm2
        yum:
          name: lvm2
          state: latest
      
      - name: add repo docker
        command:
          cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
      
      - name: Install Docker
        yum:
          name: docker-ce
          state: present
      
      - name: Make sure Docker is running
        service: name=docker state=started
        tags: docker
  ```
- créer un réseau :
  **network/main.yml**
  ```
    - name: Creating a Docker network
      docker_network:
        name: app-network
  ```
- base de données de lancement :
  **database/main.yml**
  ```
    - name: Run db
      docker_container:
        name: db
        image: jcrespota/tp-01-db:latest
        env:
          POSTGRES_DB: "db"
          POSTGRES_USER: "usr"
          POSTGRES_PASSWORD: "pwd"
        networks:
          - name: app-network
  ```
- lancer l'application :
  **app/main.yml**
  ```
    - name: Run backend
        docker_container:
          name: api
          image: jcrespota/tp-01-backend:latest
          networks:
            - name: app-network
          ports: "8080:8080"
  ```
- proxy de lancement :
    **proxy/main.yml**
  ```
    - name: Run frontend
      docker_container:
        name: frontend
        image: jcrespota/tp-01-frontend:latest
        networks:
          - name: app-network
        ports: "80:80"
  ```

Les différents rôles sont lancer grace au fichier playbooks.yml.












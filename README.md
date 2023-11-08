# TP part 01 - Docker
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

  db:
    build:
      context: ./db
    container_name: db
    networks:
      - app-network

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



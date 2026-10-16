# TP Docker - Partie 1

Application 3-tiers conteneurisée avec Docker, réalisée dans le cadre du cours DevOps (EFREI, S7) :

- **Database** : PostgreSQL 17.2 (Alpine), initialisée par des scripts SQL
- **Backend** : API Spring Boot (Java 21), construite avec un Dockerfile multistage
- **HttpServer** : Apache 2.4 (Alpine), utilisé comme reverse proxy devant le backend

L'ensemble est orchestré avec Docker Compose.

## Structure du projet

```
TP1/
├── Database/
│   ├── Dockerfile
│   └── init/
│       ├── 01-CreateScheme.sql
│       └── 02-InsertData.sql
├── Backend/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── HttpServer/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── docker-compose.yml
└── .env              (non versionné)
```

## Prérequis

- Docker
- Docker Compose (commande `docker compose`)

Java et Maven ne sont pas nécessaires sur la machine : le backend est compilé dans le conteneur.

## Lancement rapide

1. Créer un fichier `.env` à la racine de `TP1` :

```env
POSTGRES_DB=db
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
```

2. Construire et lancer l'application :

```bash
docker compose up -d --build
```

3. Tester l'API via le reverse proxy :

```
http://localhost/departments/IRC/students
```

Seul le port 80 (Apache) est exposé sur la machine hôte. Le backend et la base ne sont joignables que par le réseau interne `app-network`.

## Détail des services

### Database

Image `postgres:17.2-alpine`. Au premier démarrage, Postgres exécute les scripts placés dans `/docker-entrypoint-initdb.d`, par ordre alphabétique :

- `01-CreateScheme.sql` : crée les tables `departments` et `students`
- `02-InsertData.sql` : insère les données initiales

```dockerfile
FROM postgres:17.2-alpine
COPY init/ /docker-entrypoint-initdb.d/
```

Les identifiants ne sont pas écrits dans le Dockerfile : ils sont passés au lancement (`-e` ou fichier `.env`) pour garder les secrets hors du code et garder une image réutilisable. Un volume persiste les données au-delà du cycle de vie du conteneur.

Lancement manuel (sans Compose) :

```bash
docker network create app-network
docker build -t my-database ./Database
docker run --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v $(pwd)/data:/var/lib/postgresql/data -d my-database

# Adminer pour consulter la base : http://localhost:8090
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

### Backend

API Spring Boot basée sur [simple-api-student](https://github.com/takima-training/simple-api-student). Le Dockerfile multistage compile le projet avec le JDK et Maven, puis ne garde que le jar dans une image JRE plus légère :

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
RUN apk add --no-cache maven
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

La connexion à la base est configurée dans `src/main/resources/application.yml` avec des variables d'environnement :

```yaml
datasource:
  url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
  username: ${DB_USER}
  password: ${DB_PASSWORD}
  driver-class-name: org.postgresql.Driver
```

Lancement manuel :

```bash
docker build -t simple-api ./Backend
docker run --name simple-api --network app-network -p 8080:8080 \
  -e DB_HOST=my-database -e DB_NAME=db \
  -e DB_USER=usr -e DB_PASSWORD=pwd \
  -d simple-api
```

### HttpServer

Image `httpd:2.4-alpine`, qui sert une page `index.html` et fait office de reverse proxy vers le backend. Les modules `mod_proxy` et `mod_proxy_http` sont activés dans `httpd.conf`, avec cette configuration ajoutée en fin de fichier :

```apache
ServerName localhost
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://simple-api:8080/
    ProxyPassReverse / http://simple-api:8080/
</VirtualHost>
```

```dockerfile
FROM httpd:2.4-alpine
COPY index.html /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

Le reverse proxy donne un point d'entrée unique à l'application, évite d'exposer le backend directement et pourrait aussi gérer le HTTPS ou la répartition de charge.

## Docker Compose

Choix faits dans `docker-compose.yml` :

- **Un seul port exposé** : seul `httpd` publie le port 80
- **Healthcheck** sur Postgres (`pg_isready`) avec `condition: service_healthy` : le backend attend que la base accepte vraiment les connexions avant de démarrer
- **`restart: unless-stopped`** : redémarrage automatique en cas d'erreur, sauf arrêt volontaire
- **Volume nommé `db-data`** : géré par Docker, plus portable qu'un chemin de la machine hôte
- **Secrets dans `.env`** : lu automatiquement par Compose et non versionné

### Commandes utiles

| Commande | Effet |
|---|---|
| `docker compose up -d --build` | Construit les images et lance tous les services |
| `docker compose ps` | Affiche l'état des services |
| `docker compose logs -f backend` | Suit les logs d'un service |
| `docker compose stop` / `start` | Arrête ou relance sans rien supprimer |
| `docker compose down` | Supprime conteneurs et réseau (volumes conservés) |
| `docker compose down -v` | Supprime aussi les volumes, donc les données |

## Points notables

- **Compatibilité Java** : pour l'étape Hello World, compiler avec `javac --release 21 Main.java` si le JDK local est plus récent que le JRE 21 du conteneur. Sinon, le lancement échoue avec `UnsupportedClassVersionError`.
- **Consommation** : `docker stats` montre environ 6 Mo de RAM pour Apache contre près de 300 Mo pour le backend Spring Boot, à cause de la JVM.

## Auteur

Armand Labernardière - EFREI, 4e année

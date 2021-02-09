# Docker Guide

Ziel dieses Repositories ist es die Grundlagen von Docker und Containern theoretisch und mit praktischen Beispielen zu erklären. Weiters will ich hier dokumentieren wie man ein Development Environment mit Docker und Docker-Compose erstellen kann. Weil ich bis jetzt noch keine geschriebenes Tutorial gefunden habe, dass Docker so erklärt wie ich es für optimal halte schreibe ich hier mein eigenes.

## Docker Grundlagen

Docker ist ein System dass es erlaubt Software in Container zu verlegen. Der Vorteil von Containern ist, dass diese immer gleich bleiben, egal auf welcher Maschine sie laufen. Um Docker zu verstehen muss man den Bezug zwischen Dockerfiles, Images und Containern verstehen. 

## Dockerfile

Im Dockerfile wird beschreiben wie ein Docker-Image aufbebaut werden soll. Man kann es als Grobrezept zur erstellung eines Containers umschreiben. Jedes Dockerfile startet damit, dass angegeben wird welches Image als Basis verwendet wird. Solche Images beinhalten meißtens eine Installation der Tools die benötigt werden um Code zu kompilieren und auszuführen. Es gibt für so gut wie alle Programmiersprachen offizielle Docker-Images die von den Entwicklern zur Verfügung gestellt werden. Um Images zu finden empfehle ich [DockerHub](https://hub.docker.com/) zu verwenden. Dockerhub wird standardmäßig von Docker verwendet um Images herunterzuladen. Es ist also ähnlich wie NPM zu NodeJS. Wenn man ein komplett neues Image erstellen will kann man auch ein Linux Betriebssytem wie Ubuntu als Basis verwenden. Um jetzt ein Image zu erstellen muss man in einer Datei namens `Dockerfile` (ohne Dateityp) mit der Definition der Basis beginnen:

```dockerfile
FROM node
```

Als nächstes würde ich empfehlen das `Working Directory`, also den Ort an dem der Code gespeichert wird, festzulegen. Wo der Code gespeichert wird macht keinen Unterschied, und man könnte ihn theoretisch im root Verzeichniss speichern aber das ist nicht empfohlen. Hier wird `/usr/src/myapp` als Speicherort verwendet.

```dockerfile
WORKDIR /usr/src/myapp
```

Bevor man jetzt den Code hinüberkopiert, wird empfohlen die benötigten Module zu installieren. Da in diesem Beispiel NodeJS verwendet wird, wird hier das `package.json` und `package-lock.json` kopiert und der Befehl `npm install` ausgeführt. Bei python wäre dies das `requirements.txt` und `pip install -r requirements.txt`. Um eine Datei oder einen Ordner zu kopieren wird der Befehl `COPY` verwendet, bei dem das der Ausgangs- und Zielpfad definiert werden. Wenn man in einem Dockerfile einen Kommandozeilen Befehl ausführen will kann `RUN` verwendet werden:

```dockerfile
COPY package*.json ./
RUN npm install --only=production --quiet
```

Jetzt kann der restliche Code kopiert werden: 
```dockerfile
COPY . .
```

Falls das im Container ausgeführte Programm Netzwerk konnektivität erfordert, kann mit dem `EXPOSE` Befehl angegeben werden welche Ports das Programm benötigt. Hier wird der Standardmäßige HTTP Port 80 verwendet:

```dockerfile
EXPOSE 80
```

Zu guter letzt muss der Code der im Container auch ausgeführt werden. Dazu wird jedoch nicht `RUN` sondern `CMD` verwendet. Im unterschied zu `RUN` wird `CMD` nicht nur ausgeführt wenn das Image erstellt wird, sondern ist der Befehl der Standardmäßig beim erstellen eines Containers ausgeführt wird.

```dockerfile
CMD ["npm", "start"]
```

Das fertige Dockerfile sollte in etwa so aussehen:

```Dockerfile
FROM node
WORKDIR /code
COPY package*.json ./
RUN npm install --only=production --quiet
COPY . .
EXPOSE 80
CMD ["npm", "start"]
```

### Images
Wenn ein Dockerfile ein Grobrezept ist, ist ein Image ein mehrbändiges Handbuch zur Erstellung eines Containers. Ein Image beinhaltet alle Dateien, die zum starten eines Containers benötigt werden. Um aus einem 'abstraktem' Dockerfile in ein Image zu bauen wird der `docker build` Befehl verwendet. Dieser Befehl sucht im aktuellen Verzeichniss nach einem Dockerfile und baut daraus ein Image. Wenn man diesem Image einen Namen geben will kann man den `-t` Parameter verwenden. Ein Befehl zum erstellen eines Images könnte also so aussehen:
```
docker build -t rwutscher/node-exapmle .
```

### Container
Sobald man ein Image hat kann man daraus Container erstellen. Dafür wird der `docker run` Befehl verwendet. Um einen Container des gerade eben erstellten Image zu starten würde der Befehl so aussehen:
```
docker run -p 5000:80 rwutscher/node-exapmle -d
```
Dieser Befehl hat 2 Parameter:

|Parameter|Beschreibung|
|---|---|
|-p|Definiert auf welchen Port der Host-Maschine (5000) der Port des Containers(80) weitergeleitet werden soll|
|-d|Startet den Container als Hintergrundprozess|

### Volumes
Volumes können verwendet werden um Code zwischen Host und Container zu teilen. Dieses Feature ist beim Entwickeln von Software unabdingbar, da man seine Software in einem Docker Container ausführen kann ohne das Image neu zu erstellen.

## Docker-Compose
Wozu braucht man jetzt Docker-Compose? Kurzgesagt macht Docker-Compose den Umgang mit meherern Containern und das Entwickeln von Containerbasierten Programmen einfacher.

Mit Docker-Compose kann man schnell mehrere Container in einen Service zusammenfassen und die Startparameter dieser Container bestimmen.

Um Docker-Compose zu verwenden muss man zuerst eine `docker-compose.yml` Datei anlegen. 

In dieser Datei muss man zunächst die Version von Docker-Compose, die verwendet wird, angeben. Dies ist wichtig, da sich die Syntax zwischen den Versionen stark verändern kann.
```yml
version: '3.9'
```

Wenn die Version angegeben ist, kann man unter `services` die verschiedenen Docker Apps die erstellt werden sollen angeben. Hier werden das vorher erstellte Dockerimage und ein MySQL-Server verwendet.

```yml
services:
  nodeApp:
    ...
  mysql:
    ...
```

Bei selbst erstellten Images gibt es 3 wichtige parameter:

|Parameter|Beschreibung|
|-|-|
|build|Der Pfad zum Ordner in dem das Dockerfile gespeichert ist|
|ports|Das Portforwarding des Containers|
|volumes|Die Volumes die für den Container verwendet werden sollen (siehe Volumes). Ausgangs- und Zielverzeichniss werden durch einen `:` getrennt|

Es kann auch ein `command` definiert werden, der statt dem standardmäßigen `CMD` im Dockerfile ausgeführt wird.

```yml
build: .
ports:
  - "5000:80"
volumes:
  - .:/code
```

Um ein standardmäßiges MySQL Image zu verwenden wird diese Konfiguration, die ich auf der [DockerMySQL](https://docs.docker.com/samples/library/mysql/) Seite gefunden habe, verwendet.

```yml
image: mysql
command: --default-authentication-plugin=mysql_native_password
restart: always
environment:
    MYSQL_ROOT_PASSWORD: example
```

Die fertige docker-compose.yml Datei sollte cirka so aussehen:

```yml
version: '3'
services:
  nodeApp:
    build: .
    ports:
     - "5000:80"
    volumes:
     - .:/code
  mysql:
    image: mysql
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
        MYSQL_ROOT_PASSWORD: example
```

# //TODO

## Docker installieren

### Linux

### Windows

#### Docker-Machine

### Docker-Compose

## Beispiel

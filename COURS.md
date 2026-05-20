# Cours complet — Docker

---

# PARTIE 1 — THÉORIE

---

## 1. Qu'est-ce que Docker ?

Une application a besoin d'un environnement précis pour tourner : une version spécifique de Python, des bibliothèques installées, des variables d'environnement configurées. Le problème classique : "ça marche sur ma machine, mais pas sur le serveur".

Docker résout ce problème en **empaquetant l'application avec tout son environnement** dans une unité portable appelée un **conteneur**.

> Analogie : un conteneur Docker c'est comme un container de transport maritime. Peu importe le bateau (ton ordinateur, un serveur, un cloud), le contenu à l'intérieur reste intact et identique.

Docker n'est pas une machine virtuelle. Une VM émule un ordinateur entier avec son propre OS. Un conteneur Docker partage le noyau (kernel) de la machine hôte et n'embarque que ce qui est strictement nécessaire à l'application. Résultat : beaucoup plus léger et beaucoup plus rapide à démarrer.

### VM vs Conteneur

| | Machine Virtuelle | Conteneur Docker |
|---|---|---|
| OS complet | Oui (plusieurs Go) | Non (partage le kernel) |
| Démarrage | Minutes | Secondes |
| Isolation | Totale | Processus isolé |
| Taille | Plusieurs Go | Quelques Mo à quelques centaines de Mo |
| Usage | Simuler un OS entier | Empaqueter une application |

---

## 2. Les concepts fondamentaux

### Image Docker

Une **image** est un modèle figé, en lecture seule, qui contient tout ce dont une application a besoin : l'OS de base, les dépendances installées, le code, la configuration.

> Analogie : une image c'est une recette de cuisine. Elle décrit exactement comment préparer le plat, mais elle n'est pas le plat elle-même.

Une image est construite à partir d'un `Dockerfile` avec la commande `docker build`.

### Conteneur Docker

Un **conteneur** est une instance en cours d'exécution d'une image. On peut créer plusieurs conteneurs à partir de la même image.

> Analogie : si l'image est la recette, le conteneur est le plat préparé. Tu peux préparer dix plats à partir de la même recette — ce sont dix conteneurs différents issus de la même image.

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Conteneur (en cours d'exécution)
```

### Dockerfile

Un `Dockerfile` est un fichier texte qui contient une série d'instructions pour construire une image. Chaque instruction crée une nouvelle **couche** (layer) dans l'image.

Les couches sont mises en cache par Docker. Si une instruction n'a pas changé depuis le dernier build, Docker réutilise la couche en cache au lieu de tout reconstruire. C'est pourquoi les builds suivants sont beaucoup plus rapides.

---

## 3. Les instructions Dockerfile essentielles

### `FROM`

```dockerfile
FROM ubuntu:latest
```

Définit l'image de base sur laquelle on construit. Toujours la première instruction. `ubuntu:latest` signifie la dernière version stable d'Ubuntu disponible sur Docker Hub.

### `RUN`

```dockerfile
RUN apt-get update
RUN apt-get upgrade -y
```

Exécute une commande **au moment du build** (pas au démarrage du conteneur). Chaque `RUN` crée une nouvelle couche. Le résultat de la commande est "gravé" dans l'image.

Le flag `-y` répond automatiquement "oui" à toutes les questions d'`apt-get`. Sans lui, `apt-get` attend une confirmation de l'utilisateur et le build se bloque.

### `WORKDIR`

```dockerfile
WORKDIR /app
```

Définit le répertoire de travail pour toutes les instructions suivantes (`COPY`, `RUN`, `CMD`...). Si le répertoire n'existe pas, il est créé automatiquement. C'est l'équivalent d'un `cd /app` persistant dans l'image.

### `COPY`

```dockerfile
COPY ./api.py /app/api.py
```

Copie des fichiers depuis ta machine locale vers l'image Docker. Le premier chemin est relatif au **contexte de build** (le dossier passé à `docker build`). Le second chemin est la destination dans l'image.

### `CMD`

```dockerfile
CMD ["echo", "Hello, World!"]
CMD ["python3", "api.py"]
```

Définit la commande exécutée **au démarrage du conteneur** (pas au build). Il ne peut y avoir qu'une seule instruction `CMD`. La syntaxe en tableau (format exec) est préférable à la syntaxe en chaîne.

**Différence clé entre `RUN` et `CMD` :**
- `RUN` = s'exécute pendant `docker build` → modifie l'image
- `CMD` = s'exécute pendant `docker run` → démarre le processus du conteneur

### `EXPOSE`

```dockerfile
EXPOSE 5252
```

Documente que le conteneur écoute sur ce port. C'est **informatif uniquement** — ça ne publie pas automatiquement le port sur la machine hôte. La publication se fait avec `-p` lors du `docker run`.

---

## 4. Les commandes Docker essentielles

### `docker build`

```bash
docker build -f ./Dockerfile -t softy-pinko:task0 .
```

Construit une image à partir d'un Dockerfile.

| Flag | Description |
|---|---|
| `-f ./Dockerfile` | Chemin vers le Dockerfile (optionnel si c'est `./Dockerfile`) |
| `-t softy-pinko:task0` | Nom et tag de l'image (`nom:tag`) |
| `.` | Le contexte de build — le dossier dont les fichiers seront accessibles pendant le build |

### `docker run`

```bash
docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1
```

Crée et démarre un conteneur à partir d'une image.

| Flag | Description |
|---|---|
| `-p 5252:5252` | Mappe le port hôte:port conteneur |
| `-it` | Mode interactif avec terminal (pour voir les logs en direct) |
| `--rm` | Supprime automatiquement le conteneur quand il s'arrête |
| `--name softy-pinko-task1` | Donne un nom au conteneur |

### Le mapping de ports `-p`

```bash
-p 9000:9000
# hôte:conteneur
```

Le conteneur est isolé — ses ports ne sont pas accessibles depuis l'extérieur par défaut. Le flag `-p` crée un "tunnel" : les requêtes qui arrivent sur le port 9000 de ta machine sont redirigées vers le port 9000 du conteneur.

> Analogie : le conteneur est dans une pièce fermée. `-p 9000:9000` c'est ouvrir une fenêtre : tout ce qui arrive par la fenêtre 9000 de la maison (ton ordinateur) entre par la fenêtre 9000 de la pièce (le conteneur).

---

## 5. Nginx — Serveur web statique

Nginx (prononcé "engine-x") est un serveur web très performant. Dans ce projet, on l'utilise pour servir les fichiers statiques du front-end (HTML, CSS, JS, images).

L'image Docker officielle `nginx:latest` est préconfigurée : elle démarre automatiquement Nginx au lancement du conteneur. On n'a pas besoin d'instruction `CMD`.

### Fichier de configuration Nginx

Nginx lit sa configuration dans `/etc/nginx/conf.d/default.conf`. En copiant notre propre fichier à cet emplacement dans l'image, on écrase la config par défaut.

```nginx
server {
    listen 9000;
    server_name _;

    location / {
        root /var/www/html/softy-pinko-front-end;
        index index.html;
    }
}
```

**`listen 9000`** — Nginx écoute sur le port 9000. Ce port doit correspondre au port mappé avec `-p` dans `docker run`.

**`server_name _`** — Accepte les requêtes peu importe le nom de domaine utilisé. Le `_` est un wildcard.

**`location /`** — Définit comment traiter les requêtes qui commencent par `/` (donc toutes les requêtes).

**`root`** — Le répertoire où chercher les fichiers. Si l'URL est `/`, Nginx cherche dans ce dossier.

**`index index.html`** — Fichier servi par défaut quand on demande un répertoire.

---

## 6. Flask — Serveur back-end Python

Flask est un micro-framework web Python. Il permet de créer une API HTTP avec très peu de code.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/api/hello')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5252)
```

**`host='0.0.0.0'`** — Écoute sur toutes les interfaces réseau, pas seulement `127.0.0.1` (localhost). C'est indispensable dans un conteneur Docker : sans ça, l'API n'est accessible que depuis l'intérieur du conteneur lui-même.

> Analogie : `127.0.0.1` c'est une adresse qui ne fonctionne que si tu es "dans la pièce". `0.0.0.0` c'est "tout le monde peut frapper à cette porte".

**`port=5252`** — Le port sur lequel Flask écoute. Doit correspondre au port mappé dans `docker run -p 5252:5252`.

### CORS — Cross-Origin Resource Sharing

Par défaut, un navigateur bloque les requêtes JavaScript vers un domaine/port différent de celui de la page. C'est une règle de sécurité appelée **Same-Origin Policy**.

Dans notre cas : la page est servie depuis `localhost:9000` (front-end) et fait une requête vers `localhost:5252` (back-end). Ports différents = origines différentes = bloqué par défaut.

**Flask-CORS** ajoute les en-têtes HTTP nécessaires pour dire au navigateur "c'est OK, cette origine est autorisée".

```python
from flask_cors import CORS
CORS(app)  # autorise toutes les origines
```

---

## 7. Docker Compose

Gérer plusieurs conteneurs manuellement (un `docker build` et un `docker run` pour chaque) devient vite fastidieux. **Docker Compose** permet de définir et démarrer plusieurs services en un seul fichier et une seule commande.

```yaml
version: "3"

services:
  back-end:
    build: ./back-end
    ports:
      - "5252:5252"

  front-end:
    build: ./front-end
    ports:
      - "9000:9000"
```

### Commandes Docker Compose

```bash
docker-compose up          # construit et démarre tous les services
docker-compose up --build  # force la reconstruction des images
docker-compose down        # arrête et supprime les conteneurs
```

**`build: ./back-end`** — indique à Compose de construire l'image à partir du `Dockerfile` dans le dossier `./back-end`.

**`ports`** — équivalent du `-p` de `docker run`.

**Réseau automatique** — Docker Compose crée automatiquement un réseau privé entre tous les services. Les services peuvent se parler en utilisant leur nom comme hostname. Par exemple, le service `back-end` est accessible depuis le service `front-end` via `http://back-end:5252`.

---

## 8. Reverse Proxy et Load Balancer

### Proxy direct vs Reverse Proxy

Un **proxy direct** se place entre le client et internet. C'est le client qui configure le proxy — il envoie ses requêtes au proxy, qui les transmet vers internet.

Un **reverse proxy** se place devant les serveurs. C'est transparent pour le client — il ne sait pas qu'il y a un proxy. Le reverse proxy reçoit toutes les requêtes et décide à quel serveur les envoyer.

```
                         ┌─────────────────────────────┐
                         │        Reverse Proxy         │
Client  ──── requête ───►│  port 80                     │──► front-end (port 9000)
                         │  (point d'entrée unique)     │──► back-end  (port 5252)
                         └─────────────────────────────┘
```

**Avantages du reverse proxy :**
- Un seul point d'entrée pour le client
- Les serveurs internes ne sont pas directement exposés
- Peut gérer SSL, la compression, le cache...
- Peut distribuer la charge entre plusieurs serveurs (load balancing)

### Load Balancing — Répartition de charge

Quand le trafic est élevé, un seul serveur peut être insuffisant. On lance plusieurs instances du même serveur et le load balancer distribue les requêtes entre elles.

**Round Robin** — L'algorithme le plus simple. Les requêtes sont distribuées à tour de rôle :
- Requête 1 → serveur A
- Requête 2 → serveur B
- Requête 3 → serveur A
- Requête 4 → serveur B
- ...

Chaque serveur reçoit exactement le même nombre de requêtes.

### Configuration Nginx comme reverse proxy

```nginx
upstream back-end {
    server back-end1:5252;
    server back-end2:5252;
}

server {
    listen 80;

    location /api {
        proxy_pass http://back-end;
    }

    location / {
        proxy_pass http://front-end:9000;
    }
}
```

**`upstream`** — Définit un groupe de serveurs. Nginx applique automatiquement Round Robin quand plusieurs serveurs sont listés.

**`proxy_pass`** — Redirige la requête vers l'URL indiquée. Le client ne voit jamais cette redirection.

**`location /api`** — Les requêtes dont l'URL commence par `/api` vont vers le back-end.

**`location /`** — Toutes les autres requêtes vont vers le front-end.

**L'ordre des `location` est important** — Nginx teste les règles du plus spécifique au moins spécifique. `/api` est testé avant `/`.

---

## 9. Réseau Docker et communication entre conteneurs

Par défaut, les conteneurs Docker sont isolés. Ils ne peuvent pas se parler directement.

**Solution 1 — ports exposés sur la machine hôte :** les conteneurs communiquent via `localhost` en passant par la machine hôte. Simple mais les ports doivent être publiés.

**Solution 2 — réseau Docker personnalisé :** les conteneurs sur le même réseau Docker peuvent se parler directement par leur nom. Plus propre, pas besoin d'exposer les ports à la machine hôte.

Docker Compose crée automatiquement ce réseau. Dans un `docker-compose.yml`, les services se voient par leur nom.

```yaml
# Le service "proxy" peut parler à "back-end1" via http://back-end1:5252
# sans que ce port soit exposé sur la machine hôte
```

---

# PARTIE 2 — WALKTHROUGH DES TÂCHES

---

## Tâche 0 — Créer sa première image Docker

**Objectif :** Créer un Dockerfile basé sur Ubuntu qui affiche "Hello, World!" au démarrage du conteneur.

### Structure attendue

```
task0/
└── Dockerfile
```

### Concepts clés

- `FROM ubuntu:latest` — image de base
- `RUN apt-get update` — met à jour la liste des paquets (au build)
- `RUN apt-get upgrade -y` — met à jour les paquets installés (au build)
- `CMD ["echo", "Hello, World!"]` — commande exécutée au démarrage du conteneur

### Commandes pour tester

```bash
docker build -f ./Dockerfile -t softy-pinko:task0 .
docker run -it --rm --name softy-pinko-task0 softy-pinko:task0
# Doit afficher : Hello, World!
```

### Questions fréquentes

**Pourquoi `apt-get update` avant `apt-get upgrade` ?**
`apt-get update` télécharge la liste des paquets disponibles depuis les dépôts. Sans cette étape, `apt-get upgrade` ne saurait pas quelles nouvelles versions existent. Ces deux commandes vont toujours ensemble.

**Pourquoi deux instructions `RUN` séparées et pas une seule ?**
Chaque `RUN` crée une couche. Si on modifie `apt-get upgrade` mais pas `apt-get update`, Docker peut réutiliser la couche `update` depuis le cache. En pratique pour des raisons de cache on peut les séparer, mais on peut aussi les chaîner avec `&&` pour réduire le nombre de couches.

**Pourquoi `CMD ["echo", "Hello, World!"]` et pas `RUN echo "Hello, World!"` ?**
`RUN` s'exécute pendant la construction de l'image — son effet est gravé dans l'image. `CMD` s'exécute à chaque démarrage du conteneur. On veut afficher "Hello, World!" quand on lance le conteneur, pas pendant le build.

---

## Tâche 1 — Back-end Flask

**Objectif :** Créer une image Docker avec Python, pip et Flask, qui démarre un serveur API avec un endpoint `/api/hello`.

### Structure attendue

```
task1/
├── Dockerfile
└── api.py
```

### Concepts clés

- Installer des paquets système avec `apt-get install -y`
- Installer des paquets Python avec `pip3 install`
- `WORKDIR /app` — définit le répertoire de travail
- `COPY ./api.py /app/api.py` — copie le code dans l'image
- `CMD ["python3", "api.py"]` — démarre Flask au lancement
- `-p 5252:5252` — expose le port Flask sur la machine hôte

### Le problème `EXTERNALLY-MANAGED`

Sur les versions récentes d'Ubuntu, Python est "géré de l'extérieur" (par le système). `pip3 install` peut refuser de s'exécuter avec l'erreur `This environment is externally managed`. La solution : supprimer le fichier qui impose cette restriction **avant** d'appeler `pip3`.

```dockerfile
RUN rm /usr/lib/python*/EXTERNALLY-MANAGED
```

### Pourquoi `host='0.0.0.0'` dans api.py ?

Sans `0.0.0.0`, Flask écoute sur `127.0.0.1` — l'adresse de loopback, accessible seulement depuis l'intérieur du conteneur. Avec `0.0.0.0`, Flask écoute sur toutes les interfaces, y compris celle qui est connectée à la machine hôte via le mapping de ports.

### Commandes pour tester

```bash
docker build -f ./Dockerfile -t softy-pinko:task1 .
docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1
# Dans le navigateur : http://localhost:5252/api/hello
# Doit afficher : Hello, World!
```

### Questions fréquentes

**Pourquoi `pip3` et pas `apt-get install python3-flask` ?**
La version de Flask dans les dépôts APT est souvent ancienne. `pip3` installe la dernière version directement depuis PyPI (le dépôt officiel Python).

**Pourquoi `WORKDIR /app` avant `COPY` ?**
`WORKDIR /app` crée le répertoire `/app` et s'y positionne. Le `COPY ./api.py /app/api.py` copie ensuite le fichier dans ce répertoire. `CMD ["python3", "api.py"]` s'exécutera depuis `/app`, donc Python trouvera `api.py` dans le répertoire courant.

---

## Tâche 2 — Front-end Nginx

**Objectif :** Créer un conteneur Nginx qui sert les fichiers statiques du front-end sur le port 9000.

### Structure attendue

```
task2/
├── back-end/
│   ├── Dockerfile
│   └── api.py
└── front-end/
    ├── Dockerfile
    ├── softy-pinko-front-end.conf
    └── softy-pinko-front-end/   ← cloné depuis GitHub
        ├── index.html
        └── ...
```

### Le Dockerfile front-end

Contrairement au back-end, on utilise `nginx:latest` comme image de base. Nginx est déjà installé et configuré pour démarrer automatiquement — pas besoin de `CMD`.

```dockerfile
FROM nginx:latest
COPY ./softy-pinko-front-end /var/www/html/softy-pinko-front-end
COPY ./softy-pinko-front-end.conf /etc/nginx/conf.d/default.conf
```

### Le fichier de configuration Nginx

```nginx
server {
    listen 9000;
    server_name _;

    location / {
        root /var/www/html/softy-pinko-front-end;
        index index.html;
    }
}
```

**Pourquoi copier vers `/etc/nginx/conf.d/default.conf` ?**
C'est l'emplacement que Nginx charge automatiquement au démarrage. En remplaçant ce fichier, on impose notre configuration à la place de celle par défaut.

### Commandes pour tester

```bash
# Clone le front-end d'abord (dans task2/front-end/)
git clone https://github.com/atlas-school/softy-pinko-front-end

docker build -f ./front-end/Dockerfile -t softy-pinko-front-end:task2 ./front-end
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task2 softy-pinko-front-end:task2
# Dans le navigateur : http://localhost:9000
```

### Questions fréquentes

**Pourquoi `server_name _` ?**
`_` est un nom de serveur invalide qui sert de "catch-all". Nginx acceptera les requêtes peu importe le nom d'hôte utilisé (localhost, 127.0.0.1, une IP...).

**Pourquoi le contexte de build est `./front-end` et non `.` ?**
`docker build -f ./front-end/Dockerfile -t ... ./front-end` — le dernier argument est le contexte. En mettant `./front-end`, seuls les fichiers de ce dossier sont accessibles pour le `COPY`. C'est pour que `COPY ./softy-pinko-front-end ...` pointe vers `task2/front-end/softy-pinko-front-end`.

---

## Tâche 3 — Connecter le front-end et le back-end

**Objectif :** Faire communiquer les deux conteneurs — le front-end fait une requête AJAX vers le back-end et affiche la réponse dynamiquement.

### Modifications nécessaires

**1. Dans `index.html`** — ajouter un élément HTML pour afficher le contenu dynamique, et un script JavaScript pour faire la requête :

```html
<h1 id="dynamic-content"></h1>
```

```html
<script>
    $(function() {
        $.ajax({
            type: "GET",
            url: "http://localhost:5252/api/hello",
            success: function(data) {
                $('#dynamic-content').text(data);
            }
        });
    });
</script>
```

**2. Dans `api.py`** — ajouter Flask-CORS pour autoriser les requêtes cross-origin :

```python
from flask_cors import CORS
CORS(app)
```

**3. Dans le `Dockerfile` back-end** — installer `flask-cors` :

```dockerfile
RUN pip3 install flask-cors
```

### Pourquoi CORS est nécessaire ici

Le navigateur charge la page depuis `localhost:9000`. Le JavaScript de cette page fait une requête vers `localhost:5252`. Les navigateurs bloquent par défaut les requêtes vers un port différent (same-origin policy). Flask-CORS ajoute l'en-tête `Access-Control-Allow-Origin: *` à chaque réponse, ce qui indique au navigateur que c'est autorisé.

### Commandes pour tester

```bash
# Terminal 1 — back-end
docker build -f ./back-end/Dockerfile -t softy-pinko-back-end:task3 ./back-end
docker run -p 5252:5252 -it --rm --name softy-pinko-back-end-task3 softy-pinko-back-end:task3

# Terminal 2 — front-end
docker build -f ./front-end/Dockerfile -t softy-pinko-front-end:task3 ./front-end
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task3 softy-pinko-front-end:task3
# Dans le navigateur : http://localhost:9000
# "Hello, World!" doit apparaître en haut de la page
```

### Questions fréquentes

**Pourquoi le JavaScript utilise `localhost:5252` et pas le nom du conteneur ?**
Le JavaScript s'exécute dans le navigateur de l'utilisateur, pas dans un conteneur Docker. Du point de vue du navigateur, le back-end est accessible via `localhost:5252` car le port est mappé sur la machine hôte.

**Les deux conteneurs communiquent-ils directement ?**
Non. Le front-end sert juste des fichiers statiques. C'est le navigateur qui fait la requête vers le back-end. Les deux conteneurs n'ont pas besoin de se parler entre eux.

---

## Tâche 4 — Simplifier avec Docker Compose

**Objectif :** Remplacer les multiples commandes `docker build` et `docker run` par un seul fichier `docker-compose.yml`.

### Structure attendue

```
task4/
├── back-end/
│   ├── Dockerfile
│   └── api.py
├── front-end/
│   ├── Dockerfile
│   ├── softy-pinko-front-end.conf
│   └── softy-pinko-front-end/
└── docker-compose.yml
```

### Le fichier docker-compose.yml

```yaml
version: "3"

services:
  back-end:
    build: ./back-end
    ports:
      - "5252:5252"

  front-end:
    build: ./front-end
    ports:
      - "9000:9000"
```

### Commandes pour tester

```bash
docker-compose up --build
# Les deux services démarrent
# http://localhost:9000 pour le front-end
# http://localhost:5252/api/hello pour le back-end
```

### Questions fréquentes

**Que fait `build: ./back-end` ?**
Compose cherche un `Dockerfile` dans le répertoire `./back-end` et construit l'image automatiquement. Équivalent de `docker build -f ./back-end/Dockerfile ... ./back-end`.

**Faut-il toujours passer `--build` ?**
Non. `--build` force la reconstruction des images même si elles existent déjà en cache. Sans `--build`, Compose réutilise les images existantes. À utiliser quand on a modifié le code ou le Dockerfile.

**Comment arrêter proprement ?**
`Ctrl+C` stoppe les conteneurs. `docker-compose down` les supprime. `docker-compose down --volumes` supprime aussi les volumes associés.

---

## Tâche 5 — Proxy Server

**Objectif :** Ajouter un reverse proxy Nginx qui centralise tout le trafic. Le client ne communique plus directement avec le front-end ou le back-end.

### Architecture finale

```
Client (navigateur)
       │
       ▼
  proxy:80  (Nginx reverse proxy)
       │
       ├── /api/* ──────► back-end:5252  (Flask)
       │
       └── /* ──────────► front-end:9000  (Nginx)
```

### Structure attendue

```
task5/
├── back-end/
│   ├── Dockerfile
│   └── api.py
├── front-end/
│   ├── Dockerfile
│   ├── softy-pinko-front-end.conf
│   └── softy-pinko-front-end/
├── proxy/
│   ├── Dockerfile
│   └── proxy.conf
└── docker-compose.yml
```

### Le Dockerfile du proxy

```dockerfile
FROM nginx:latest
COPY ./proxy.conf /etc/nginx/conf.d/default.conf
```

Même principe que le front-end : on remplace la config Nginx par défaut par la nôtre.

### La configuration du proxy

```nginx
server {
    listen 80;

    location /api {
        proxy_pass http://back-end:5252;
    }

    location / {
        proxy_pass http://front-end:9000;
    }
}
```

**`proxy_pass http://back-end:5252`** — le nom `back-end` est résolu automatiquement par le réseau Docker Compose vers l'IP du conteneur back-end.

**Ordre des locations** — `/api` est plus spécifique que `/`. Nginx teste d'abord `/api`, donc les requêtes vers `/api/hello` vont bien au back-end.

### Modifications du docker-compose.yml

```yaml
version: "3"

services:
  back-end:
    build: ./back-end
    # plus de "ports" — le back-end n'est plus exposé directement

  front-end:
    build: ./front-end
    # plus de "ports" — le front-end non plus

  proxy:
    build: ./proxy
    ports:
      - "80:80"  # seul le proxy est exposé
```

### Modification du JavaScript front-end

Puisque tout passe par le proxy sur le port 80, l'URL du back-end change :

```javascript
// Avant (task3/task4) :
url: "http://localhost:5252/api/hello"

// Après (task5) :
url: "http://localhost/api/hello"
```

### Questions fréquentes

**Pourquoi retirer `ports` du back-end et du front-end ?**
Avec le proxy, les clients n'ont plus besoin d'accéder directement au back-end ou au front-end. Retirer les ports exposés renforce la sécurité : ces services ne sont accessibles que via le proxy, jamais directement depuis l'extérieur.

**Comment le proxy sait-il où est le back-end ?**
Docker Compose crée un réseau privé. Dans ce réseau, chaque service est accessible via son nom défini dans `docker-compose.yml`. `back-end` dans la config Nginx se résout automatiquement en l'IP du conteneur back-end.

**Pourquoi le port 80 ?**
Le port 80 est le port HTTP standard. Quand on tape `http://localhost` sans préciser de port, le navigateur utilise automatiquement le port 80.

---

## Tâche 6 — Scale Horizontally

**Objectif :** Lancer deux instances du back-end et configurer le proxy pour faire du load balancing Round Robin entre elles.

### Architecture finale

```
Client (navigateur)
       │
       ▼
  proxy:80  (Nginx)
       │
       ├── /* ──────────► front-end:9000
       │
       └── /api/* ──► back-end1:5252
                  └── back-end2:5252  (Round Robin)
```

### Modifications du docker-compose.yml

```yaml
version: "3"

services:
  back-end1:
    build: ./back-end

  back-end2:
    build: ./back-end

  front-end:
    build: ./front-end

  proxy:
    build: ./proxy
    ports:
      - "80:80"
```

On crée deux services `back-end1` et `back-end2` à partir du même Dockerfile. Ils sont identiques, mais tournent dans deux conteneurs séparés.

### Modifications de proxy.conf

```nginx
upstream back-end {
    server back-end1:5252;
    server back-end2:5252;
}

server {
    listen 80;

    location /api {
        proxy_pass http://back-end;
    }

    location / {
        proxy_pass http://front-end:9000;
    }
}
```

**`upstream back-end { ... }`** — définit un groupe de serveurs nommé `back-end`. Nginx applique Round Robin par défaut : requête 1 → back-end1, requête 2 → back-end2, requête 3 → back-end1...

**`proxy_pass http://back-end`** — pointe maintenant vers le groupe `upstream` plutôt qu'un serveur unique. Nginx choisit automatiquement quel serveur du groupe utiliser.

### Questions fréquentes

**Pourquoi deux services séparés et pas `replicas: 2` ?**
Dans cette version du projet, on nomme explicitement `back-end1` et `back-end2` pour que Nginx puisse référencer chaque instance par son nom dans le bloc `upstream`. Avec `replicas`, les noms sont générés automatiquement et moins prévisibles.

**Est-ce que les deux back-ends partagent des données ?**
Non. Chaque conteneur est totalement indépendant. Dans ce projet ce n'est pas un problème car l'API est stateless (elle ne stocke rien). Dans une vraie application, il faudrait une base de données partagée externe.

**Comment vérifier que le Round Robin fonctionne ?**
En ajoutant un identifiant dans la réponse Flask (par exemple le nom du host ou un numéro), on peut vérifier que les réponses alternent entre les deux serveurs en rafraîchissant la page plusieurs fois.

---

# PARTIE 3 — TABLEAU RÉCAPITULATIF

| Tâche | Répertoire | Ce qui est créé | Concepts clés |
|---|---|---|---|
| 0 | `task0/` | `Dockerfile` Ubuntu | `FROM`, `RUN`, `CMD`, `docker build`, `docker run` |
| 1 | `task1/` | `Dockerfile` + `api.py` Flask | `WORKDIR`, `COPY`, port mapping, Flask, `0.0.0.0` |
| 2 | `task2/` | Dockerfile Nginx + config | `nginx:latest`, `location`, `root`, `index`, `proxy_pass` |
| 3 | `task3/` | Front+back connectés | CORS, AJAX, communication inter-conteneurs via hôte |
| 4 | `task4/` | `docker-compose.yml` | `services`, `build`, `ports`, `docker-compose up` |
| 5 | `task5/` | Proxy Nginx centralisé | Reverse proxy, `upstream` simple, réseau Docker Compose |
| 6 | `task6/` | 2 instances back-end | Load balancing Round Robin, `upstream` multi-serveurs |

---

## Commandes utiles

```bash
# Construire une image
docker build -f ./Dockerfile -t nom:tag .

# Lancer un conteneur
docker run -p hôte:conteneur -it --rm --name nom image:tag

# Voir les conteneurs en cours
docker ps

# Voir toutes les images
docker images

# Supprimer un conteneur
docker rm nom

# Supprimer une image
docker rmi nom:tag

# Docker Compose — construire et démarrer
docker-compose up --build

# Docker Compose — démarrer en arrière-plan
docker-compose up -d --build

# Docker Compose — arrêter et supprimer
docker-compose down

# Voir les logs d'un service Compose
docker-compose logs back-end
```

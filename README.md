# 🎬 MEGARAM CASA — Projet-RabbitMQ

Architecture microservices pour la gestion de films, projections et réservations d'une société de cinéma, orchestrée avec **RabbitMQ** comme message broker et **Docker**.

---

## 📋 Table des matières

- [Architecture](#-architecture)
- [Technologies](#-technologies)
- [Structure du projet](#-structure-du-projet)
- [Prérequis](#-prérequis)
- [Démarrage rapide](#-démarrage-rapide)
- [Services & Ports](#-services--ports)
- [Variables d'environnement](#-variables-denvironnement)
- [API Reference](#-api-reference)
- [Tests](#-tests)
- [RabbitMQ Dashboard](#-rabbitmq-dashboard)
- [Commandes utiles](#-commandes-utiles)
- [Corrections apportées](#-corrections-apportées)
- [Dépannage](#-dépannage)

---

## 🏗️ Architecture

```
                        ┌─────────────────────┐
                        │     API Gateway      │
                        │   Node.js :3000      │
                        └──────────┬──────────┘
                                   │ HTTP Proxy
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────────┐   ┌─────────────────────┐
│ microservice-   │    │  microservice-       │   │ microservice-       │
│ films           │    │  projections         │   │ reservations        │
│ Node.js :3001   │    │  Node.js :3002       │   │ PHP 8.1 :3003       │
│ (Producer)      │    │  (Consumer)          │   │ (Slim Framework)    │
└────────┬────────┘    └──────────┬──────────┘   └─────────────────────┘
         │                        │
         │    AMQP (port 5672)    │
         ▼                        │
┌─────────────────────────────────┴──┐
│           RabbitMQ Broker           │
│     rabbitmq:3.13-management        │
│   AMQP :5672 | Dashboard :15672    │
└─────────────────────────────────────┘
```

### Flux de données RabbitMQ

```
POST /api/films
      │
      ▼
microservice-films (Producer)
      │
      │  publish → films_queue
      ▼
   RabbitMQ
      │
      │  consume ← films_queue
      ▼
microservice-projections (Consumer)
      │
      ▼
GET /api/projections  →  retourne les films reçus
```

---

## 🛠️ Technologies

| Composant | Technologie | Version |
|---|---|---|
| API Gateway | Node.js + Express | v20 LTS |
| Microservice Films | Node.js + Express + amqplib | v20 LTS |
| Microservice Projections | Node.js + Express + amqplib | v20 LTS |
| Microservice Réservations | PHP + Slim Framework | PHP 8.1 |
| Message Broker | RabbitMQ | 3.13 + Management |
| Conteneurisation | Docker + Docker Compose | - |

---

## 📁 Structure du projet

```
Projet-RabbitMQ/
├── .env                              ← Variables d'environnement globales
├── docker-compose.yml                ← Orchestration de tous les services
├── README.md                         ← Ce fichier
│
├── api-gateway/
│   ├── Dockerfile                    ← Image Node.js 20 Alpine
│   ├── app.js                        ← Entry point, proxy vers les microservices
│   ├── package.json
│   └── package-lock.json
│
├── microservice-films/
│   ├── Dockerfile                    ← Image Node.js 20 Alpine
│   ├── .env                          ← Variables locales (optionnel)
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│       ├── app.js                    ← Entry point (port 3001)
│       ├── controllers/
│       │   └── filmController.js     ← Logique métier films
│       ├── models/
│       ├── rabbitmq/
│       │   └── producer.js           ← Publie dans films_queue
│       └── routes/
│           └── filmRoutes.js         ← POST /api/films
│
├── microservice-projections/
│   ├── Dockerfile                    ← Image Node.js 20 Alpine
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│       ├── app.js                    ← Entry point (port 3002)
│       ├── controllers/
│       ├── rabbitmq/
│       │   └── consumer.js           ← Consomme films_queue
│       └── routes/
│           └── projectionRoutes.js   ← GET /api/projections
│
└── microservice-reservations/
    ├── Dockerfile                    ← Image PHP 8.1 CLI Alpine + Composer
    ├── composer.json                 ← Dépendances PHP (Slim Framework)
    ├── public/
    │   └── index.php                 ← Entry point PHP (port 3003)
    └── src/
        └── ReservationController.php
```

---

## ✅ Prérequis

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et démarré
- [Docker Compose](https://docs.docker.com/compose/) v2+
- Ports disponibles : **3000, 3001, 3002, 3003, 5672, 15672**

Vérifier que Docker fonctionne :
```powershell
docker --version
docker-compose --version
```

---

## 🚀 Démarrage rapide

### 1. Cloner / ouvrir le projet
```powershell
cd "c:\Users\KOUSRA YOUNES\Desktop\HH\Projet-RabbitMQ"
```

### 2. Lancer tous les services
```powershell
docker-compose up --build
```
> La première fois, Docker télécharge les images (~2-3 min). Les fois suivantes, le démarrage prend ~30 secondes.

### 3. Lancer en arrière-plan (mode détaché)
```powershell
docker-compose up --build -d
```

### 4. Vérifier que tout est démarré
```powershell
docker-compose ps
```

Résultat attendu :
```
NAME                   STATUS
megaram-rabbitmq       Up (healthy)
megaram-films          Up (healthy)
megaram-projections    Up (healthy)
megaram-reservations   Up (healthy)
megaram-gateway        Up (healthy)
```

### 5. Arrêter tous les services
```powershell
docker-compose down
```

---

## 🌐 Services & Ports

| Service | URL | Description |
|---|---|---|
| **API Gateway** | http://localhost:3000 | Point d'entrée unique |
| **Microservice Films** | http://localhost:3001 | Gestion & publication des films |
| **Microservice Projections** | http://localhost:3002 | Réception & affichage des films |
| **Microservice Réservations** | http://localhost:3003 | Gestion des réservations (PHP) |
| **RabbitMQ AMQP** | localhost:5672 | Broker de messages |
| **RabbitMQ Dashboard** | http://localhost:15672 | Interface d'administration |

---

## 🔐 Variables d'environnement

Le fichier `.env` à la racine contient les variables partagées :

```env
RABBITMQ_USER=admin
RABBITMQ_PASS=admin123
```

Ces variables sont injectées automatiquement dans `docker-compose.yml`.

### Variables par service

| Variable | Service(s) | Valeur Docker |
|---|---|---|
| `RABBITMQ_URL` | films, projections | `amqp://admin:admin123@rabbitmq:5672/` |
| `NODE_ENV` | films, projections, gateway | `production` |
| `PORT` | films | `3001` |
| `PORT` | projections | `3002` |
| `FILMS_SERVICE_URL` | gateway | `http://microservice-films:3001` |
| `PROJECTIONS_SERVICE_URL` | gateway | `http://microservice-projections:3002` |
| `RESERVATIONS_SERVICE_URL` | gateway | `http://microservice-reservations:3003` |

> ⚠️ **Important** : En production, changez `RABBITMQ_USER` et `RABBITMQ_PASS` et n'exposez pas le fichier `.env`.

---

## 📡 API Reference

### 🎬 Microservice Films — `http://localhost:3001`

#### `GET /health`
Vérifie que le service est opérationnel.

**Réponse :**
```json
{
  "status": "ok",
  "service": "microservice-films"
}
```

#### `POST /api/films`
Publie un film dans la queue RabbitMQ `films_queue`.

**Body (JSON) :**
```json
{
  "titre": "Inception",
  "genre": "Science-Fiction",
  "duree": 148
}
```

**Réponse :**
```json
{
  "message": "Film envoyé à RabbitMQ",
  "film": {
    "titre": "Inception",
    "genre": "Science-Fiction",
    "duree": 148
  }
}
```

---

### 📽️ Microservice Projections — `http://localhost:3002`

#### `GET /health`
Vérifie que le service est opérationnel.

**Réponse :**
```json
{
  "status": "ok",
  "service": "microservice-projections"
}
```

#### `GET /api/projections`
Retourne tous les films reçus depuis la queue RabbitMQ.

**Réponse :**
```json
[
  {
    "titre": "Inception",
    "genre": "Science-Fiction",
    "duree": 148
  }
]
```

---

### 🎟️ Microservice Réservations — `http://localhost:3003`

Service PHP avec Slim Framework. Accessible via `GET http://localhost:3003/`.

---

### 🌐 API Gateway — `http://localhost:3000`

Proxy unique vers tous les microservices. Redirige les requêtes vers les services appropriés.

---

## 🧪 Tests

### Test rapide via PowerShell

#### 1. Vérifier les health checks
```powershell
Invoke-WebRequest -Uri "http://localhost:3001/health" -UseBasicParsing | Select-Object StatusCode, Content
Invoke-WebRequest -Uri "http://localhost:3002/health" -UseBasicParsing | Select-Object StatusCode, Content
```

#### 2. Envoyer un film (Producer → RabbitMQ)
```powershell
Invoke-WebRequest -Uri "http://localhost:3001/api/films" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"titre":"Inception","genre":"SF","duree":148}' `
  -UseBasicParsing | Select-Object StatusCode, Content
```

#### 3. Vérifier que le Consumer a reçu le film
```powershell
Invoke-WebRequest -Uri "http://localhost:3002/api/projections" `
  -UseBasicParsing | Select-Object Content
```

#### 4. Tester le service PHP
```powershell
Invoke-WebRequest -Uri "http://localhost:3003/" -UseBasicParsing | Select-Object StatusCode
```

#### 5. Envoyer plusieurs films pour tester la queue
```powershell
$films = @(
  '{"titre":"Avengers","genre":"Action","duree":143}',
  '{"titre":"Interstellar","genre":"SF","duree":169}',
  '{"titre":"The Dark Knight","genre":"Action","duree":152}'
)

foreach ($film in $films) {
  Invoke-WebRequest -Uri "http://localhost:3001/api/films" `
    -Method POST `
    -ContentType "application/json" `
    -Body $film `
    -UseBasicParsing | Select-Object StatusCode, Content
}
```

---

## 🐇 RabbitMQ Dashboard

Accédez à l'interface d'administration :

```
URL      : http://localhost:15672
Login    : admin
Password : admin123
```

Dans le dashboard vous pouvez :
- 📊 Voir les **queues** actives (`films_queue`)
- 📈 Monitorer le **débit de messages** en temps réel
- 🔌 Voir les **connexions** des microservices
- 📤 **Publier manuellement** des messages de test
- 🗑️ Vider une queue si nécessaire

---

## 🔧 Commandes utiles

### Docker Compose

```powershell
# Démarrer (avec rebuild)
docker-compose up --build

# Démarrer en arrière-plan
docker-compose up --build -d

# Voir l'état des conteneurs
docker-compose ps

# Voir les logs de tous les services
docker-compose logs -f

# Voir les logs d'un service spécifique
docker-compose logs -f microservice-films
docker-compose logs -f rabbitmq

# Arrêter sans supprimer les volumes
docker-compose stop

# Arrêter et supprimer les conteneurs
docker-compose down

# Arrêter et supprimer TOUT (conteneurs + volumes + images)
docker-compose down -v --rmi all
```

### Docker général

```powershell
# Lister les conteneurs en cours
docker ps

# Entrer dans un conteneur (shell interactif)
docker exec -it megaram-films sh
docker exec -it megaram-rabbitmq sh

# Voir les logs d'un conteneur
docker logs megaram-films
docker logs megaram-projections

# Inspecter un conteneur
docker inspect megaram-rabbitmq

# Nettoyer les ressources inutilisées
docker system prune -f
```

---

## 🩹 Corrections apportées

Les corrections suivantes ont été effectuées pour faire fonctionner le projet en environnement Docker :

### 1. URL RabbitMQ codée en dur → Variable d'environnement

**Fichier :** `microservice-films/src/rabbitmq/producer.js`
```js
// ❌ Avant (ne fonctionne pas dans Docker)
const connection = await amqp.connect('amqp://localhost');

// ✅ Après
const rabbitmqUrl = process.env.RABBITMQ_URL || 'amqp://localhost';
const connection = await amqp.connect(rabbitmqUrl);
```

**Fichier :** `microservice-projections/src/rabbitmq/consumer.js`
```js
// ❌ Avant
const connection = await amqp.connect('amqp://localhost');

// ✅ Après
const rabbitmqUrl = process.env.RABBITMQ_URL || 'amqp://localhost';
const connection = await amqp.connect(rabbitmqUrl);
```

> Dans Docker, les conteneurs communiquent par **nom de service** (`rabbitmq`), pas par `localhost`.

### 2. Route `/health` manquante

Ajout d'un endpoint `GET /health` dans `microservice-films/src/app.js` et `microservice-projections/src/app.js` pour que les healthchecks Docker fonctionnent.

### 3. Gestion d'erreur améliorée

`initRabbitMQ()` et `startConsumer()` sont maintenant appelés avec `.catch(console.error)` pour éviter les crashs silencieux.

---

## 🚨 Dépannage

### ❌ `no configuration file provided: not found`
**Cause :** La commande `docker-compose` est lancée depuis le mauvais répertoire.

**Solution :**
```powershell
cd "c:\Users\KOUSRA YOUNES\Desktop\HH\Projet-RabbitMQ"
docker-compose up --build
```

---

### ❌ `ECONNREFUSED 127.0.0.1:5672`
**Cause :** Un service Node.js tente de se connecter à RabbitMQ via `localhost` au lieu du nom du service Docker.

**Solution :** Vérifier que `RABBITMQ_URL` est bien défini dans le code :
```js
const rabbitmqUrl = process.env.RABBITMQ_URL || 'amqp://localhost';
```

---

### ❌ Service `unhealthy` après démarrage
**Cause :** Le healthcheck échoue car la route `/health` n'existe pas, ou RabbitMQ n'est pas encore prêt.

**Solution :** Attendre ~30-60 secondes. Si le problème persiste :
```powershell
docker-compose logs -f microservice-films
```

---

### ❌ Port déjà utilisé (`address already in use`)
**Cause :** Un autre processus utilise un des ports (3000, 3001, 3002, 3003, 5672, 15672).

**Solution :**
```powershell
# Trouver le processus qui utilise le port (exemple port 3001)
netstat -ano | findstr :3001

# Tuer le processus (remplacer PID par le numéro trouvé)
taskkill /PID <PID> /F
```

---

### ❌ `curl` ne fonctionne pas dans PowerShell
**Cause :** PowerShell utilise `curl` comme alias pour `Invoke-WebRequest`.

**Solution :** Utiliser `Invoke-WebRequest` :
```powershell
Invoke-WebRequest -Uri "http://localhost:3001/health" -UseBasicParsing | Select-Object StatusCode, Content
```

---

## 👥 Auteur

**MEGARAM CASA** — Projet académique microservices avec RabbitMQ

---

## 📄 Licence

ISC

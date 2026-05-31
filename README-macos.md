# Déployer un clone de ChatGPT souverain dans le cadre de l'expérimentation Mistral AMUE
## Version maquette locale — macOS Apple Silicon (M2)

> ✅🍎 **VERSION VALIDÉE SUR MACOS**
> Ce document est une adaptation de la version Linux pour une installation de maquette locale sur MacBook Air M2. Il ne couvre pas un déploiement en production.

---

> **Contexte** : Ce tutoriel permet d'installer la stack MyIA souveraine (Mistral AMUE + LiteLLM + OpenWebUI) sur un MacBook Air M2 pour expérimentation locale. L'objectif est d'avoir une instance fonctionnelle sur votre poste en moins d'une heure, sans serveur Linux.

---

| Date | Auteur | Modification | Validation |
|---|---|---|---|
| 29 mai 2026 | | Version initiale macOS M2 | |

---

## Sommaire

1. [Les ingrédients](#1-les-ingrédients)
2. [Architecture de la solution](#2-architecture-de-la-solution)
3. [Mise en place pas à pas](#3-mise-en-place-pas-à-pas)
4. [Vérifier que tout fonctionne](#4-vérifier-que-tout-fonctionne)
5. [Et ensuite ?](#5-et-ensuite-)

---

## 1. Les ingrédients

### Matériel

| Composant | Votre config | Suffisant ? |
|---|---|---|
| Puce | Apple M2 (ARM64) | ✅ |
| RAM | 16 Go | ✅ (8 Go alloués à Docker, largement suffisant) |
| Disque | 20 Go libres recommandés | ✅ |
| OS | macOS Sequoia 15.x | ✅ |

> Pas de GPU requis : les inférences sont déléguées à l'API Mistral (cloud souverain français).

### Logiciels à installer

- **Docker Desktop for Mac** (ARM64) — obligatoire, remplace Docker Engine sur Mac
- **Homebrew** — gestionnaire de paquets macOS
- **Python 3.11+** — optionnel, pour les tests

### Accès & Clés

- **Clé API Mistral AMUE** : fournie par votre référent numérique via le dispositif AMUE.
  Forme attendue : `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  Endpoint de base : `https://api.mistral.ai/v1`

### Compétences requises

- Bases du Terminal macOS
- Notions de Docker (lancer des conteneurs)
- Notions de LLM et d'API REST (cf. Ateliers 1 & 2 Esup)

---

### RGPD

> Avant de démarrer : les prompts que vous envoyez transitent vers les serveurs de Mistral AI pour traitement. Mistral est un opérateur français, les données sont hébergées en Europe, et l'accord AMUE prévoit la non-utilisation de vos données pour le réentraînement des modèles — mais vérifiez ce point avec votre DPO avant tout usage impliquant des données sensibles. Ne faites jamais transiter de données personnelles d'étudiants ou d'agents via cet outil sans analyse préalable.

---

## 2. Architecture de la solution

Identique à la version Linux — tout tourne dans des conteneurs Docker Desktop :

```
  ┌──────────────────────────────────────────┐
  │        MacBook Air M2                     │
  │                                          │
  │  http://localhost:3000                   │
  │  ┌───────────────────────────────────┐   │
  │  │   OpenWebUI (interface ChatGPT)   │   │
  │  └─────────────────┬─────────────────┘   │
  │                    │ :4000               │
  │  ┌─────────────────▼─────────────────┐   │
  │  │   LiteLLM Proxy (gateway)         │   │
  │  └─────────────────┬─────────────────┘   │
  │                    │                     │
  │  ┌─────────────────▼─────────────────┐   │
  │  │   PostgreSQL + Redis              │   │
  │  └───────────────────────────────────┘   │
  └──────────────────────┬───────────────────┘
                         │ HTTPS
                         ▼
          API Mistral AMUE (mistral-large, etc.)
```

---

## 3. Mise en place pas à pas

> Tous les fichiers sont créés dans `~/myia`. Ouvrir le Terminal et ne plus le quitter.

---

### Étape 1 — Installer les prérequis

**Docker Desktop for Mac**

Télécharger et installer la version Apple Silicon (ARM64) depuis le site officiel :
https://docs.docker.com/desktop/install/mac-install/

Une fois installé, lancer Docker Desktop depuis le Launchpad. Attendre que la baleine dans la barre de menu soit stable (pas animée).

> **Réglage mémoire important** : par défaut Docker Desktop alloue la moitié de votre RAM. Avec 16 Go, il alloue 8 Go — c'est suffisant. Pour vérifier ou ajuster : Docker Desktop → Settings → Resources → Memory.

Vérifier l'installation dans le Terminal :

```bash
docker --version          # Docker version 24+
docker compose version    # Docker Compose version v2+
```

**Homebrew et Python** (si pas déjà installés)

```bash
# Installer Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installer Python
brew install python@3.11

# Vérification
python3 --version         # Python 3.11+
```

**Créer l'arborescence du projet**

```bash
mkdir -p ~/myia/{config,data/{litellm,openwebui,postgres,redis}}
cd ~/myia
```

---

### Étape 2 — Fichier de configuration LiteLLM

Créer `config/litellm_config.yaml` :

```yaml
general_settings:
  store_model_in_db: true

litellm_settings:
  num_retries: 3
  request_timeout: 3600
  allowed_fails: 5
  cooldown_time: 30
  drop_params: true
  cache: true
  cache_params:
    type: redis
    host: redis
    port: 6379
    password: os.environ/REDIS_PASSWORD

model_list:

  - model_name: mistral-large
    litellm_params:
      model: mistral/mistral-large-latest
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      max_tokens: 131072
      max_input_tokens: 128000
      max_output_tokens: 4096
      description: "Mistral Large — modèle puissant (AMUE)"

  - model_name: mistral-small
    litellm_params:
      model: mistral/mistral-small-latest
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      max_tokens: 32768
      max_input_tokens: 32000
      max_output_tokens: 4096
      description: "Mistral Small — rapide et économique (AMUE)"

  - model_name: codestral
    litellm_params:
      model: mistral/codestral-latest
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      max_tokens: 32768
      description: "Codestral — assistance à la programmation (AMUE)"

  - model_name: mistral-embed
    litellm_params:
      model: mistral/mistral-embed
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      mode: embedding
      description: "Embeddings Mistral pour le RAG"

  # Modèle local via Ollama — utilisable sans clé AMUE
  - model_name: qwen3
    litellm_params:
      model: ollama/qwen3:0.6b
      api_base: http://host.docker.internal:11434
    model_info:
      description: "Qwen3 0.6b — modèle local via Ollama"
```

> **Note Ollama** : `host.docker.internal` est l'adresse spéciale Docker Desktop sur Mac pour joindre un service qui tourne sur l'hôte (ton Mac) depuis un conteneur. Ollama doit être lancé avec `ollama serve` avant de démarrer la stack.

> **Note AMUE** : Si l'AMUE vous a communiqué un endpoint dédié, remplacez tous les `api_base` en conséquence.

---

### Étape 3 — Fichier d'environnement

Créer le fichier `.env` dans `~/myia` :

```bash
cat > .env << 'EOF'
# === MISTRAL AMUE ===
MISTRAL_API_KEY=sk-votre-cle-mistral-amue-ici

# === LITELLM ===
LITELLM_MASTER_KEY=sk-master-CHANGEZ-MOI
LITELLM_SECRET_KEY=jwt-secret-CHANGEZ-MOI-aussi

# === POSTGRESQL ===
POSTGRES_USER=litellm
POSTGRES_PASSWORD=pgpassword-CHANGEZ
POSTGRES_DB=litellm

# === REDIS ===
REDIS_PASSWORD=redis-password-CHANGEZ

# === OPENWEBUI ===
WEBUI_SECRET_KEY=openwebui-secret-CHANGEZ
WEBUI_NAME=MyIA — Maquette locale
WEBUI_URL=http://localhost:3000

# === PORTS (modifier si conflit) ===
PORT_OWUI=3000
PORT_LITELLM=4000

# === VOLUMES ===
VOL_DATA_POSTGRES=./data/postgres
VOL_DATA_REDIS=./data/redis
VOL_DATA_OPENWEBUI=./data/openwebui
EOF

chmod 600 .env
```

---

### Étape 3b — Personnaliser les secrets avant le premier démarrage

> ⚠️ Cette étape est obligatoire et doit être faite **avant** `docker compose up -d`. Une fois PostgreSQL démarré une première fois, les mots de passe sont gravés en base et ne peuvent plus être changés sans tout réinitialiser.

Ouvrir le `.env` dans un éditeur :

```bash
open -e ~/myia/.env
```

Remplacer **toutes** les valeurs contenant `CHANGEZ` :

| Variable | Rôle | Valeur |
|---|---|---|
| `MISTRAL_API_KEY` | Clé API fournie par l'AMUE | Coller votre vraie clé `sk-xxx...` |
| `LITELLM_MASTER_KEY` | Clé admin LiteLLM | `sk-` + 32 caractères aléatoires |
| `LITELLM_SECRET_KEY` | Secret JWT interne | Chaîne aléatoire longue |
| `POSTGRES_PASSWORD` | Mot de passe base de données | Chaîne aléatoire |
| `REDIS_PASSWORD` | Mot de passe cache Redis | Chaîne aléatoire |
| `WEBUI_SECRET_KEY` | Secret session OpenWebUI | Chaîne aléatoire |

Pour générer des valeurs robustes directement dans le Terminal :

```bash
# Générer 6 secrets d'un coup — copier-coller les valeurs dans le .env
for i in 1 2 3 4 5 6; do python3 -c "import secrets; print(secrets.token_urlsafe(32))"; done
```

Sauvegarder le fichier avant de passer à l'étape suivante.

---

### Étape 4 — Docker Compose

Créer `docker-compose.yml` dans `~/myia` :

```yaml


services:

  postgres:
    image: postgres:16-alpine
    platform: linux/arm64             # optimisé pour Apple Silicon
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ${VOL_DATA_POSTGRES:-./data/postgres}:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
    networks:
      - nw_llm

  redis:
    image: redis:7-alpine
    platform: linux/arm64             # optimisé pour Apple Silicon
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1
    volumes:
      - ${VOL_DATA_REDIS:-./data/redis}:/data
    networks:
      - nw_llm
    healthcheck:
      test: ["CMD", "redis-cli", "--pass", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    restart: unless-stopped
    ports:
      - "${PORT_LITELLM:-4000}:4000"
    environment:
      MISTRAL_API_KEY: ${MISTRAL_API_KEY}
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      LITELLM_SECRET: ${LITELLM_SECRET_KEY}
      DATABASE_URL: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}"
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      STORE_MODEL_IN_DB: "True"
    volumes:
      - ./config/litellm_config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - nw_llm

  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "${PORT_OWUI:-3000}:8080"
    environment:
      OPENAI_API_BASE_URL: http://litellm:4000/v1
      OPENAI_API_KEY: ${LITELLM_MASTER_KEY}
      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}
      WEBUI_NAME: ${WEBUI_NAME}
      WEBUI_URL: ${WEBUI_URL}
      DEFAULT_MODELS: "mistral-large"
      # ⚠️ Mettre "true" pour créer le premier compte admin, puis repasser à "false"
      ENABLE_SIGNUP: "false"
      RAG_EMBEDDING_ENGINE: "openai"
      RAG_EMBEDDING_MODEL: "mistral-embed"
      RAG_OPENAI_API_BASE_URL: http://litellm:4000/v1
      RAG_OPENAI_API_KEY: ${LITELLM_MASTER_KEY}
      SCARF_NO_ANALYTICS: "true"
      DO_NOT_TRACK: "true"
      ANONYMIZED_TELEMETRY: "false"
    volumes:
      - ${VOL_DATA_OPENWEBUI:-./data/openwebui}:/app/backend/data
    depends_on:
      litellm:
        condition: service_started
    networks:
      - nw_llm

networks:
  nw_llm:
    driver: bridge
```

> **Note Apple Silicon** : le tag `platform: linux/arm64` est ajouté sur postgres et redis pour forcer les images natives ARM64 et éviter l'émulation x86 (plus lente). LiteLLM et OpenWebUI publient des images multi-arch qui se sélectionnent automatiquement.

> **Note timeouts** : les `start_period` sont volontairement plus longs que dans la version Linux (60s pour PostgreSQL, 180s pour LiteLLM). Docker Desktop sur Mac ajoute une couche de virtualisation qui ralentit les démarrages — les migrations Prisma de LiteLLM prennent jusqu'à 3 minutes sur M2.

---

### Étape 5 — Premier démarrage

```bash
cd ~/myia

# Lancer tous les services
docker compose up -d

# Le premier démarrage télécharge les images (~1-2 Go) — patienter
# Surveiller le démarrage
docker compose ps

# Logs si besoin
docker compose logs -f litellm
docker compose logs -f openwebui
```

Résultat attendu après ~60-90 secondes :

```
NAME                    STATUS                    PORTS
myia-postgres    Up X minutes (healthy)    5432/tcp
myia-redis       Up X minutes (healthy)    6379/tcp
myia-litellm     Up X minutes (healthy)    0.0.0.0:4000->4000/tcp
myia-openwebui   Up X minutes (healthy)    0.0.0.0:3000->8080/tcp
```

---

### Étape 6 — Configuration initiale OpenWebUI

1. Ouvrir http://localhost:3000 dans Safari ou Chrome
2. Créer le **compte administrateur** (premier compte = admin automatiquement)
3. Aller dans **Paramètres → Connexions** et vérifier que LiteLLM est détecté
4. Aller dans **Admin → Utilisateurs** pour créer d'autres comptes si besoin

---

### Étape 6b — Tester avec Ollama en attendant la clé AMUE

Si tu n'as pas encore ta clé Mistral AMUE, tu peux utiliser un modèle local via Ollama.

**Installer Ollama**
```bash
brew install ollama
```

**Démarrer Ollama** — laisser ce terminal ouvert :
```bash
ollama serve
```

**Télécharger un modèle léger** dans un second terminal :
```bash
ollama pull qwen3:0.6b
```

**Activer le modèle dans LiteLLM** — décommenter la section Ollama dans `config/litellm_config.yaml` :
```yaml
  - model_name: qwen3
    litellm_params:
      model: ollama/qwen3:0.6b
      api_base: http://host.docker.internal:11434
    model_info:
      description: "Qwen3 0.6b — modèle local via Ollama"
```

**Redémarrer LiteLLM** :
```bash
docker compose -f docker-compose.macos.yml restart litellm
```

Le modèle `qwen3` apparaît dans le menu déroulant d'OpenWebUI.

> **Note mémoire** : `qwen3:0.6b` (500 Mo) et `nomic-embed-text` (292 Mo) coexistent sans problème sur M2 16 Go. Éviter `qwen3:1.7b` — trop gourmand en parallèle.

---

## 4. Vérifier que tout fonctionne

Identique à la version Linux — même commandes, même script.

---

### 4.1 — Les conteneurs sont-ils tous en vie ?

```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Tous les services doivent afficher `healthy`. Si un service est en `starting` ou `unhealthy` :

```bash
docker compose logs --tail=50 litellm
docker compose logs --tail=50 openwebui
```

---

### 4.2 — LiteLLM reçoit-il bien la clé Mistral ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI" \
  http://localhost:4000/health | python3 -m json.tool
```

---

### 4.3 — Les modèles sont-ils bien exposés ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI" \
  http://localhost:4000/models | python3 -m json.tool
```

Les quatre modèles doivent apparaître : `mistral-large`, `mistral-small`, `codestral`, `mistral-embed`.

---

### 4.4 — Le chat fonctionne-t-il de bout en bout ?

```python
# recette.py
import openai, sys

BASE_URL = "http://localhost:4000/v1"
API_KEY  = "sk-master-CHANGEZ-MOI"

client = openai.OpenAI(api_key=API_KEY, base_url=BASE_URL)
ok = True

def check(label, fn):
    global ok
    try:
        fn()
        print(f"✅  {label}")
    except Exception as e:
        print(f"❌  {label} → {e}")
        ok = False

def test_chat():
    rep = client.chat.completions.create(
        model="mistral-large",
        messages=[
            {"role": "system", "content": "Tu es un assistant de l'ESR français."},
            {"role": "user",   "content": "Dis bonjour en une phrase."}
        ],
        max_tokens=50
    )
    assert rep.choices[0].message.content.strip(), "Réponse vide"

def test_streaming():
    tokens = []
    stream = client.chat.completions.create(
        model="mistral-small",
        messages=[{"role": "user", "content": "Compte de 1 à 3."}],
        stream=True
    )
    for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            tokens.append(delta)
    assert tokens, "Aucun token reçu en streaming"

def test_codestral():
    rep = client.chat.completions.create(
        model="codestral",
        messages=[{"role": "user", "content": "Écris une fonction Python qui retourne la somme de deux entiers."}],
        max_tokens=100
    )
    assert "def " in rep.choices[0].message.content, "Pas de code Python dans la réponse"

def test_embeddings():
    emb = client.embeddings.create(
        model="mistral-embed",
        input=["test embedding"]
    )
    assert len(emb.data[0].embedding) > 0, "Vecteur vide"

print("=== Recette MyIA AMUE — macOS M2 ===\n")
check("Chat simple (mistral-large)",     test_chat)
check("Streaming (mistral-small)",       test_streaming)
check("Génération de code (codestral)", test_codestral)
check("Embeddings RAG (mistral-embed)", test_embeddings)

print(f"\n{'✅ Recette réussie.' if ok else '❌ Des tests ont échoué.'}")
sys.exit(0 if ok else 1)
```

```bash
pip3 install openai
python3 recette.py
```

---

### 4.5 — L'interface OpenWebUI est-elle fonctionnelle ?

Ouvrir http://localhost:3000 et dérouler les vérifications suivantes :

**Connexion et accès**
- [ ] La page de connexion s'affiche sans erreur
- [ ] La création du compte administrateur fonctionne
- [ ] Les modèles `mistral-large`, `mistral-small` et `codestral` apparaissent dans le menu déroulant

**Conversation**
- [ ] Envoyer *"Bonjour, qui es-tu ?"* → réponse en streaming, en français
- [ ] Changer de modèle et constater que la réponse change de style

**RAG — prérequis**

Sans clé AMUE, le modèle d'embeddings doit être Ollama. Avant de tester le RAG :

```bash
ollama pull nomic-embed-text
```

Puis dans OpenWebUI : **Admin → Settings → Documents**
- Embedding Model Engine → `Ollama`
- Embedding Model → `nomic-embed-text`
- Ollama URL → `http://host.docker.internal:11434` (ne pas mettre `localhost` ou `127.0.0.1`)
- Sauvegarder

> **Note mémoire M2** : `qwen3:0.6b` (500 Mo) et `nomic-embed-text` (292 Mo) coexistent sans problème. Éviter `qwen3:1.7b` (1.9 Go) en parallèle — la mémoire devient trop contrainte.

**RAG — test**
- [ ] Créer un petit PDF de test si nécessaire (voir commande ci-dessous)
- [ ] Uploader le PDF via l'icône trombone dans le chat
- [ ] Poser une question sur le contenu de ce document
- [ ] Vérifier que la réponse cite le document sans halluciner

Pour créer un PDF de test minimal :
```bash
cat > /tmp/test.txt << 'TXTEOF'
L'université de Strasbourg a été fondée en 1621.
Elle compte environ 52 000 étudiants.
TXTEOF
cupsfilter /tmp/test.txt > /tmp/test.pdf 2>/dev/null
```

> **Note** : commencer par un petit PDF (1-2 pages) pour valider le pipeline. Les PDFs scannés (images) ne sont pas supportés sans OCR — utiliser des PDFs natifs avec texte sélectionnable.

---

### 4.6 — Le fallback fonctionne-t-il en cas de panne ?

Commenter temporairement `mistral-large` dans `config/litellm_config.yaml`, redémarrer LiteLLM et vérifier que la réponse arrive via le fallback :

```bash
docker compose restart litellm

curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI" \
  -H "Content-Type: application/json" \
  -d '{"model": "mistral-large", "messages": [{"role": "user", "content": "test fallback"}]}'

docker compose logs litellm | grep -i fallback
```

Remettre la config d'origine avant de continuer.

---

### 4.7 — Que faire si quelque chose ne va pas ?

| Symptôme | Cause probable | Action |
|---|---|---|
| Conteneur `litellm` en `unhealthy` | PostgreSQL pas encore prêt | Attendre 30s, relancer `docker compose up -d` |
| Erreur 401 sur `/health` | Mauvaise `LITELLM_MASTER_KEY` | Vérifier le `.env` et redémarrer |
| Erreur 429 ou `quota exceeded` | Clé AMUE épuisée | Contacter votre référent AMUE |
| Modèles absents dans OpenWebUI | OpenWebUI ne joint pas LiteLLM | Vérifier les logs OpenWebUI |
| RAG sans résultat | Modèle d'embedding non configuré | Vérifier les variables `RAG_EMBEDDING_*` |
| Images lentes au démarrage | Émulation ARM/x86 | Vérifier le tag `platform: linux/arm64` dans le Compose |
| Impossible de créer le compte admin | `ENABLE_SIGNUP: "false"` | Passer à `"true"` le temps de créer le compte, puis remettre à `"false"` |
| Modèle Ollama inaccessible | Ollama non démarré ou mauvais endpoint | Vérifier que `ollama serve` tourne, et que `api_base` est bien `http://host.docker.internal:11434` |
| Ollama embed error EOF | URL incorrecte dans OpenWebUI | Admin → Settings → Documents → URL Ollama → `http://host.docker.internal:11434` (pas `127.0.0.1`) |
| Ollama embed error 500 | Mémoire insuffisante ou PDF trop gros | Stopper les autres modèles Ollama (`ollama stop`), tester avec un petit PDF d'abord |
| Docker Desktop ne démarre pas | Ressources insuffisantes | Vérifier dans Settings → Resources que la mémoire est ≥ 6 Go |

---

### 4.8 — Arrêter et reprendre la maquette

```bash
# Arrêter sans perdre les données
docker compose stop

# Reprendre
docker compose start

# Tout supprimer (données comprises) — pour repartir de zéro
docker compose down -v
```

---

## 5. Et ensuite ?

Votre maquette tourne sur http://localhost:3000. Vous pouvez expérimenter, tester le RAG, former vos collègues localement.

Pour passer à une instance partagée dans votre établissement, consultez le tutoriel de déploiement en production (version Linux/serveur) qui couvre :

- Exposition publique sécurisée avec Nginx et certificat TLS
- Authentification via le SSO de votre établissement (Shibboleth / Keycloak)
- Gestion des quotas et budgets par équipe ou département
- Monitoring, alerting et sauvegardes automatisées

---

*Adaptation macOS M2 du tutoriel rédigé dans le cadre des Ateliers IA Esup 2025/2026 — basé sur les briques open source LiteLLM, OpenWebUI et l'offre Mistral AMUE.*

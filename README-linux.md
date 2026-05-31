# Déployer un clone de ChatGPT souverain dans le cadre de l'expérimentation Mistral AMUE

> ⚠️ **VERSION DE TRAVAIL — contenu non encore validé**
> Le présent document est un document de travail non encore validé par nos équipes. Merci d'attendre sa finalisation avant de l'appliquer chez vous.
> *La version initiale de cette proposition a été générée sur Claude.ai puis travaillée par des humains.*

| Date | Auteur | Modification | Validation |
|---|---|---|---|
| 16 avr. 2026 | Nicolas Truchaud | Version initiale | |
| 20 mai 2026 | David Boit | Debian dans tableau OS, lien doc officielle Docker, ports et volumes externalisés dans le .env avec valeur par défaut (`:-`), réseau Docker renommé `nw_llm`. | |

---

> **Contexte** : Ce tutoriel s'appuie sur les Ateliers IA Esup 2025/2026 (Université de Rennes / Université de Strasbourg) pour assembler une stack complète type ChatGPT souverain. Le choix architectural retenu ici est d'utiliser l'API Mistral via la clé négociée par l'AMUE pour l'enseignement supérieur français — ce qui permet de démarrer sans infrastructure GPU et avec un haut niveau de souveraineté (données hébergées en Europe, opérateur français). Si votre établissement dispose de GPU et souhaite une indépendance totale vis-à-vis de toute API externe, la brique Mistral API peut être remplacée par un moteur vLLM local — la configuration est décrite dans l'Atelier 2 Esup.

---

## Sommaire

1. [Les ingrédients](#1-les-ingrédients)
2. [Architecture de la solution](#2-architecture-de-la-solution)
3. [Mise en place pas à pas](#3-mise-en-place-pas-à-pas)
4. [Vérifier que tout fonctionne](#4-vérifier-que-tout-fonctionne)
5. [Et ensuite ?](#5-et-ensuite-)

---

## 1. Les ingrédients

### Matériel & Infrastructure

| Composant | Minimum | Recommandé |
|---|---|---|
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 Go | 16 Go |
| Disque | 20 Go SSD | 50 Go SSD |
| OS | Ubuntu 22.04 LTS ou Debian 12 | Ubuntu 24.04 LTS ou Debian 12 |
| Réseau | Accès HTTPS sortant vers api.mistral.ai | + nom de domaine propre |

> Pas de GPU requis : les inférences sont déléguées à l'API Mistral (cloud souverain français).

### Logiciels requis

- **Docker Engine** 24+ et **Docker Compose** v2
- **Git**
- **Python 3.11+** (optionnel, pour les tests)

### Accès & Clés

- **Clé API Mistral AMUE** : fournie par votre référent numérique via le dispositif AMUE.
  Forme attendue : `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  Endpoint de base : `https://api.mistral.ai/v1` (ou endpoint AMUE dédié si communiqué)
- **Accès SSH** au serveur cible
- **Droits sudo** sur le serveur

### Compétences requises

- Administration Linux (niveau intermédiaire)
- Bases de Docker / Docker Compose
- Notions de LLM et d'API REST (cf. Ateliers 1 & 2 Esup)

---

### RGPD

> Avant de démarrer : les prompts que vous envoyez transitent vers les serveurs de Mistral AI pour traitement. Mistral est un opérateur français, les données sont hébergées en Europe, et l'accord AMUE prévoit la non-utilisation de vos données pour le réentraînement des modèles — mais vérifiez ce point avec votre DPO avant tout usage impliquant des données sensibles. Ne faites jamais transiter de données personnelles d'étudiants ou d'agents via cet outil sans analyse préalable.

---

## 2. Architecture de la solution

```
                            ┌─────────────────────────────────┐
                            │        Établissement             │
  ┌─────────┐   HTTPS       │  ┌──────────────────────────┐   │
  │  Users  │──────────────▶│  │   Nginx / Traefik         │   │
  └─────────┘               │  │   (reverse proxy + TLS)   │   │
                            │  └────────────┬─────────────┘   │
                            │               │ :3000            │
                            │  ┌────────────▼─────────────┐   │
                            │  │   OpenWebUI              │   │
                            │  │   (interface ChatGPT-like)│   │
                            │  └────────────┬─────────────┘   │
                            │               │ :4000 (OpenAI API)│
                            │  ┌────────────▼─────────────┐   │
                            │  │   LiteLLM Proxy          │   │
                            │  │   (gateway + quotas)     │   │
                            │  └────────────┬─────────────┘   │
                            │               │                  │
                            │  ┌────────────▼─────────────┐   │
                            │  │  PostgreSQL + Redis       │   │
                            │  │  (logs, budgets, users)   │   │
                            │  └──────────────────────────┘   │
                            └──────────────┬──────────────────┘
                                           │ HTTPS
                                           ▼
                            ┌──────────────────────────────┐
                            │  API Mistral (AMUE)           │
                            │  mistral-large / small /      │
                            │  codestral / embed...         │
                            └──────────────────────────────┘
```

**Pourquoi cette pile ?**

- **Mistral AI** : modèles français, données hébergées en Europe, conformité RGPD facilitée
- **LiteLLM** : passerelle open source qui uniformise l'accès, gère les quotas par utilisateur/équipe, et permet d'ajouter d'autres modèles sans changer l'interface
- **OpenWebUI** : interface open source type ChatGPT, supporte RAG, SSO, outils, gestion des espaces de travail
- **PostgreSQL + Redis** : persistance des logs, budgets, sessions — aucune donnée ne sort du périmètre établissement

---

## 3. Mise en place pas à pas

> Tous les fichiers sont créés dans `~/chatia-amue`. C'est le répertoire de travail pour toute la suite.

### Étape 1 — Préparer le serveur

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Docker
# Méthode recommandée : suivre la documentation officielle selon votre distribution
# https://docs.docker.com/engine/install/
# Pour Ubuntu/Debian, le script automatique fonctionne aussi :
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Vérification
docker --version          # Docker version 24+
docker compose version    # Docker Compose version v2+

# Créer l'arborescence du projet
mkdir -p ~/chatia-amue/{config,data/{litellm,openwebui,postgres,redis}}
cd ~/chatia-amue
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

  # Mistral Large — modèle principal, le plus capable
  - model_name: mistral-large
    litellm_params:
      model: mistral/mistral-large-latest
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1            # remplacer si endpoint AMUE dédié
    model_info:
      max_tokens: 131072
      max_input_tokens: 128000
      max_output_tokens: 4096
      description: "Mistral Large — modèle puissant (AMUE)"

  # Mistral Small — plus rapide, moins coûteux, idéal pour tâches simples
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

  # Codestral — spécialisé code
  - model_name: codestral
    litellm_params:
      model: mistral/codestral-latest
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      max_tokens: 32768
      description: "Codestral — assistance à la programmation (AMUE)"

  # Mistral Embed — modèle d'embeddings pour le RAG
  - model_name: mistral-embed
    litellm_params:
      model: mistral/mistral-embed
      api_key: os.environ/MISTRAL_API_KEY
      api_base: https://api.mistral.ai/v1
    model_info:
      mode: embedding
      description: "Embeddings Mistral pour le RAG"
```

> **Note AMUE** : Si l'AMUE vous a communiqué un endpoint dédié (ex: `https://api.amue-mistral.fr/v1`), remplacez tous les `api_base` en conséquence.

---

### Étape 3 — Fichier d'environnement

Créer le fichier `.env` dans `~/chatia-amue` (à ne jamais committer dans un dépôt public) :

```bash
cat > .env << 'EOF'
# === MISTRAL AMUE ===
MISTRAL_API_KEY=sk-votre-cle-mistral-amue-ici

# === LITELLM ===
LITELLM_MASTER_KEY=sk-master-CHANGEZ-MOI-en-prod
LITELLM_SECRET_KEY=jwt-secret-CHANGEZ-MOI-aussi

# === POSTGRESQL ===
POSTGRES_USER=litellm
POSTGRES_PASSWORD=pgpassword-securise-CHANGEZ
POSTGRES_DB=litellm

# === REDIS ===
REDIS_PASSWORD=redis-password-CHANGEZ

# === OPENWEBUI ===
WEBUI_SECRET_KEY=openwebui-secret-CHANGEZ
WEBUI_NAME=ChatIA — Mon Université
WEBUI_URL=https://chat.votre-etablissement.fr

# === PORTS (modifiables si conflit sur l'hôte) ===
PORT_OWUI=3000
PORT_LITELLM=4000

# === VOLUMES (chemins sur l'hôte) ===
VOL_DATA_POSTGRES=./data/postgres
VOL_DATA_REDIS=./data/redis
VOL_DATA_OPENWEBUI=./data/openwebui
EOF

# Sécuriser les permissions
chmod 600 .env
```

---

### Étape 4 — Docker Compose complet

Créer `docker-compose.yml` dans `~/chatia-amue` :

```yaml


services:

  # ─────────────────────────────────
  # Base de données PostgreSQL
  # ─────────────────────────────────
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ${VOL_DATA_POSTGRES}:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
    networks:
      - nw_llm

  # ─────────────────────────────────
  # Cache Redis
  # ─────────────────────────────────
  redis:
    image: redis:7-alpine
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

  # ─────────────────────────────────
  # LiteLLM — passerelle API
  # ─────────────────────────────────
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

  # ─────────────────────────────────
  # OpenWebUI — interface utilisateur
  # ─────────────────────────────────
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "${PORT_OWUI:-3000}:8080"
    environment:
      # Connexion à LiteLLM (mode OpenAI compatible)
      OPENAI_API_BASE_URL: http://litellm:4000/v1
      OPENAI_API_KEY: ${LITELLM_MASTER_KEY}
      # Interface
      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}
      WEBUI_NAME: ${WEBUI_NAME}
      WEBUI_URL: ${WEBUI_URL}
      # Modèle par défaut
      DEFAULT_MODELS: "mistral-large"
      # ⚠️ Mettre "true" pour créer le premier compte admin, puis repasser à "false"
      ENABLE_SIGNUP: "false"
      # Modèle d'embeddings pour le RAG interne
      RAG_EMBEDDING_ENGINE: "openai"
      RAG_EMBEDDING_MODEL: "mistral-embed"
      RAG_OPENAI_API_BASE_URL: http://litellm:4000/v1
      RAG_OPENAI_API_KEY: ${LITELLM_MASTER_KEY}
      # Désactiver la télémétrie
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

---

### Étape 5 — Premier démarrage

```bash
# Se placer dans le répertoire du projet
cd ~/chatia-amue

# Lancer tous les services
docker compose up -d

# Surveiller le démarrage (attendre ~60s la première fois)
docker compose ps

# Logs en temps réel si besoin
docker compose logs -f litellm
docker compose logs -f openwebui

# Vérifier la santé des services
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Résultat attendu — tous les services doivent afficher `healthy` :

```
NAME                    STATUS                    PORTS
chatia-amue-postgres    Up X minutes (healthy)    5432/tcp
chatia-amue-redis       Up X minutes (healthy)    6379/tcp
chatia-amue-litellm     Up X minutes (healthy)    0.0.0.0:4000->4000/tcp
chatia-amue-openwebui   Up X minutes (healthy)    0.0.0.0:3000->8080/tcp
```

---

### Étape 6 — Configuration initiale OpenWebUI

1. Ouvrir http://localhost:3000
2. Créer le **compte administrateur** (premier compte = admin automatiquement)
3. Aller dans **Paramètres → Connexions** et vérifier que LiteLLM est détecté
4. Aller dans **Admin → Utilisateurs** pour créer les comptes

---

## 4. Vérifier que tout fonctionne

On vérifie couche par couche, du bas vers le haut : d'abord l'infrastructure, puis l'API, puis l'interface utilisateur.

---

### 4.1 — Les conteneurs sont-ils tous en vie ?

```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Tous les services doivent afficher `healthy`. Si un service est en `starting` ou `unhealthy`, consulter ses logs :

```bash
docker compose logs --tail=50 litellm
docker compose logs --tail=50 openwebui
```

---

### 4.2 — LiteLLM reçoit-il bien la clé Mistral ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI-en-prod" \
  http://localhost:4000/health | python3 -m json.tool
```

La réponse doit contenir un statut `healthy` pour chaque modèle configuré. Si un modèle remonte `unhealthy`, c'est généralement la clé AMUE qui est incorrecte ou l'endpoint qui ne répond pas — vérifier la valeur de `MISTRAL_API_KEY` dans le `.env`.

---

### 4.3 — Les modèles Mistral sont-ils bien exposés ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI-en-prod" \
  http://localhost:4000/models | python3 -m json.tool
```

On doit voir les quatre modèles déclarés dans `litellm_config.yaml` :

```json
{
  "data": [
    {"id": "mistral-large",  "object": "model"},
    {"id": "mistral-small",  "object": "model"},
    {"id": "codestral",      "object": "model"},
    {"id": "mistral-embed",  "object": "model"}
  ]
}
```

---

### 4.4 — Le chat fonctionne-t-il de bout en bout ?

Ce script Python parcourt tous les cas essentiels en une seule passe.

```python
# recette.py
import openai, sys

BASE_URL = "http://localhost:4000/v1"
API_KEY  = "sk-master-CHANGEZ-MOI-en-prod"

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
    assert rep.usage.completion_tokens > 0, "Pas de tokens comptabilisés"

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

print("=== Recette ChatIA AMUE ===\n")
check("Chat simple (mistral-large)",     test_chat)
check("Streaming (mistral-small)",       test_streaming)
check("Génération de code (codestral)", test_codestral)
check("Embeddings RAG (mistral-embed)", test_embeddings)

print(f"\n{'✅ Recette réussie — passage en production possible.' if ok else '❌ Des tests ont échoué — ne pas passer en production.'}")
sys.exit(0 if ok else 1)
```

```bash
pip install openai
python3 recette.py
```

---

### 4.5 — L'interface OpenWebUI est-elle fonctionnelle ?

Ouvrir http://localhost:3000 et dérouler les vérifications suivantes dans l'ordre :

**Connexion et accès**
- [ ] La page de connexion s'affiche sans erreur
- [ ] La création du compte administrateur (premier lancement) fonctionne
- [ ] Après connexion, les modèles `mistral-large`, `mistral-small` et `codestral` apparaissent dans le menu déroulant

**Conversation**
- [ ] Envoyer *"Bonjour, qui es-tu ?"* → la réponse s'affiche en streaming, en français
- [ ] Le compteur de tokens apparaît sous la réponse (activer dans Paramètres si besoin)
- [ ] Changer de modèle en cours de conversation et constater que la réponse change de style

**RAG — Retrieval Augmented Generation**
- [ ] Cliquer sur l'icône trombone et uploader un PDF (ex: le règlement intérieur de votre établissement)
- [ ] Poser une question dont la réponse se trouve dans ce document
- [ ] Vérifier que la réponse cite correctement le document et n'hallucine pas

**Administration**
- [ ] Aller dans **Admin → Utilisateurs** et créer un second compte de test
- [ ] Se connecter avec ce compte : vérifier qu'il voit les modèles mais pas le panneau admin
- [ ] Retourner en admin et vérifier que la conversation du compte de test apparaît dans les logs

---

### 4.6 — Le fallback fonctionne-t-il en cas de panne ?

Ce test vérifie que LiteLLM bascule automatiquement sur `mistral-small` si `mistral-large` devient indisponible, sans interruption de service.

Simuler une panne du modèle principal en le commentant temporairement dans `config/litellm_config.yaml` :

```yaml
# - model_name: mistral-large
#   litellm_params:
#   ...
```

Redémarrer LiteLLM :

```bash
docker compose restart litellm
```

Envoyer une requête sur `mistral-large` et vérifier que la réponse arrive quand même via le fallback :

```bash
curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI-en-prod" \
  -H "Content-Type: application/json" \
  -d '{"model": "mistral-large", "messages": [{"role": "user", "content": "test fallback"}]}'
```

Vérifier le basculement dans les logs :

```bash
docker compose logs litellm | grep -i fallback
```

Remettre la config d'origine et redémarrer avant de passer en production.

---

### 4.7 — Que faire si quelque chose ne va pas ?

| Symptôme | Cause probable | Action |
|---|---|---|
| Conteneur `litellm` en `unhealthy` | PostgreSQL pas encore prêt | Attendre 30s, relancer `docker compose up -d` |
| Erreur 401 sur `/health` | Mauvaise valeur de `LITELLM_MASTER_KEY` | Vérifier le `.env` et redémarrer |
| Erreur 429 ou `quota exceeded` | Clé AMUE épuisée ou trop de requêtes | Contacter votre référent AMUE |
| Modèles absents dans OpenWebUI | OpenWebUI ne joint pas LiteLLM | Vérifier les logs OpenWebUI, tester `curl http://litellm:4000/models` depuis le conteneur |
| RAG sans résultat pertinent | Modèle d'embedding non configuré | Vérifier les variables `RAG_EMBEDDING_*` dans `docker-compose.yml` |
| Streaming coupé au bout de 30s | Timeout nginx trop court | Augmenter `proxy_read_timeout` à 600s dans la config Nginx |
| Impossible de créer le compte admin | `ENABLE_SIGNUP: "false"` | Passer à `"true"` le temps de créer le compte, puis remettre à `"false"` |

---

## 5. Et ensuite ?

Votre instance ChatIA est maintenant fonctionnelle en local. Vous pouvez vous connecter sur http://localhost:3000, dialoguer avec les modèles Mistral, et tester le RAG avec vos propres documents.

C'est suffisant pour expérimenter, former vos collègues et valider l'intérêt de la solution dans votre contexte.

Pour déployer cette stack dans votre établissement et l'ouvrir à vos utilisateurs, les étapes suivantes feront l'objet d'un second tutoriel :

- Exposition publique sécurisée avec Nginx et certificat TLS
- Authentification via le SSO de votre établissement (Shibboleth / Keycloak)
- Gestion des quotas et budgets par équipe ou département
- Monitoring, alerting et sauvegardes automatisées

---

*Tutoriel rédigé dans le cadre des Ateliers IA Esup 2025/2026 — basé sur les briques open source LiteLLM, OpenWebUI et l'offre Mistral AMUE.*

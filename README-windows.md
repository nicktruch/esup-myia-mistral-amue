# Déployer votre IA souveraine dans le cadre de l'expérimentation Mistral AMUE
## Version maquette locale — Windows 11

> ⚠️ **VERSION DE TRAVAIL — contenu non encore validé**
> Ce document est une adaptation de la version Linux pour une installation de maquette locale sur Windows 11. Il ne couvre pas un déploiement en production.
> N'hésitez pas à nous faire remonter vos questions ou remarques dans les [issues github](https://github.com/EsupPortail/esup-myia-mistral-amue/issues) associées à ce projet.

---

> **Contexte** : Ce tutoriel permet d'installer la stack MyIA souveraine (Mistral AMUE via ILaaS + LiteLLM + OpenWebUI) sur Windows 11 pour expérimentation locale. L'objectif est d'avoir une instance fonctionnelle sur votre poste en moins d'une heure, sans serveur Linux.
>
> **Modèle disponible via l'accord AMUE** : Mistral Medium 3 (`mistral-medium-latest`), hébergé sur l'infrastructure souveraine ILaaS du CINES. Température recommandée : 0.1 ou inférieure.
>
> **RAG / embeddings** : non disponibles via ILaaS — configurer Ollama + `nomic-embed-text` dans OpenWebUI (voir étape 8).

---

| Date | Auteur | Modification | Validation |
|---|---|---|---|
| juin 2026 | | Version initiale Windows 11 | |

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

| Composant | Minimum | Recommandé |
|---|---|---|
| CPU | 4 cœurs (x86-64) | 8 cœurs |
| RAM | 16 Go | 16 Go (8 Go alloués à Docker) |
| Disque | 20 Go libres | SSD recommandé |
| OS | Windows 11 64 bits | Windows 11 64 bits |

> Pas de GPU requis : les inférences sont déléguées à l'API ILaaS (cloud souverain français).

### Logiciels à installer

- **WSL2** — sous-système Linux pour Windows, requis par Docker Desktop
- **Docker Desktop for Windows** (backend WSL2)
- **Git for Windows**
- **Python 3.11+** — optionnel, pour les tests

### Accès & Clés

- **Clé API ILaaS (Mistral AMUE)** : fournie par votre référent numérique via le dispositif AMUE.
  Forme attendue : `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  Endpoint de base : `https://llm.ilaas.fr/v1` — [documentation ILaaS](https://www.ilaas.fr/services-inference/)
  Modèle disponible : **Mistral Medium 3** (`mistral-medium-latest`)

- **Option Mistral standard** : si vous n'avez pas encore de clé ILaaS, une clé Mistral classique (mistral.ai) fonctionne aussi. Voir les blocs commentés dans `config/litellm_config.yaml` et `.env.example`.

- **Sans clé du tout** : l'étape 8 décrit comment utiliser Ollama (modèle local) en attendant votre clé ILaaS.

### Compétences requises

- Bases du terminal Windows (PowerShell)
- Notions de Docker (lancer des conteneurs)
- Notions de LLM et d'API REST (cf. Ateliers 1 & 2 Esup)

---

### RGPD

> Avant de démarrer : les prompts que vous envoyez transitent vers les serveurs de Mistral AI via l'infrastructure ILaaS (CINES) pour traitement. Mistral est un opérateur français, les données sont hébergées en Europe, et l'accord AMUE prévoit la non-utilisation de vos données pour le réentraînement des modèles — mais vérifiez ce point avec votre DPO avant tout usage impliquant des données sensibles. Ne faites jamais transiter de données personnelles d'étudiants ou d'agents via cet outil sans analyse préalable.

---

## 2. Architecture de la solution

Identique à la version Linux — tout tourne dans des conteneurs Docker Desktop (via WSL2) :

```
  ┌──────────────────────────────────────────┐
  │        Windows 11 (WSL2 + Docker)        │
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
          API ILaaS — llm.ilaas.fr
          Mistral Medium 3 (accord AMUE)
```

---

## 3. Mise en place pas à pas

> Tous les fichiers de configuration sont dans le repo cloné. Le répertoire de travail est `%USERPROFILE%\myia`. Ouvrir **Windows Terminal** (PowerShell) et ne plus le quitter.

---

### Étape 1 — Activer WSL2

WSL2 est requis par Docker Desktop. Ouvrir **PowerShell en administrateur** et exécuter :

```powershell
wsl --install
```

Cette commande installe WSL2 et Ubuntu. **Redémarrer** le PC quand demandé.

Après le redémarrage, Ubuntu se lance automatiquement pour finaliser l'installation (créer un utilisateur Linux). Une fois fait, vous pouvez fermer cette fenêtre.

Vérification dans PowerShell :

```powershell
wsl --status
wsl --list --verbose
```

WSL version doit être 2. Si ce n'est pas le cas :

```powershell
wsl --set-default-version 2
```

---

### Étape 2 — Installer Docker Desktop

Télécharger et installer Docker Desktop for Windows (backend WSL2) :
https://docs.docker.com/desktop/install/windows-install/

Une fois installé, lancer Docker Desktop depuis le menu Démarrer. Attendre que l'icône baleine dans la barre des tâches soit stable (non animée).

> **Réglage mémoire important** : Docker Desktop alloue par défaut une partie de votre RAM à WSL2. Pour vérifier ou ajuster : Docker Desktop → Settings → Resources → Memory (recommandé : 8 Go minimum).

Vérification dans PowerShell :

```powershell
docker --version          # Docker version 24+
docker compose version    # Docker Compose version v2+
```

---

### Étape 3 — Installer Git et Python

**Via winget** (gestionnaire de paquets intégré à Windows 11) :

```powershell
winget install --id Git.Git -e --source winget
winget install --id Python.Python.3.11 -e --source winget
```

Fermer et rouvrir PowerShell pour que les chemins soient pris en compte.

Vérification :

```powershell
git --version         # git version 2.x
python --version      # Python 3.11+
```

---

### Étape 4 — Cloner le repo

```powershell
git clone https://github.com/EsupPortail/esup-myia-mistral-amue.git $env:USERPROFILE\myia
cd $env:USERPROFILE\myia

# Créer les répertoires de données
New-Item -ItemType Directory -Force data\postgres, data\redis, data\openwebui
```

Le repo contient déjà tous les fichiers nécessaires :
- `docker-compose.yml` — stack Linux/Windows compatible
- `config\litellm_config.yaml` — modèles et paramètres LiteLLM (préconfiguré pour ILaaS)
- `.env.example` — template de configuration à copier

---

### Étape 5 — Fichier d'environnement

Copier le template :

```powershell
Copy-Item .env.example .env
```

Ouvrir `.env` dans un éditeur :

```powershell
notepad .env
# ou, si VS Code est installé :
code .env
```

Remplacer **toutes** les valeurs de substitution :

| Variable | Rôle | Valeur |
|---|---|---|
| `ILAAS_API_KEY` | Clé API ILaaS fournie par l'AMUE | Votre vraie clé `sk-xxx...` |
| `LITELLM_MASTER_KEY` | Clé admin LiteLLM | `sk-` + 32 caractères aléatoires |
| `LITELLM_SECRET_KEY` | Secret JWT interne | Chaîne aléatoire longue |
| `POSTGRES_PASSWORD` | Mot de passe base de données | Chaîne aléatoire |
| `REDIS_PASSWORD` | Mot de passe cache Redis | Chaîne aléatoire |
| `WEBUI_SECRET_KEY` | Secret session OpenWebUI | Chaîne aléatoire |
| `WEBUI_NAME` | Nom affiché dans l'interface | Ex: `MyIA — Université de Strasbourg` |
| `WEBUI_URL` | URL publique future de l'instance | Ex: `https://chat.votre-etablissement.fr` |

Pour générer des valeurs robustes dans PowerShell :

```powershell
1..6 | ForEach-Object { python -c "import secrets; print(secrets.token_urlsafe(32))" }
```

Sauvegarder le fichier.

> ⚠️ Remplir le `.env` **avant** `docker compose up -d`. Une fois PostgreSQL démarré une première fois, les mots de passe sont gravés en base et ne peuvent plus être changés sans réinitialiser les données.

> **Option Mistral standard** : si vous utilisez une clé Mistral classique (hors accord AMUE/ILaaS), décommentez la ligne `# MISTRAL_API_KEY` dans le `.env` et adaptez `config\litellm_config.yaml` (voir les blocs commentés dans ce fichier).

---

### Étape 6 — Vérifier la configuration LiteLLM

Le fichier `config\litellm_config.yaml` est préconfiguré pour l'endpoint ILaaS :

- **Endpoint actif** : `https://llm.ilaas.fr/v1`
- **Variable de clé** : `ILAAS_API_KEY`
- **Modèle disponible via ILaaS AMUE** : `mistral-medium` (Mistral Medium 3 — `mistral-medium-latest`)
- **Température** : 0.1 (recommandé par ILaaS)
- **RAG / embeddings** : non disponibles via ILaaS — voir étape 8 pour Ollama

Aucune modification n'est nécessaire si vous avez une clé ILaaS.

---

### Étape 7 — Premier démarrage

```powershell
cd $env:USERPROFILE\myia

# Lancer tous les services
docker compose up -d

# Le premier démarrage télécharge les images (~1-2 Go) — patienter
docker compose ps

# Logs si besoin
docker compose logs -f litellm
docker compose logs -f openwebui
```

Résultat attendu après ~60-90 secondes :

```
NAME             STATUS                    PORTS
myia-postgres    Up X minutes (healthy)    5432/tcp
myia-redis       Up X minutes (healthy)    6379/tcp
myia-litellm     Up X minutes (healthy)    0.0.0.0:4000->4000/tcp
myia-openwebui   Up X minutes (healthy)    0.0.0.0:3000->8080/tcp
```

> **Note timeouts** : Docker Desktop sur Windows utilise WSL2, ce qui ajoute une couche de virtualisation. Les migrations Prisma de LiteLLM peuvent prendre jusqu'à 2-3 minutes au premier démarrage.

---

### Étape 7b — Configuration initiale OpenWebUI

1. Ouvrir http://localhost:3000 dans Edge ou Chrome
2. Créer le **compte administrateur** (premier compte = admin automatiquement)
3. Aller dans **Paramètres → Connexions** et vérifier que LiteLLM est détecté
4. Aller dans **Admin → Utilisateurs** pour créer d'autres comptes si besoin

---

### Étape 8 — Tester avec Ollama en attendant la clé ILaaS

Si vous n'avez pas encore votre clé ILaaS, ou pour activer les **embeddings RAG** (non disponibles via ILaaS), vous pouvez utiliser Ollama.

**Installer Ollama for Windows**

Télécharger l'installeur depuis https://ollama.com/download/windows et l'exécuter. Ollama se lance automatiquement en arrière-plan au démarrage de Windows (icône dans la barre des tâches).

Vérification :

```powershell
ollama --version
```

**Télécharger un modèle léger** :

```powershell
ollama pull qwen3:0.6b
```

**Activer le modèle dans LiteLLM** — décommenter la section Ollama dans `config\litellm_config.yaml` :

```yaml
  - model_name: qwen3
    litellm_params:
      model: ollama/qwen3:0.6b
      api_base: http://host.docker.internal:11434
    model_info:
      description: "Qwen3 0.6b — modèle local via Ollama"
```

**Redémarrer LiteLLM** :

```powershell
docker compose restart litellm
```

Le modèle `qwen3` apparaît dans le menu déroulant d'OpenWebUI.

**Pour activer le RAG avec Ollama** (embeddings) :

```powershell
ollama pull nomic-embed-text
```

Puis dans OpenWebUI : **Admin → Settings → Documents**
- Embedding Model Engine → `Ollama`
- Embedding Model → `nomic-embed-text`
- Ollama URL → `http://host.docker.internal:11434`
- Sauvegarder

> **Note** : `host.docker.internal` est l'adresse spéciale Docker Desktop pour joindre un service qui tourne sur l'hôte Windows depuis un conteneur. Elle fonctionne nativement, comme sur macOS.

---

## 4. Vérifier que tout fonctionne

---

### 4.1 — Les conteneurs sont-ils tous en vie ?

```powershell
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Tous les services doivent afficher `healthy`. Si un service est en `starting` ou `unhealthy` :

```powershell
docker compose logs --tail=50 litellm
docker compose logs --tail=50 openwebui
```

---

### 4.2 — LiteLLM reçoit-il bien la clé ILaaS ?

```powershell
# PowerShell 7+
curl -s -H "Authorization: Bearer sk-master-CHANGEZ-MOI" http://localhost:4000/health

# PowerShell 5 (alternatif)
Invoke-RestMethod -Uri http://localhost:4000/health -Headers @{ Authorization = "Bearer sk-master-CHANGEZ-MOI" }
```

La réponse doit contenir un statut `healthy` pour `mistral-medium`. Si le modèle remonte `unhealthy`, vérifier la valeur de `ILAAS_API_KEY` dans le `.env`.

---

### 4.3 — Le modèle est-il bien exposé ?

```powershell
curl -s -H "Authorization: Bearer sk-master-CHANGEZ-MOI" http://localhost:4000/models
```

On doit voir `mistral-medium` dans la réponse.

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
        print(f"OK  {label}")
    except Exception as e:
        print(f"ERREUR  {label} -> {e}")
        ok = False

def test_chat():
    rep = client.chat.completions.create(
        model="mistral-medium",
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
        model="mistral-medium",
        messages=[{"role": "user", "content": "Compte de 1 à 3."}],
        stream=True
    )
    for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            tokens.append(delta)
    assert tokens, "Aucun token reçu en streaming"

print("=== Recette MyIA AMUE — Windows 11 ===\n")
check("Chat simple (mistral-medium)",   test_chat)
check("Streaming (mistral-medium)",     test_streaming)

print(f"\n{'Recette reussie.' if ok else 'Des tests ont echoue.'}")
sys.exit(0 if ok else 1)
```

```powershell
pip install openai
python recette.py
```

---

### 4.5 — L'interface OpenWebUI est-elle fonctionnelle ?

Ouvrir http://localhost:3000 et dérouler les vérifications suivantes :

**Connexion et accès**
- [ ] La page de connexion s'affiche sans erreur
- [ ] La création du compte administrateur fonctionne
- [ ] Le modèle `mistral-medium` apparaît dans le menu déroulant

**Conversation**
- [ ] Envoyer *"Bonjour, qui es-tu ?"* → réponse en streaming, en français

**RAG** (nécessite Ollama + nomic-embed-text configuré — voir étape 8)
- [ ] Uploader un PDF via l'icône trombone dans le chat
- [ ] Poser une question sur le contenu du document
- [ ] Vérifier que la réponse cite le document sans halluciner

---

### 4.6 — Que faire si quelque chose ne va pas ?

| Symptôme | Cause probable | Action |
|---|---|---|
| Docker Desktop ne démarre pas | WSL2 non activé ou virtualisation désactivée | Relancer `wsl --install` en admin et redémarrer ; vérifier la virtualisation dans le BIOS |
| `docker compose` non reconnu | Docker Desktop non démarré | Lancer Docker Desktop depuis le menu Démarrer |
| Conteneur `litellm` en `unhealthy` | PostgreSQL pas encore prêt | Attendre 30s, relancer `docker compose up -d` |
| Erreur 401 sur `/health` | Mauvaise `LITELLM_MASTER_KEY` | Vérifier le `.env` et redémarrer |
| Erreur 429 ou `quota exceeded` | Clé ILaaS épuisée | Contacter votre référent AMUE |
| Modèles absents dans OpenWebUI | OpenWebUI ne joint pas LiteLLM | Vérifier les logs OpenWebUI |
| RAG sans résultat | Modèle d'embedding non configuré | Configurer Ollama + nomic-embed-text dans Admin → Settings → Documents |
| Impossible de créer le compte admin | `ENABLE_SIGNUP: "false"` | Passer à `"true"` le temps de créer le compte, puis remettre à `"false"` |
| Modèle Ollama inaccessible | Ollama non démarré | Vérifier que l'icône Ollama est présente dans la barre des tâches |
| `host.docker.internal` ne répond pas | Docker Desktop non démarré | Relancer Docker Desktop |
| Performances lentes (I/O) | Projet dans le système de fichiers Windows | Déplacer le projet vers le système de fichiers WSL2 (voir note ci-dessous) |

> **Note performances** : Docker Desktop sur Windows accède plus rapidement aux fichiers situés dans le système de fichiers WSL2 (`\\wsl$\Ubuntu\home\<user>\myia`) que dans le système de fichiers Windows (`C:\Users\...`). Si les performances sont insuffisantes, cloner le repo directement depuis un terminal Ubuntu WSL2.

---

### 4.7 — Arrêter et reprendre la maquette

```powershell
# Arrêter sans perdre les données
docker compose stop

# Reprendre
docker compose start

# Tout supprimer (données comprises) — pour repartir de zéro
docker compose down -v
```

---

## 5. Et ensuite ?

Votre maquette tourne sur http://localhost:3000. Vous pouvez expérimenter, tester le RAG avec Ollama, former vos collègues localement.

Pour passer à une instance partagée dans votre établissement, consultez le tutoriel de déploiement en production (version Linux/serveur) qui couvre :

- Exposition publique sécurisée avec Nginx et certificat TLS
- Authentification via le SSO de votre établissement (Shibboleth / Keycloak)
- Gestion des quotas et budgets par équipe ou département
- Monitoring, alerting et sauvegardes automatisées

---

*Adaptation Windows 11 du tutoriel rédigé dans le cadre des Ateliers IA Esup 2025/2026 — basé sur les briques open source LiteLLM, OpenWebUI et l'offre Mistral AMUE via ILaaS (CINES).*

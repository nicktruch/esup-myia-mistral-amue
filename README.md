# MyIA — votre IA souveraine basée sur l'infrastructure Mistral AMUE

Stack open source pour déployer une IA souveraine type ChatGPT ou Claude, basée sur l'offre Mistral AMUE pour l'enseignement supérieur français.

> **Basé sur les Ateliers IA Esup 2025/2026** — Université de Strasbourg ([Morgan Bohn](https://github.com/dotmobo)) / Université de la Polynésie Française ([Nicolas Truchaud](https://github.com/nicktruch))

---

## Stack

| Composant | Rôle |
|---|---|
| [Mistral AI](https://mistral.ai) via [ILaaS](https://www.ilaas.fr/services-inference/) | Mistral Medium 3 souverain (accord AMUE — endpoint `llm.ilaas.fr`) |
| [LiteLLM](https://litellm.ai) | Passerelle API, quotas, load-balancing |
| [OpenWebUI](https://github.com/open-webui/open-webui) | Interface utilisateur type ChatGPT |
| [PostgreSQL](https://www.postgresql.org) | Persistance logs, budgets, clés |
| [Redis](https://redis.io) | Cache des requêtes |

---

## Démarrage rapide

```bash
# 1. Cloner le repo
git clone https://github.com/EsupPortail/esup-myia-mistral-amue.git
cd esup-myia-mistral-amue

# 2. Copier et remplir le fichier de config
cp .env.example .env
# Éditer .env : renseigner ILAAS_API_KEY (clé fournie par votre référent AMUE)
# Option clé Mistral standard (non ILaaS) : voir les commentaires dans .env et config/litellm_config.yaml

# 3. Créer les dossiers de données
mkdir -p data/{postgres,redis,openwebui}

# 4. Lancer
docker compose up -d          # Linux / serveur
# docker compose -f docker-compose.macos.yml up -d   # macOS Apple Silicon

# 5. Ouvrir l'interface
open http://localhost:3000
```

---

## Tutoriels

| Plateforme | Tutoriel |
|---|---|
| 🐧 Linux (Ubuntu / Debian) | [README-linux.md](README-linux.md) |
| 🍎 macOS Apple Silicon (M2/M3) | [README-macos.md](README-macos.md) |
| 🪟 Windows 11 | [README-windows.md](README-windows.md) |

---

## Structure du projet

```
.
├── README.md                   # ce fichier — présentation et démarrage rapide
├── README-linux.md             # tutoriel complet Linux
├── README-macos.md             # tutoriel complet macOS M2
├── docker-compose.yml          # Linux / serveur
├── docker-compose.macos.yml    # macOS Apple Silicon
├── config/
│   └── litellm_config.yaml     # modèles et paramètres LiteLLM
├── .env.example                # template de configuration
├── .gitignore
└── CHANGELOG.md
```

---

## Ressources

- [Wiki GT IA Esup](https://www.esup-portail.org/wiki/x/BgDLVg)
- [Canal RocketChat GT IA](https://rocket.esup-portail.org/channel/GT-IA)
- [Liste de diffusion](mailto:myia@groupes.renater.fr)
- [ILaaS — Service d'inférence LLM](https://www.ilaas.fr/services-inference/)
- [ILaaS — Service STT asynchrone](https://www.ilaas.fr/services-de-retranscription/)

---

## Historique

| Date | Auteur | Modification |
|---|---|---|
| 16 avr. 2026 | Nicolas Truchaud (Université de la Polynésie Française) | Version initiale |
| 20 mai 2026 | David Boit | OS Debian, doc Docker officielle, ports/volumes dans .env, réseau nw_llm |
| 30 mai 2026 | Nicolas Truchaud | Corrections config LiteLLM, healthchecks, ENABLE_SIGNUP, support Ollama, tutoriel macOS M2 |

---

## Licence

MIT — Les modèles Mistral sont soumis aux conditions d'utilisation de Mistral AI et de l'accord AMUE.

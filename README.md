# transfertPro-infra — orchestration Docker

Ce repo contient uniquement l'orchestration Docker du projet (`docker-compose.yml`, `.env.example`).
Le backend et le frontend vivent chacun dans leur propre repo Git, à cloner en tant que dossiers
frères de `transfertPro-infra/` :

- Backend : https://github.com/Alou66/transferePro_api
- Frontend : https://github.com/Alou66/transferePro_web

## Installation

```bash
mkdir transfertPro && cd transfertPro

git clone <url-de-ce-repo> transfertPro-infra
git clone https://github.com/Alou66/transferePro_api.git transfertPro_api
git clone https://github.com/Alou66/transferePro_web.git transferePro_web

cd transfertPro-infra
cp .env.example .env
# éditer .env avec tes propres valeurs (mots de passe, JWT_SECRET, etc.)
```

## Lancer le projet

```bash
cd transfertPro-infra
docker compose up --build
```

- Frontend : http://localhost:8080
- Backend : http://localhost:3001
- Postgres : localhost:5433

## Structure attendue

```
transfertPro/
├── transfertPro-infra/   (ce repo)
│   ├── docker-compose.yml
│   └── .env              (non versionné, à créer depuis .env.example)
├── transfertPro_api/     (repo séparé)
└── transferePro_web/     (repo séparé)
```

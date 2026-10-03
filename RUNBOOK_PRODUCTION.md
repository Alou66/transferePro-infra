# Runbook Production — TransferePro

> Guide opérationnel pour diagnostiquer, corriger et restaurer proprement TransferePro en production.

---

## 1. Architecture de production

```text
                         Internet
                            │
                            ▼
                    transfert-pro.online
                            │
                            ▼
                     ┌─────────────┐
                     │    Caddy    │
                     │ HTTPS / TLS │
                     └──────┬──────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          ┌─────────────┐       ┌─────────────┐
          │  Frontend   │       │   Backend   │
          │    Nginx    │       │ Express API │
          │    :80      │       │    :3001    │
          └─────────────┘       └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │ PostgreSQL  │
                                │    :5432    │
                                └─────────────┘
```

### Composants

| Composant     | Technologie               | Rôle                   |
| ------------- | ------------------------- | ---------------------- |
| Frontend      | React / Vite / TypeScript | Interface utilisateur  |
| Backend       | Express / TypeScript      | API REST               |
| ORM           | Prisma                    | Accès PostgreSQL       |
| Database      | PostgreSQL                | Données applicatives   |
| Reverse proxy | Caddy                     | HTTPS + routage        |
| Containers    | Docker Compose            | Exécution des services |
| Registry      | GitHub Container Registry | Images Docker          |
| CI/CD         | GitHub Actions            | Build + déploiement    |
| Serveur       | AWS EC2                   | Production             |

---

# 2. Règle principale en production

## Ne jamais corriger au hasard

Lorsqu'un problème apparaît :

```text
Incident
   ↓
Observer
   ↓
Identifier le composant
   ↓
Lire les logs
   ↓
Identifier la cause
   ↓
Corriger
   ↓
Vérifier
   ↓
Documenter
```

Éviter les commandes destructives comme :

```bash
docker compose down
docker system prune -a
rm -rf ...
```

sans avoir identifié précisément leur impact.

---

# 3. Connexion à l'EC2

Connexion :

```bash
ssh -i /chemin/vers/transfertpro.pem ubuntu@51.20.56.118
```

Puis :

```bash
cd ~/transfertPro/transfertPro-infra
```

Toujours travailler depuis ce répertoire pour les commandes Docker Compose.

---

# 4. Vérification générale

Première commande en cas d'incident :

```bash
sudo docker compose ps
```

Vérifier que les services principaux sont `Up` / `running` :

```text
postgres
backend
frontend
caddy
```

Ensuite :

```bash
sudo docker compose logs --tail=100
```

Cette commande permet d'avoir une première vision globale.

---

# 5. Vérifier l'API

Health check :

```bash
curl -i https://transfert-pro.online/api/health
```

Résultat attendu :

```http
HTTP/2 200
```

avec :

```json
{
  "success": true,
  "message": "API TransferePro is running"
}
```

### Si le health check échoue

Diagnostiquer le backend :

```bash
sudo docker compose ps backend
```

Puis :

```bash
sudo docker compose logs --tail=200 backend
```

---

# 6. Problème Backend

## Symptômes

Exemples :

```text
HTTP 500
HTTP 502
API indisponible
login impossible
erreurs Prisma
container backend arrêté
```

### Étape 1 — vérifier le container

```bash
sudo docker compose ps backend
```

### Étape 2 — consulter les logs

```bash
sudo docker compose logs --tail=200 backend
```

### Étape 3 — vérifier l'image utilisée

```bash
sudo docker inspect $(sudo docker compose ps -q backend) \
  --format='{{.Config.Image}}'
```

### Étape 4 — tester directement le health check

```bash
curl -i https://transfert-pro.online/api/health
```

---

# 7. Backend après un mauvais déploiement

Le CI/CD possède un rollback automatique.

Le fonctionnement est :

```text
Nouveau commit
      ↓
Nouvelle image SHA
      ↓
Déploiement
      ↓
Health check
      │
      ├── 200 → succès
      │
      └── échec
             ↓
        ancienne version
             ↓
          rollback
```

## Vérifier la version actuellement utilisée

```bash
grep '^API_VERSION=' .env
```

Puis :

```bash
sudo docker inspect $(sudo docker compose ps -q backend) \
  --format='{{.Config.Image}}'
```

Les deux doivent correspondre.

---

# 8. Rollback manuel Backend

Utiliser uniquement si le rollback automatique n'a pas pu être effectué.

### 1. Identifier la version précédente

La version doit être un SHA d'image GHCR connue.

### 2. Modifier uniquement API_VERSION

```bash
nano .env
```

Mettre :

```text
API_VERSION=<SHA_PRECEDENT>
```

### 3. Redéployer

```bash
sudo docker compose pull backend
sudo docker compose up -d backend
```

### 4. Vérifier

```bash
sudo docker compose ps backend
```

Puis :

```bash
curl -i https://transfert-pro.online/api/health
```

### 5. Vérifier l'image

```bash
sudo docker inspect $(sudo docker compose ps -q backend) \
  --format='{{.Config.Image}}'
```

---

# 9. Problème Frontend

## Symptômes

```text
site inaccessible
page blanche
erreur 502
anciens fichiers affichés
assets JavaScript manquants
```

### Vérifier le container

```bash
sudo docker compose ps frontend
```

### Voir les logs

```bash
sudo docker compose logs --tail=200 frontend
```

### Tester le site

```bash
curl -I https://transfert-pro.online/
```

Résultat attendu :

```http
HTTP/2 200
```

---

# 10. Vérifier la version Frontend

```bash
grep '^WEB_VERSION=' .env
```

Puis :

```bash
sudo docker inspect $(sudo docker compose ps -q frontend) \
  --format='{{.Config.Image}}'
```

La version doit correspondre à :

```text
WEB_VERSION=<SHA>
```

et :

```text
ghcr.io/alou66/transferepro-web:<SHA>
```

---

# 11. Rollback manuel Frontend

Si le dernier déploiement frontend est problématique :

```bash
nano .env
```

Modifier uniquement :

```text
WEB_VERSION=<SHA_PRECEDENT>
```

Puis :

```bash
sudo docker compose pull frontend
sudo docker compose up -d frontend
```

Vérifier :

```bash
sudo docker compose ps frontend
```

Puis :

```bash
curl -I https://transfert-pro.online/
```

Et :

```bash
sudo docker inspect $(sudo docker compose ps -q frontend) \
  --format='{{.Config.Image}}'
```

---

# 12. Problème Caddy / HTTPS

## Symptômes

```text
HTTPS inaccessible
502 Bad Gateway
certificat TLS problématique
API et frontend tous les deux inaccessibles
```

### Vérifier Caddy

```bash
sudo docker compose ps caddy
```

### Logs

```bash
sudo docker compose logs --tail=200 caddy
```

### Tester HTTPS

```bash
curl -I https://transfert-pro.online/
```

Puis :

```bash
curl -i https://transfert-pro.online/api/health
```

### Interprétation

Si :

```text
/               → 200
/api/health     → 200
```

Caddy fonctionne probablement correctement.

Si les deux échouent, vérifier Caddy et l'EC2 avant de toucher au frontend ou au backend.

---

# 13. Problème PostgreSQL

## Vérifier le container

```bash
sudo docker compose ps postgres
```

### Logs

```bash
sudo docker compose logs --tail=200 postgres
```

Rechercher notamment :

```text
database system is ready to accept connections
```

ou des erreurs :

```text
connection refused
authentication failed
disk full
database corrupted
```

## Important

Ne jamais supprimer le container ou le volume PostgreSQL pour résoudre un simple problème applicatif.

Éviter absolument :

```bash
docker compose down -v
```

en production.

Cette commande peut supprimer les volumes et donc les données selon la configuration.

---

# 14. Backend + PostgreSQL

Si les logs backend indiquent :

```text
Prisma connection error
Can't reach database
connection refused
```

vérifier dans cet ordre :

```bash
sudo docker compose ps postgres
```

puis :

```bash
sudo docker compose logs --tail=200 postgres
```

puis :

```bash
sudo docker compose logs --tail=200 backend
```

Ne pas modifier les variables de connexion avant d'avoir confirmé le problème.

---

# 15. Container qui redémarre en boucle

Vérifier :

```bash
sudo docker compose ps
```

Puis :

```bash
sudo docker compose logs --tail=200 <service>
```

Exemple :

```bash
sudo docker compose logs --tail=200 backend
```

Consulter également :

```bash
sudo docker inspect $(sudo docker compose ps -q backend) \
  --format='{{.State.Status}} {{.State.ExitCode}} {{.State.Error}}'
```

Chercher la cause dans les logs avant de redémarrer.

---

# 16. EC2 manque de disque

Vérifier :

```bash
df -h
```

Puis :

```bash
sudo docker system df
```

Identifier les gros fichiers :

```bash
sudo du -xh /var/lib/docker 2>/dev/null | sort -h | tail -20
```

### Nettoyage prudent

Le CI/CD utilise déjà :

```bash
sudo docker image prune -f
```

Ne pas utiliser directement :

```bash
docker system prune -a
```

sans vérifier ce qui sera supprimé.

---

# 17. EC2 manque de mémoire

Vérifier :

```bash
free -h
```

Puis :

```bash
docker stats --no-stream
```

Identifier le container consommant le plus :

```text
backend
frontend
postgres
caddy
```

Ensuite consulter ses logs et sa configuration avant toute modification.

---

# 18. Déploiement qui échoue dans GitHub Actions

Commencer par identifier l'étape qui échoue :

```text
Build
↓
Push GHCR
↓
SSH
↓
Pull image
↓
Compose
↓
Container
↓
Health check
```

### Si Build échoue

Le problème est probablement dans le code ou le Dockerfile.

### Si Push GHCR échoue

Vérifier :

* permissions `packages: write`
* authentification GHCR
* nom de l'image

### Si SSH échoue

Vérifier :

* `EC2_HOST`
* `EC2_USER`
* `EC2_SSH_KEY`
* accès SSH AWS

### Si Health check échoue

Se connecter à l'EC2 :

```bash
cd ~/transfertPro/transfertPro-infra
```

Puis :

```bash
sudo docker compose ps
```

et consulter les logs du service concerné.

---

# 19. Vérification après chaque correction

Une correction n'est terminée que lorsque les vérifications suivantes passent.

### Infrastructure

```bash
sudo docker compose ps
```

### Frontend

```bash
curl -I https://transfert-pro.online/
```

Attendu :

```text
HTTP/2 200
```

### Backend

```bash
curl -i https://transfert-pro.online/api/health
```

Attendu :

```text
HTTP/2 200
```

### Versions

Backend :

```bash
grep '^API_VERSION=' .env
```

Frontend :

```bash
grep '^WEB_VERSION=' .env
```

Puis comparer avec les images réellement utilisées :

```bash
sudo docker inspect $(sudo docker compose ps -q backend) \
  --format='{{.Config.Image}}'
```

```bash
sudo docker inspect $(sudo docker compose ps -q frontend) \
  --format='{{.Config.Image}}'
```

---

# 20. Procédure d'urgence

En cas d'indisponibilité totale :

### Étape 1

```bash
cd ~/transfertPro/transfertPro-infra
```

### Étape 2

```bash
sudo docker compose ps
```

### Étape 3

Tester :

```bash
curl -I https://transfert-pro.online/
```

et :

```bash
curl -i https://transfert-pro.online/api/health
```

### Étape 4

Logs :

```bash
sudo docker compose logs --tail=200 caddy
```

```bash
sudo docker compose logs --tail=200 frontend
```

```bash
sudo docker compose logs --tail=200 backend
```

```bash
sudo docker compose logs --tail=200 postgres
```

### Étape 5

Identifier le composant fautif.

### Étape 6

Corriger uniquement ce composant.

### Étape 7

Effectuer les health checks.

### Étape 8

Documenter l'incident.

---

# 21. Règles de sécurité production

## Ne jamais

```bash
docker compose down -v
```

sans procédure de récupération validée.

Ne jamais supprimer PostgreSQL ou ses volumes pour résoudre un problème applicatif.

Ne jamais modifier plusieurs composants simultanément sans raison.

Ne jamais modifier `.env` avec des secrets dans Git.

Ne jamais faire :

```bash
git push
```

depuis l'EC2 pour corriger la production.

Ne jamais utiliser :

```bash
latest
```

comme version de rollback.

---

# 22. Stratégie de déploiement

Les images applicatives sont identifiées par le SHA Git :

```text
ghcr.io/alou66/transferepro-api:<SHA>
ghcr.io/alou66/transferepro-web:<SHA>
```

Le fichier `.env` contient les versions actuellement déployées :

```text
API_VERSION=<SHA>
WEB_VERSION=<SHA>
```

Cette approche permet de savoir exactement quelle version est en production.

---

# 23. Rollback

### Backend

```text
API_VERSION
     ↓
ancienne SHA
     ↓
docker compose pull backend
     ↓
docker compose up -d backend
     ↓
health check
```

### Frontend

```text
WEB_VERSION
     ↓
ancienne SHA
     ↓
docker compose pull frontend
     ↓
docker compose up -d frontend
     ↓
health check
```

---

# 24. Après un incident

Documenter au minimum :

```text
Date :
Heure :
Service concerné :
Symptôme :
Cause :
Impact :
Action effectuée :
Version concernée :
Version restaurée :
Résultat :
Action préventive :
```

Exemple :

```text
Date : 2026-09-28
Service : Backend
Symptôme : API /api/health retourne 500
Cause : erreur introduite par le dernier déploiement
Impact : API indisponible
Action : rollback vers la SHA précédente
Version restaurée : <SHA>
Résultat : HTTP 200
Action préventive : ajouter un test d'intégration
```

---

# 25. Checklist rapide

## 🚨 Incident

```text
[ ] SSH vers EC2
[ ] cd ~/transfertPro/transferePro-infra
[ ] docker compose ps
[ ] tester frontend
[ ] tester API
[ ] consulter les logs
[ ] identifier le composant
[ ] identifier la cause
[ ] corriger
[ ] vérifier
[ ] documenter
```

## ✅ Après correction

```text
[ ] Frontend HTTP 200
[ ] API HTTP 200
[ ] PostgreSQL healthy
[ ] Containers running
[ ] Versions correctes
[ ] Aucun log critique
[ ] Fonctionnalité concernée testée
```

---

# 26. Philosophie d'exploitation

La production doit suivre cette règle :

```text
Observer
   ↓
Comprendre
   ↓
Corriger
   ↓
Vérifier
   ↓
Documenter
```

Et non :

```text
Problème
   ↓
Redémarrer tout
   ↓
Espérer
```

Le but n'est pas simplement de remettre l'application en ligne.

Le but est de **comprendre pourquoi elle est tombée et empêcher la répétition du problème**.

---

# 27. Réinitialisation des données de test

## Usage unique et prévu

Cette procédure sert **exclusivement** au reset initial des données de test,
avant le démarrage réel de la plateforme.

Une fois les premiers agents réels inscrits et les premiers transferts réels
effectués, cette opération ne doit plus être utilisée : elle supprimerait des
comptes agents et des opérations réelles, sans possibilité de récupération.

## Ce que l'opération supprime

```text
transfers          — TOUS les transferts (contenu, bénéficiaires, paiements)
cash_collections   — TOUS les encaissements
users (AGENT)      — TOUS les comptes agents
```

## Ce que l'opération conserve

```text
users (ADMIN)      — le compte administrateur, y compris celui connecté
cities             — toutes les villes
_prisma_migrations — historique des migrations
```

Aucune table, aucun volume PostgreSQL et aucun schéma n'est supprimé. Il ne
s'agit pas d'un `DROP`, d'un `TRUNCATE` ni d'un `DELETE FROM users` sans
condition : la suppression des comptes est explicitement filtrée sur
`role = 'AGENT'`.

## Protections actives

L'opération n'est possible que si les quatre conditions sont réunies :

```text
1. requête authentifiée (JWT)
2. rôle ADMIN
3. corps contenant exactement {"confirmation":"RESET"}
4. variable d'environnement ENABLE_DATA_RESET=true côté backend
```

Si `ENABLE_DATA_RESET` n'est pas à `true`, l'API répond `403` même si
l'administrateur est connecté et saisit correctement `RESET`. C'est le
contrôle qui compte : l'interface masque déjà le bouton dans ce cas, mais ce
n'est pas elle qui protège la base.

## Procédure

Se connecter à l'EC2 puis :

```bash
cd ~/transfertPro/transferePro-infra
```

### 1. Activer temporairement la fonctionnalité

```bash
nano .env
```

Ajouter ou modifier :

```text
ENABLE_DATA_RESET=true
```

Appliquer :

```bash
docker compose up -d backend
```

### 2. Vérifier que l'API autorise bien l'opération

```bash
docker compose exec backend printenv ENABLE_DATA_RESET
```

La sortie doit afficher `true`.

### 3. Effectuer la réinitialisation

Dans l'interface : `Administration → Maintenance`, bouton
`Réinitialiser les données`, puis saisir exactement `RESET`.

### 4. Vérifier le résultat

```bash
docker compose logs --tail=50 backend
```

Une ligne d'audit doit apparaître :

```text
[AUDIT] action=RESET_TRANSACTIONAL_DATA actor=<uuid> role=ADMIN outcome=SUCCESS date=<ISO>
```

Contrôle en base (lecture seule) :

```bash
docker compose exec postgres psql -U transfertpro -d transfertpro -c \
  "SELECT (SELECT count(*) FROM users WHERE role = 'ADMIN') AS admins, (SELECT count(*) FROM users WHERE role = 'AGENT') AS agents, (SELECT count(*) FROM transfers) AS transfers, (SELECT count(*) FROM cash_collections) AS encaissements, (SELECT count(*) FROM cities) AS villes;"
```

Résultat attendu :

```text
 admins | agents | transfers | encaissements | villes
--------+--------+-----------+---------------+--------
      1 |      0 |         0 |             0 |       3
```

Puis se reconnecter avec le compte administrateur pour confirmer que l'accès
fonctionne, et vérifier que l'inscription d'un agent et la liste des villes
fonctionnent toujours.

### 5. Désactiver immédiatement

```bash
nano .env
```

```text
ENABLE_DATA_RESET=false
```

Puis :

```bash
docker compose up -d backend
docker compose exec backend printenv ENABLE_DATA_RESET
```

La sortie doit afficher `false`. L'endpoint refuse désormais toute suppression.

## Ne jamais

```text
[ ] Laisser ENABLE_DATA_RESET=true après le reset initial
[ ] Utiliser cette procédure pour « nettoyer » des données de production
[ ] Remplacer cette procédure par un DROP DATABASE / DROP SCHEMA / docker volume rm
[ ] Contourner la confirmation RESET pour aller plus vite
```

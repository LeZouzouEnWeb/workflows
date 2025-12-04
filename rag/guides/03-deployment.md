# Workflow de déploiement générique

Le workflow `deploy.yml` est un workflow générique réutilisable qui gère :
- Le déploiement sur les serveurs via SFTP (dev, homol, prod)
- La synchronisation automatique des branches (miroir)
- La construction du chemin distant normalisé

## Utilisation

### Configuration basique (DEV - sans miroir)

```yaml
deploy-dev:
  if: github.event.pull_request.merged == true
  uses: LeZouzouEnWeb/workflows/.github/workflows/deploy.yml@main
  secrets:
    SFTP_HOST: ${{ secrets.SFTP_HOST }}
    SFTP_USER: ${{ secrets.SFTP_USER }}
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  with:
    remote_env: dev
    ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
    ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    enable_mirror: false
```

### Configuration avec miroir (HOMOL - develop → homol)

```yaml
deploy-homol:
  if: github.event.pull_request.merged == true
  uses: LeZouzouEnWeb/workflows/.github/workflows/deploy.yml@main
  permissions:
    contents: write  # Requis pour le miroir
  secrets:
    SFTP_HOST: ${{ secrets.SFTP_HOST }}
    SFTP_USER: ${{ secrets.SFTP_USER }}
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  with:
    remote_env: homol
    ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
    ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    enable_mirror: ${{ github.event.pull_request.base.ref == 'homol' && github.event.pull_request.head.ref == 'develop' }}
    mirror_source: develop
    mirror_target: homol
```

### Configuration avec miroir (PROD - homol → main)

```yaml
deploy-prod:
  if: github.event.pull_request.merged == true
  uses: LeZouzouEnWeb/workflows/.github/workflows/deploy.yml@main
  permissions:
    contents: write  # Requis pour le miroir
  secrets:
    SFTP_HOST: ${{ secrets.SFTP_HOST }}
    SFTP_USER: ${{ secrets.SFTP_USER }}
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  with:
    remote_env: prod
    ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
    ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    enable_mirror: ${{ github.event.pull_request.base.ref == 'main' && github.event.pull_request.head.ref == 'homol' }}
    mirror_source: homol
    mirror_target: main
```

## Secrets requis

| Secret | Description |
|--------|-------------|
| `SFTP_HOST` | Hôte du serveur SFTP |
| `SFTP_USER` | Utilisateur SFTP |
| `SSH_PRIVATE_KEY` | Clé privée SSH (au format base64 si nécessaire) |
| `REMOTE_CHEMIN` | Chemin de base sur le serveur distant |

## Variables de dépôt

| Variable | Description |
|----------|-------------|
| `ADRESSE_GLOBAL` | Préfixe global (ex: `/var/www`) |
| `ADRESSE_LOCAL` | Sous-dossier projet/app (ex: `monapp`) |

## Inputs

| Input | Type | Défaut | Description |
|-------|------|--------|-------------|
| `SFTP_PORT` | string | `22` | Port SFTP |
| `ADRESSE_GLOBAL` | string | - | Préfixe global du chemin distant |
| `ADRESSE_LOCAL` | string | - | Sous-dossier projet |
| `remote_env` | string | - | Environnement: `dev`, `homol` ou `prod` |
| `PATH_ORDER` | string | `env-local` | Ordre des segments: `env-local` ou `local-env` |
| `enable_mirror` | boolean | `false` | Active la synchronisation de branche |
| `mirror_source` | string | - | Branche source du miroir (develop ou homol) |
| `mirror_target` | string | - | Branche cible du miroir (homol ou main) |

## Fonctionnalités

### Déploiement SFTP
- Connexion SSH sécurisée avec clé privée
- Synchronisation via `rsync` (idempotente, avec suppression des fichiers retirés)
- Support du fichier `.rsync-ignore` pour exclure des fichiers
- Exclusion automatique des dossiers `.git/` et `.github/`

### Miroir de branche
- Synchronise automatiquement une branche vers une autre
- Exemple: `develop` → `homol` après fusion d'une PR
- Utilise `git reset --hard` et `git push --force-with-lease`
- Évite les commits de fusion (squash)

### Construction du chemin
- Normalisation automatique du chemin distant
- Format: `REMOTE_CHEMIN/ADRESSE_GLOBAL/[ADRESSE_LOCAL/ENV ou ENV/ADRESSE_LOCAL]`
- Configurable via `PATH_ORDER`

### Validations
- Vérification des variables d'entrée
- Test de connexion SSH avant le déploiement
- Vérification que le dossier distant existe (ne crée pas les dossiers)

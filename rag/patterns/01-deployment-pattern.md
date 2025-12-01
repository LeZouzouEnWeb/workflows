# 🚀 Pattern de déploiement SFTP (Dev/Homol/Prod)

## Vue d'ensemble

Ce pattern implémente un pipeline de déploiement SFTP à trois niveaux :
- **Dev** : Déploiement automatique depuis `develop`
- **Homol** : Déploiement après validation depuis `develop` → `homol`
- **Prod** : Déploiement après validation depuis `homol` → `main`

## Architecture

```
develop  ──PR──▶  homol  ──PR──▶  main
   │                │               │
   ▼                ▼               ▼
 /dev/          /homol/          /prod/
```

## Convention des chemins distants

```
REMOTE_CHEMIN / ADRESSE_GLOBAL / ENV / ADRESSE_LOCAL
```

### Variables

- **`REMOTE_CHEMIN`** : Racine hôte (ex: `/homepages/12/d123456789/htdocs`)
- **`ADRESSE_GLOBAL`** : Domaine principal (ex: `corbisier.fr`)
- **`ENV`** : Environnement (`dev`, `homol`, `prod`)
- **`ADRESSE_LOCAL`** : Sous-dossier virtualhost (ex: `web-git`)

### Exemple de chemin final

```
/homepages/12/d123456789/htdocs/corbisier.fr/dev/web-git
```

## Workflows impliqués

### 1. Vérification serveur (pre-merge)

Déclenché lors de l'ouverture/synchronisation d'une PR.

```yaml
name: 🔳 Vérification serveur Dev

on:
  pull_request:
    branches: [develop]
    types: [opened, synchronize]

permissions:
  contents: read

concurrency:
  group: dev-server-check-${{ github.event.pull_request.head.ref }}
  cancel-in-progress: true

env:
  REMOTE_ENV: dev
  REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
  ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
  SFTP_HOST: ${{ secrets.SFTP_HOST }}
  SFTP_USER: ${{ secrets.SFTP_USER }}
  SFTP_PORT: ${{ secrets.SFTP_PORT }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

jobs:
  check:
    name: Vérification connexion & dossier serveur dev
    runs-on: ubuntu-latest
    steps:
      - name: Vérifier variables requises
        run: |
          missing=0
          for var in REMOTE_CHEMIN ADRESSE_GLOBAL ADRESSE_LOCAL SFTP_HOST SFTP_USER SSH_PRIVATE_KEY; do
            if [ -z "${!var}" ]; then
              echo "::error::$var non défini"
              missing=1
            fi
          done
          [ "$missing" -eq 0 ] || exit 1

      - name: Construire le chemin distant
        id: path
        run: |
          REMOTE_PATH="/${REMOTE_CHEMIN}/${ADRESSE_GLOBAL}/${REMOTE_ENV}/${ADRESSE_LOCAL}"
          REMOTE_PATH=$(echo "$REMOTE_PATH" | sed 's#//*#/#g')
          echo "REMOTE_PATH=$REMOTE_PATH" >> $GITHUB_OUTPUT
          echo "::notice::Chemin distant: $REMOTE_PATH"

      - name: Préparer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ env.SSH_PRIVATE_KEY }}

      - name: Ajouter known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p "${SFTP_PORT:-22}" "$SFTP_HOST" >> ~/.ssh/known_hosts

      - name: Tester connexion SSH
        run: |
          ssh -p "${SFTP_PORT:-22}" "$SFTP_USER@$SFTP_HOST" "echo 'Connexion SSH OK'"

      - name: Créer le dossier distant si nécessaire
        run: |
          REMOTE_PATH="${{ steps.path.outputs.REMOTE_PATH }}"
          ssh -p "${SFTP_PORT:-22}" "$SFTP_USER@$SFTP_HOST" \
            "mkdir -p '$REMOTE_PATH' && echo 'Dossier créé/vérifié: $REMOTE_PATH'"
```

### 2. Déploiement (post-merge)

Déclenché après merge d'une PR.

```yaml
name: 🚀 Déploiement Dev

on:
  pull_request:
    branches: [develop]
    types: [closed]

permissions:
  contents: read

concurrency:
  group: dev-deploy-${{ github.event.pull_request.head.ref }}
  cancel-in-progress: true

env:
  REMOTE_ENV: dev
  REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
  ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
  SFTP_HOST: ${{ secrets.SFTP_HOST }}
  SFTP_USER: ${{ secrets.SFTP_USER }}
  SFTP_PORT: ${{ secrets.SFTP_PORT }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

jobs:
  deploy:
    name: Déployer sur le serveur DEV
    if: ${{ github.event.pull_request.merged == true }}
    runs-on: ubuntu-latest
    steps:
      - name: Vérifier variables requises
        run: |
          missing=0
          for var in REMOTE_CHEMIN ADRESSE_GLOBAL ADRESSE_LOCAL SFTP_HOST SFTP_USER SSH_PRIVATE_KEY; do
            if [ -z "${!var}" ]; then
              echo "::error::$var non défini"
              missing=1
            fi
          done
          [ "$missing" -eq 0 ] || exit 1

      - name: Récupérer le code
        uses: actions/checkout@v4

      - name: Construire le chemin distant
        id: path
        run: |
          REMOTE_PATH="/${REMOTE_CHEMIN}/${ADRESSE_GLOBAL}/${REMOTE_ENV}/${ADRESSE_LOCAL}"
          REMOTE_PATH=$(echo "$REMOTE_PATH" | sed 's#//*#/#g')
          echo "REMOTE_PATH=$REMOTE_PATH" >> $GITHUB_OUTPUT
          echo "::notice::Déploiement vers: $REMOTE_PATH"

      - name: Préparer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ env.SSH_PRIVATE_KEY }}

      - name: Ajouter known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p "${SFTP_PORT:-22}" "$SFTP_HOST" >> ~/.ssh/known_hosts

      - name: Vérifier connexion SSH
        run: |
          ssh -p "${SFTP_PORT:-22}" "$SFTP_USER@$SFTP_HOST" "echo 'Connexion SSH OK'"

      - name: Déployer avec rsync
        run: |
          REMOTE_PATH="${{ steps.path.outputs.REMOTE_PATH }}"
          
          # Vérifier que le dossier existe
          ssh -p "${SFTP_PORT:-22}" "$SFTP_USER@$SFTP_HOST" "[ -d \"$REMOTE_PATH\" ]" \
            || { echo "::error::Dossier $REMOTE_PATH n'existe pas"; exit 1; }
          
          # Préparer exclusions
          RSYNC_EXCLUDES=(--exclude='.git/' --exclude='.github/')
          if [ -f ".rsync-ignore" ]; then
            RSYNC_EXCLUDES+=(--exclude-from='.rsync-ignore')
          fi
          
          # Déployer
          rsync -az --delete \
            "${RSYNC_EXCLUDES[@]}" \
            -e "ssh -p ${SFTP_PORT:-22}" \
            ./ "$SFTP_USER@$SFTP_HOST:$REMOTE_PATH/"
          
          echo "✅ Déploiement terminé"
```

## Fichier `.rsync-ignore`

Créer un fichier `.rsync-ignore` à la racine pour exclure des fichiers :

```
# Fichiers de développement
node_modules/
*.log
.env
.env.local

# Fichiers système
.DS_Store
Thumbs.db

# Fichiers temporaires
*.tmp
*.bak
*.swp

# Documentation
docs/
README.md
```

## Gates de protection

### 1. Gate source de PR (homol)

Assure que seules les PR depuis `develop` peuvent aller vers `homol`.

```yaml
name: 🔒 PR de develop vers homol

on:
  pull_request:
    types: [opened, reopened, synchronize]
    branches: [homol]

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  gate:
    name: Vérifier la source de la PR
    runs-on: ubuntu-latest
    steps:
      - name: Contrôle de la branche source
        id: check
        run: |
          if [ "${{ github.base_ref }}" = "homol" ] && [ "${{ github.head_ref }}" != "develop" ]; then
            echo "::error::Seules les PR de develop vers homol sont autorisées"
            exit 1
          fi
          echo "✅ PR valide"

      - name: Commenter si invalide
        if: ${{ failure() }}
        uses: peter-evans/create-or-update-comment@v4
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            🚫 **PR refusée**
            
            Seules les PR provenant de **`develop`** peuvent être fusionnées dans **`homol`**.
```

### 2. Gate source de PR (prod)

Même principe pour `main` : seules les PR depuis `homol`.

```yaml
name: 🔒 PR de homol vers prod

on:
  pull_request:
    types: [opened, reopened, synchronize]
    branches: [main]

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  gate:
    name: Vérifier la source de la PR
    runs-on: ubuntu-latest
    steps:
      - name: Contrôle de la branche source
        run: |
          if [ "${{ github.base_ref }}" = "main" ] && [ "${{ github.head_ref }}" != "homol" ]; then
            echo "::error::Seules les PR de homol vers main sont autorisées"
            exit 1
          fi
```

## Alignement des branches

Pour `homol` et `main`, aligner sur la branche source après le merge :

```yaml
name: 🚀 Déploiement Homol

on:
  pull_request:
    branches: [homol]
    types: [closed]

jobs:
  deploy:
    if: ${{ github.event.pull_request.merged == true }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: homol
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Aligner homol sur develop
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          
          git fetch origin develop
          git reset --hard origin/develop
          git push --force-with-lease origin homol

      - name: Déployer sur serveur homol
        # ... étapes de déploiement SFTP
```

## Configuration des secrets

### Secrets d'Organisation (recommandé)

```yaml
REMOTE_CHEMIN       # /homepages/12/d123456789/htdocs
SFTP_HOST           # access-ssh.ionos.fr
SFTP_USER           # u12345678
SFTP_PORT           # 22
SSH_PRIVATE_KEY     # Clé privée SSH (format PEM)
```

### Variables de Dépôt

```yaml
ADRESSE_GLOBAL      # corbisier.fr
ADRESSE_LOCAL       # web-git
```

## Protection de branches

### Settings → Branches → Branch protection rules

#### Pour `develop` :
- ✅ Require a pull request before merging
- ✅ Require status checks to pass:
  - `Vérification connexion & dossier serveur dev`
  - (optionnel) `codeql`

#### Pour `homol` :
- ✅ Require a pull request before merging
- ✅ Require status checks to pass:
  - `Vérification connexion & dossier serveur homol`
  - `Vérifier la source de la PR` (gate)

#### Pour `main` :
- ✅ Require a pull request before merging
- ✅ Require status checks to pass:
  - `Vérification connexion & dossier serveur prod`
  - `Vérifier la source de la PR` (gate)

## Dépannage

### Échec de connexion SFTP

**Symptômes :**
```
Permission denied (publickey)
```

**Solutions :**
1. Vérifier que `SSH_PRIVATE_KEY` est bien configuré
2. Vérifier que la clé **publique** est installée sur le serveur IONOS
3. Tester la connexion manuellement :
   ```bash
   ssh -i ~/.ssh/id_rsa -p 22 u12345678@access-ssh.ionos.fr
   ```

### Dossier distant introuvable

**Symptômes :**
```
No such file or directory
```

**Solutions :**
1. Lancer le workflow de vérification serveur
2. Vérifier les variables `REMOTE_CHEMIN`, `ADRESSE_GLOBAL`, `ADRESSE_LOCAL`
3. Se connecter en SSH et créer manuellement :
   ```bash
   mkdir -p /homepages/12/d123456789/htdocs/corbisier.fr/dev/web-git
   ```

### Déploiement partiel

**Symptômes :**
Certains fichiers ne sont pas déployés.

**Solutions :**
1. Vérifier le fichier `.rsync-ignore`
2. Ajouter l'option `-v` (verbose) à rsync pour déboguer :
   ```yaml
   rsync -avz --delete ...
   ```

## Bonnes pratiques

### ✅ À faire

1. **Toujours tester en dev d'abord**
2. **Utiliser `.rsync-ignore`** pour exclure les fichiers de développement
3. **Vérifier la connexion** avant chaque déploiement
4. **Utiliser `--delete`** pour synchroniser (supprimer les fichiers obsolètes)
5. **Protéger les branches** avec les status checks requis

### ❌ À éviter

1. **Déployer directement en prod** sans passer par homol
2. **Ignorer les échecs de vérification serveur**
3. **Utiliser des mots de passe** au lieu de clés SSH
4. **Ne pas aligner les branches** (risque de divergence)
5. **Déployer sans protection de branches**

## Ressources

- [webfactory/ssh-agent](https://github.com/webfactory/ssh-agent)
- [actions/checkout](https://github.com/actions/checkout)
- [Guide IONOS SSH](https://www.ionos.fr/assistance/hebergement/acces-ssh-a-votre-hebergement/)

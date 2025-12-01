# 🚀 Workflows GitHub Actions Réutilisables

_Centralisation et standardisation de vos workflows GitHub Actions_

Ce dépôt fournit un ensemble complet de **workflows réutilisables** pour automatiser vos déploiements, sécuriser vos PR et maintenir la qualité du code.

## 📋 Table des matières

- [Aperçu des workflows](#-aperçu-des-workflows)
- [Installation rapide](#-installation-rapide)
- [Utilisation](#-utilisation)
  - [Workflows prêts à l'emploi](#workflows-prêts-à-lemploi)
  - [Workflows réutilisables](#workflows-réutilisables)
- [Configuration](#-configuration)
- [Exemples](#-exemples)
- [Documentation complète](#-documentation-complète)
- [Dépannage](#-dépannage)

---

## 📊 Aperçu des workflows

### Workflows de déploiement

| Workflow                      | Fichier                     | Déclenchement              | Description                                                     |
| ----------------------------- | --------------------------- | -------------------------- | --------------------------------------------------------------- |
| 🔳 Vérification serveur Dev   | `dev-server-check-pr.yml`   | PR → `develop` (open/sync) | Teste la connexion SSH/SFTP et crée le dossier `/dev` si besoin |
| 🚀 Déploiement Dev            | `dev-deploy.yml`            | PR → `develop` (merged)    | Déploie sur l'environnement **dev** via `rsync` après merge     |
| 🔳 Vérification serveur Homol | `homol-server-check-pr.yml` | PR → `homol` (open/sync)   | Vérifie l'accès SFTP et prépare le dossier `/homol`             |
| 🚀 Déploiement Homol          | `homol-deploy.yml`          | PR → `homol` (merged)      | Aligne `homol` sur `develop` puis déploie en SFTP               |
| 🔳 Vérification serveur Prod  | `prod-server-check-pr.yml`  | PR → `main` (open/sync)    | Vérifie SFTP et prépare le dossier `/prod`                      |
| 🚀 Déploiement Prod           | `prod-deploy.yml`           | PR → `main` (merged)       | Aligne `main` sur `homol` puis déploie l'environnement `prod`   |

### Workflows de gestion des PR

| Workflow                           | Fichier                          | Déclenchement          | Description                                                  |
| ---------------------------------- | -------------------------------- | ---------------------- | ------------------------------------------------------------ |
| 🔒 PR de develop vers homol        | `homol-check-pr-depuis-dev.yml`  | PR → `homol`           | Refuse les PR qui ne proviennent pas de `develop`            |
| 🔒 PR de homol vers main           | `prod-check-pr-depuis-homol.yml` | PR → `main`            | Refuse les PR qui ne proviennent pas de `homol`              |
| 🔗 Lier les tickets à la PR        | `link-issues-in-pr.yml`          | PR → `develop`         | Force la présence d'un ticket (#123, ticket-123, ABC-123…)   |
| 🔗 Lier au projet GitHub           | `link-project-in-pr.yml`         | PR (any)               | Ajoute automatiquement les PR et issues à un projet GitHub   |
| 🔄 Mettre à jour le titre de la PR | `update-pr-title.yml`            | PR → `develop`         | Normalise le titre de la PR depuis le nom de branche         |
| 📝 Commenter et clôturer le ticket | `comment-and-close-ticket.yml`   | PR → `develop` (merge) | Commente et ferme l'issue liée en listant les commits mergés |

### Workflows de maintenance

| Workflow                             | Fichier                         | Déclenchement  | Description                                               |
| ------------------------------------ | ------------------------------- | -------------- | --------------------------------------------------------- |
| 🗑️ Supprimer la branche après fusion | `delete-branch-after-merge.yml` | PR (closed)    | Supprime automatiquement la branche source (configurable) |
| 🔣 Analyse CodeQL                    | `codeql.yml`                    | Push/PR + cron | Analyse statique JS/TS si du code est détecté             |

---

## ⚡ Installation rapide

### Option 1 : Utilisation directe (copier les workflows)

Copie les fichiers nécessaires dans `.github/workflows/` de ton dépôt :

```plaintext
.github/
└── workflows/
    ├── codeql.yml
    ├── comment-and-close-ticket.yml
    ├── delete-branch-after-merge.yml
    ├── dev-deploy.yml
    ├── dev-server-check-pr.yml
    ├── homol-check-pr-depuis-dev.yml
    ├── homol-deploy.yml
    ├── homol-server-check-pr.yml
    ├── link-issues-in-pr.yml
    ├── link-project-in-pr.yml
    ├── prod-check-pr-depuis-homol.yml
    ├── prod-deploy.yml
    ├── prod-server-check-pr.yml
    └── update-pr-title.yml
```

### Option 2 : Workflows réutilisables (recommandé)

Appelle les workflows depuis ce dépôt central :

```yaml
jobs:
  deploy:
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@v1
    secrets: inherit
```

Voir les [exemples complets](#-exemples) ci-dessous.

---

## 🔧 Configuration

### Variables (Repository variables)

Configure dans **Settings → Secrets and variables → Actions → Variables** :

- `ADRESSE_GLOBAL` : domaine maître (ex. `corbisier.fr`)
- `ADRESSE_LOCAL` : répertoire virtualhost (ex. `web-git`)

### Secrets (Repository ou Organization)

Configure dans **Settings → Secrets and variables → Actions → Secrets** :

- `REMOTE_CHEMIN` : racine hôte (ex. `/homepages/XX/dXXXXXXXX/htdocs`)
- `SFTP_HOST` : hôte SSH/SFTP (ex. `ssh.strato.com` ou `access-ssh.ionos.fr`)
- `SFTP_USER` : utilisateur SFTP autorisé
- `SFTP_PORT` : port SSH (défaut `22`, optionnel)
- `SSH_PRIVATE_KEY` : clé privée SSH (correspondant à la clé publique installée côté serveur)
- `ADD_TO_PROJECT_PAT` : Personal Access Token pour lier au projet GitHub (optionnel)

### Convention de chemins (déploiement)

Les déploiements suivent la structure :

```plaintext
REMOTE_CHEMIN/ADRESSE_GLOBAL/ENV/ADRESSE_LOCAL
```

Exemple :

```plaintext
/homepages/12/d123456789/htdocs/corbisier.fr/dev/web-git
```

---

## 📚 Exemples

Le dossier `examples/` contient des exemples d'utilisation prêts à l'emploi :

### `examples/consumers/` - Utilisation avec version majeure (recommandé)

Exemples utilisant la dernière version stable d'une version majeure :

```yaml
name: 🚀 Déploiement DEV

on:
  pull_request:
    branches: [develop]
    types: [closed]

jobs:
  deploy:
    if: github.event.pull_request.merged == true
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@v1
    permissions:
      contents: read
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SFTP_USER: ${{ secrets.SFTP_USER }}
      SFTP_PORT: ${{ secrets.SFTP_PORT }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    with:
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
```

### `examples/consumers-pinned/` - Utilisation avec version fixe

Exemples utilisant une version spécifique (contrôle maximal) :

```yaml
jobs:
  deploy:
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@v1.0.0
    permissions:
      contents: read
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SFTP_USER: ${{ secrets.SFTP_USER }}
      SFTP_PORT: ${{ secrets.SFTP_PORT }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    with:
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
```

### `examples/consumers-latest/` - Utilisation avec @main (développement uniquement)

⚠️ **Attention** : Utilise la version en cours de développement (peut contenir des breaking changes)

```yaml
jobs:
  deploy:
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@main
    secrets: inherit
```

**Recommandations** :

- ✅ **Production** : Utilise `@v1` (version majeure stable)
- ✅ **Production critique** : Utilise `@v1.0.0` (version exacte)
- ⚠️ **Développement/Test** : Utilise `@main` (dernières fonctionnalités)

---

## 🎯 Utilisation

### Workflows prêts à l'emploi

Les workflows du dossier `.github/workflows/` sont prêts à être copiés directement dans ton dépôt.

### Workflows réutilisables

Tous les workflows sont également disponibles en mode `workflow_call` pour être appelés depuis d'autres dépôts.

#### Avec héritage des secrets (simple)

```yaml
name: 🚀 Déploiement DEV

on:
  pull_request:
    branches: [develop]
    types: [closed]

jobs:
  deploy:
    if: github.event.pull_request.merged == true
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@v1
    secrets: inherit
```

#### Avec mapping explicite (contrôle fin)

```yaml
jobs:
  deploy:
    if: github.event.pull_request.merged == true
    uses: LeZouzouEnWeb/workflows/.github/workflows/dev-deploy.yml@v1
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SFTP_USER: ${{ secrets.SFTP_USER }}
      SFTP_PORT: ${{ secrets.SFTP_PORT }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    with:
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
```

---

## 🛡️ Protection de branches recommandée

### Configuration dans GitHub

**Settings → Branches → Branch protection rules → Add rule**

#### Branche `develop`

- ✅ Require a pull request before merging
- ✅ Require status checks to pass before merging :
  - `Vérification connexion & dossier serveur dev`
  - `Detecter et lier les tickets dans la PR` (optionnel)
  - `codeql` (si pertinent)

#### Branche `homol`

- ✅ Require a pull request before merging
- ✅ Require status checks to pass before merging :
  - `Verifier la source de la PR`
  - `Verification connexion & dossier serveur homol`
  - `codeql`

#### Branche `main`

- ✅ Require a pull request before merging
- ✅ Require status checks to pass before merging :
  - `Verifier la source de la PR vers main`
  - `Verification connexion & dossier serveur prod`
  - `codeql`

> ℹ️ **Note** : Les workflows de déploiement s'exécutent **après** le merge et ne peuvent pas être requis comme checks. Pour qu'un job apparaisse dans la liste, exécute-le au moins une fois via une PR de test.

---

## 📖 Documentation complète

La documentation détaillée est disponible dans le dossier `rag/` :

### 📂 Structure de la documentation

- **`rag/fondamentaux/`** : Concepts de base (workflow_call, inputs/outputs, secrets, permissions, concurrency)
- **`rag/patterns/`** : Patterns de déploiement, gates de PR, versioning, protection de branches
- **`rag/guides/`** : Guides pratiques (conversion, appel, versioning, secrets organisation, debugging)
- **`rag/templates/`** : Templates réutilisables et exemples complets

👉 [Consulter la documentation complète](./rag/README.md)

---

## 🎨 Personnalisation

### Exclusions de déploiement

Créer un fichier `.rsync-ignore` à la racine du dépôt :

```plaintext
.git/
.github/
node_modules/
*.log
.env
```

### Branches protégées

Modifier `FORBIDDEN_BRANCHES` dans `delete-branch-after-merge.yml` :

```yaml
env:
  FORBIDDEN_BRANCHES: develop,homol,staging,main
```

### Format des titres de PR

Adapter la logique dans `update-pr-title.yml` pour d'autres préfixes :

```yaml
# Supporte : feature, fix, hotfix, bugfix, release, chore, task, refactor
```

---

## 🔍 Dépannage

### Échec de connexion SSH/SFTP

**Symptômes** : `Permission denied`, `Connection refused`, `Host key verification failed`

**Solutions** :

1. Vérifier `SFTP_HOST`, `SFTP_USER`, `SFTP_PORT` dans les secrets
2. Vérifier que `SSH_PRIVATE_KEY` est correctement formaté (inclure `-----BEGIN OPENSSH PRIVATE KEY-----`)
3. Vérifier que la clé publique correspondante est installée dans `~/.ssh/authorized_keys` sur le serveur
4. Tester la connexion manuellement : `ssh -p PORT USER@HOST`

### Dossier distant introuvable

**Symptômes** : `No such file or directory`, déploiement échoue

**Solutions** :

1. Lancer le workflow "Vérification serveur" correspondant (crée automatiquement les dossiers)
2. Vérifier la construction du chemin : `REMOTE_CHEMIN/ADRESSE_GLOBAL/ENV/ADRESSE_LOCAL`
3. Vérifier les valeurs de `REMOTE_CHEMIN`, `ADRESSE_GLOBAL`, `ADRESSE_LOCAL`

### PR bloquée par "Lier les tickets à la PR"

**Symptômes** : Le workflow échoue avec "Aucun ticket/issue détecté"

**Solutions** :

1. Ajouter une référence dans le titre : `[#123] Mon titre`
2. Ajouter dans la description : `Fixes #123`, `Closes #123`, `Resolves #123`
3. Nommer la branche : `ticket-123-description` ou `feature/ticket-123`
4. Utiliser un format projet : `ABC-123`

### Branche non supprimée après merge

**Symptômes** : La branche existe toujours après merge de la PR

**Solutions** :

1. Consulter le commentaire automatique sur la PR pour la raison
2. Vérifier que la branche n'est pas dans `FORBIDDEN_BRANCHES`
3. Vérifier que la PR provient du même dépôt (pas d'un fork)
4. Vérifier que la PR a bien été **mergée** (pas juste fermée)

### Workflow ne se déclenche pas

**Solutions** :

1. Vérifier les conditions `on:` (branches, types d'événements)
2. Vérifier les conditions `if:` dans les jobs
3. Consulter l'onglet "Actions" pour voir si le workflow est listé
4. Vérifier les permissions du `GITHUB_TOKEN`

---

## 🔗 Liens utiles

- [Documentation officielle GitHub Actions](https://docs.github.com/en/actions)
- [Reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

---

## 📝 Versioning

Ce dépôt suit le versioning sémantique (SemVer) avec tags mobiles pour faciliter l'utilisation :

- **@v1** : Tag mobile pointant vers la dernière version 1.x.x (recommandé pour la production)
- **@v1.0.0** : Version spécifique (pour un contrôle strict)
- **@main** : Branche de développement (non recommandé pour la production)

### Structure des versions

```plaintext
v1.0.0 ← Version exacte (patch)
  ↑
 v1   ← Tag mobile (majeure) - RECOMMANDÉ
  ↑
main  ← Développement
```

### Créer une nouvelle version

```bash
# 1. Créer le tag de version exacte
git tag -a v1.0.0 -m "Version 1.0.0 - Ajout des workflows de déploiement"
git push origin v1.0.0

# 2. Créer ou mettre à jour le tag majeur mobile
git tag -fa v1 -m "Update v1 to v1.0.0"
git push origin v1 --force
```

### Mettre à jour une version existante

```bash
# Pour une nouvelle version mineure (1.1.0) ou patch (1.0.1)
git tag -a v1.1.0 -m "Version 1.1.0 - Nouvelles fonctionnalités"
git push origin v1.1.0

# Mettre à jour le tag v1 pour pointer vers v1.1.0
git tag -fa v1 -m "Update v1 to v1.1.0"
git push origin v1 --force
```

### Utilisation recommandée

| Contexte | Version à utiliser | Exemple |
|----------|-------------------|---------|
| Production | Tag majeur mobile | `@v1` |
| Production critique | Version exacte | `@v1.0.0` |
| Développement/Test | Branche main | `@main` |

---

## 🤝 Contribution

Les contributions sont les bienvenues ! Voir la structure du dépôt :

```plaintext
workflows/
├── .github/
│   └── workflows/          # Workflows prêts à l'emploi ET réutilisables
│       ├── codeql.yml
│       ├── comment-and-close-ticket.yml
│       ├── delete-branch-after-merge.yml
│       ├── dev-deploy.yml
│       ├── dev-server-check-pr.yml
│       ├── homol-check-pr-depuis-dev.yml
│       ├── homol-deploy.yml
│       ├── homol-server-check-pr.yml
│       ├── link-issues-in-pr.yml
│       ├── link-project-in-pr.yml
│       ├── prod-check-pr-depuis-homol.yml
│       ├── prod-deploy.yml
│       ├── prod-server-check-pr.yml
│       └── update-pr-title.yml
├── examples/
│   ├── consumers/          # Exemples avec @v1 (recommandé)
│   ├── consumers-pinned/   # Exemples avec @v1.0.0 (version fixe)
│   └── consumers-latest/   # Exemples avec @main (développement)
├── rag/                    # Documentation complète
│   ├── fondamentaux/
│   ├── patterns/
│   ├── guides/
│   └── templates/
└── README.md               # Ce fichier
```

---

**Bon déploiement !** 🚀

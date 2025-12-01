# 📞 Appeler un workflow réutilisable

Guide complet pour utiliser des workflows réutilisables depuis vos dépôts consommateurs.

## Syntaxe de base

```yaml
jobs:
  job-name:
    uses: owner/repo/.github/workflows/workflow-file.yml@ref
    with:
      input1: value1
      input2: value2
    secrets:
      SECRET1: ${{ secrets.SECRET1 }}
```

## Anatomie d'un appel

### 1. Référence au workflow

```yaml
uses: owner/repo/.github/workflows/workflow-file.yml@ref
```

| Partie | Description | Exemple |
|--------|-------------|---------|
| `owner` | Propriétaire du dépôt | `LeZouzouEnWeb` |
| `repo` | Nom du dépôt | `corbidev-actions-central` |
| `path` | Chemin vers le workflow | `.github/workflows/deploy.yml` |
| `ref` | Référence (tag/branche/SHA) | `v1`, `main`, `a1b2c3d` |

### 2. Transmission des inputs

```yaml
with:
  environment: dev
  version: 1.2.3
  dry_run: false
```

### 3. Transmission des secrets

#### Option A : Héritage complet

```yaml
secrets: inherit
```

Tous les secrets disponibles sont transmis.

#### Option B : Mapping explicite

```yaml
secrets:
  DEPLOY_KEY: ${{ secrets.SSH_KEY }}
  API_TOKEN: ${{ secrets.SLACK_WEBHOOK }}
```

Seuls les secrets listés sont transmis.

## Étapes pour appeler un workflow

### Étape 1 : Identifier le workflow réutilisable

Consultez la documentation du workflow pour connaître :
- Les **inputs** requis et optionnels
- Les **secrets** nécessaires
- Les **outputs** disponibles
- Les **permissions** à définir

### Étape 2 : Créer le workflow consommateur

```yaml
name: Mon workflow consommateur

on:
  push:
    branches: [main]

permissions:
  contents: read  # Définir selon les besoins du workflow appelé

jobs:
  call-reusable:
    uses: owner/repo/.github/workflows/reusable.yml@v1
    with:
      # inputs requis
    secrets:
      # secrets requis
```

### Étape 3 : Configurer les variables et secrets

#### Dans le dépôt consommateur

**Settings** → **Secrets and variables** → **Actions**

- **Variables** : valeurs non sensibles (domaines, chemins publics)
- **Secrets** : credentials, tokens, clés privées

### Étape 4 : Tester l'appel

1. Créer une branche de test
2. Pousser le nouveau workflow
3. Vérifier l'exécution dans **Actions**
4. Corriger si nécessaire

### Étape 5 : Configurer les protections de branches

Si le workflow est un gate (vérification pré-merge) :

**Settings** → **Branches** → **Branch protection rules**
- Activer **Require status checks to pass**
- Sélectionner le job du workflow dans la liste

## Exemples pratiques

### Exemple 1 : Déploiement simple

```yaml
name: Déploiement Dev

on:
  push:
    branches: [develop]

permissions:
  contents: read

jobs:
  deploy:
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@v1
    with:
      environment: dev
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SFTP_USER: ${{ secrets.SFTP_USER }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Exemple 2 : Multi-environnements avec matrice

```yaml
name: Déploiement multi-env

on:
  workflow_dispatch:
    inputs:
      environments:
        description: 'Environnements à déployer (séparés par des espaces)'
        required: true
        default: 'dev homol'

jobs:
  parse-environments:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.parse.outputs.matrix }}
    steps:
      - id: parse
        run: |
          ENVS='${{ inputs.environments }}'
          MATRIX=$(echo "$ENVS" | jq -R -c 'split(" ")')
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  deploy:
    needs: parse-environments
    strategy:
      matrix:
        environment: ${{ fromJSON(needs.parse-environments.outputs.matrix) }}
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@v1
    with:
      environment: ${{ matrix.environment }}
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    secrets: inherit
```

### Exemple 3 : Chaînage avec outputs

```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    uses: owner/repo/.github/workflows/build.yml@v1
    with:
      version: 1.2.3
    secrets: inherit

  deploy:
    needs: build
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      artifact_url: ${{ needs.build.outputs.artifact_url }}
      environment: prod
    secrets: inherit

  notify:
    needs: [build, deploy]
    runs-on: ubuntu-latest
    steps:
      - name: Notifier
        run: |
          echo "Build: ${{ needs.build.outputs.artifact_url }}"
          echo "Deploy: ${{ needs.deploy.outputs.deployment_url }}"
```

### Exemple 4 : Déploiement avec approbation

```yaml
name: Déploiement Production

on:
  push:
    branches: [main]

jobs:
  # Déploiement dev automatique
  deploy-dev:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: dev
    secrets: inherit

  # Attente d'approbation pour prod
  approval:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: production  # Environnement GitHub avec reviewers
    steps:
      - run: echo "Approbation reçue"

  # Déploiement prod après approbation
  deploy-prod:
    needs: approval
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: prod
    secrets: inherit
```

## Gestion des versions

### Stratégie de versioning

#### Version majeure (v1)

```yaml
uses: owner/repo/.github/workflows/deploy.yml@v1
```

- ✅ **Recommandé pour la plupart des cas**
- Reçoit les mises à jour mineures et patches (v1.x.x)
- Équilibre entre stabilité et mises à jour

#### Version complète (v1.2.3)

```yaml
uses: owner/repo/.github/workflows/deploy.yml@v1.2.3
```

- ✅ **Pour les environnements critiques**
- Version fixe, pas de mises à jour automatiques
- Nécessite des mises à jour manuelles

#### Branche (main)

```yaml
uses: owner/repo/.github/workflows/deploy.yml@main
```

- ⚠️ **Pour le développement uniquement**
- Toujours la dernière version
- Peut casser à tout moment

#### SHA complet

```yaml
uses: owner/repo/.github/workflows/deploy.yml@a1b2c3d4e5f6789...
```

- ✅ **Pour l'audit et la conformité**
- Immuable, parfaitement reproductible
- Difficile à maintenir

### Mettre à jour vers une nouvelle version

```yaml
# Avant
uses: owner/repo/.github/workflows/deploy.yml@v1.2.3

# Après
uses: owner/repo/.github/workflows/deploy.yml@v1.3.0
```

Processus :
1. Lire les notes de version (CHANGELOG)
2. Vérifier les breaking changes
3. Mettre à jour la référence
4. Tester dans un environnement de dev
5. Déployer en production

## Transmission des secrets

### Cas 1 : Secrets d'Organisation

Les secrets d'Organisation sont automatiquement disponibles dans les dépôts autorisés.

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets:
      # Secrets d'Organisation (infrastructure)
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Cas 2 : Secrets de Dépôt

Les secrets spécifiques au dépôt doivent être créés dans chaque dépôt consommateur.

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets:
      # Secrets de Dépôt (spécifiques au projet)
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      API_KEY: ${{ secrets.API_KEY }}
```

### Cas 3 : Mélange Organisation + Dépôt

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets:
      # Organisation
      SSH_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      # Dépôt
      DB_URL: ${{ secrets.DATABASE_URL }}
      API_KEY: ${{ secrets.API_KEY }}
```

### Cas 4 : Héritage complet (simplicité)

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets: inherit
```

⚠️ **Attention** : Tous les secrets sont transmis. À utiliser avec précaution.

## Permissions requises

Les permissions doivent être définies dans le **workflow consommateur**, pas dans le workflow réutilisable.

### Exemple : Workflow qui commente des PR

```yaml
name: Vérification PR

on:
  pull_request:
    branches: [develop]

permissions:
  contents: read         # Lire le code
  pull-requests: write   # ← REQUIS pour commenter

jobs:
  check:
    uses: owner/repo/.github/workflows/pr-check.yml@v1
    secrets: inherit
```

Sans `pull-requests: write`, le workflow appelé ne pourra pas commenter.

## Déboguer un appel de workflow

### Erreur : "Resource not accessible"

**Cause** : Permissions manquantes dans le consommateur.

**Solution** :
```yaml
permissions:
  contents: read
  pull-requests: write  # ← Ajouter selon les besoins
```

### Erreur : "Required input not provided"

**Cause** : Input obligatoire non fourni.

**Solution** :
```yaml
with:
  environment: dev  # ← Ajouter l'input manquant
```

### Erreur : "Secret not found"

**Cause** : Secret non défini dans le dépôt consommateur.

**Solution** : Configurer le secret dans **Settings → Secrets and variables → Actions**.

### Workflow ne démarre pas

**Cause** : Référence incorrecte ou workflow non trouvé.

**Vérifications** :
- Le chemin est correct : `.github/workflows/file.yml`
- Le tag/branche existe
- Le dépôt est accessible (privé vs public)

## Bonnes pratiques

### ✅ À faire

1. **Toujours définir les permissions** explicitement
   ```yaml
   permissions:
     contents: read
     pull-requests: write
   ```

2. **Utiliser des tags** en production
   ```yaml
   uses: owner/repo/.github/workflows/deploy.yml@v1
   ```

3. **Documenter les prérequis** dans le README
   - Variables requises
   - Secrets requis
   - Permissions nécessaires

4. **Tester en dev** avant la production
   ```yaml
   uses: owner/repo/.github/workflows/deploy.yml@main  # Test
   uses: owner/repo/.github/workflows/deploy.yml@v1    # Prod
   ```

5. **Mapper explicitement les secrets critiques**
   ```yaml
   secrets:
     PROD_KEY: ${{ secrets.PRODUCTION_KEY }}
   ```

### ❌ À éviter

1. **`secrets: inherit`** sans comprendre les implications
2. **Références instables** (`@main`) en production
3. **Oublier les permissions** dans le consommateur
4. **Hardcoder des valeurs** qui devraient être des inputs
5. **Ne pas lire la documentation** du workflow appelé

## Checklist avant d'appeler un workflow

- [ ] Lire la documentation du workflow réutilisable
- [ ] Identifier les inputs requis
- [ ] Identifier les secrets nécessaires
- [ ] Configurer les variables dans le dépôt
- [ ] Configurer les secrets dans le dépôt
- [ ] Définir les permissions dans le workflow consommateur
- [ ] Choisir une référence appropriée (@v1)
- [ ] Tester avec un déclenchement manuel ou une PR de test
- [ ] Vérifier les logs d'exécution
- [ ] Configurer les protections de branches si nécessaire

## Ressources

- [Template de workflow consommateur](../templates/consumer-template.yml)
- [Fondamentaux - workflow_call](../fondamentaux/01-workflow-call.md)
- [Fondamentaux - Secrets](../fondamentaux/03-secrets.md)
- [Fondamentaux - Permissions](../fondamentaux/04-permissions.md)

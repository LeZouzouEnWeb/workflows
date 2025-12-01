# ⏸️ Concurrency - Contrôle d'exécution

## Qu'est-ce que la concurrency ?

La concurrency contrôle l'exécution simultanée de workflows pour :
- **Éviter les conflits** (ex: déploiements simultanés)
- **Économiser des ressources** (annuler les runs obsolètes)
- **Garantir l'ordre** des opérations

## Syntaxe de base

```yaml
concurrency:
  group: deploy-prod
  cancel-in-progress: true
```

### Composants

- **`group`** : Identifiant du groupe de concurrency (string)
- **`cancel-in-progress`** : Annuler les runs en cours du même groupe (boolean)

## Niveaux de concurrency

### Au niveau workflow

```yaml
name: Déploiement

on: [push]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Déploiement"
```

### Au niveau job

```yaml
name: Multi-jobs

on: [push]

jobs:
  build:
    concurrency:
      group: build-${{ github.ref }}
      cancel-in-progress: true
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build"
      
  deploy:
    needs: build
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: false  # Ne pas annuler les déploiements
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

## Groupes de concurrency dynamiques

### Par branche

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

- `main` → groupe `deploy-refs/heads/main`
- `develop` → groupe `deploy-refs/heads/develop`
- `feature/xyz` → groupe `deploy-refs/heads/feature/xyz`

### Par PR

```yaml
concurrency:
  group: pr-checks-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Chaque PR a son propre groupe, permettant :
- ✅ Annulation des runs obsolètes sur **la même PR**
- ✅ Exécution parallèle pour **différentes PR**

### Par environnement

```yaml
concurrency:
  group: deploy-${{ inputs.environment }}
  cancel-in-progress: false
```

- Un seul déploiement à la fois par environnement
- `deploy-dev`, `deploy-homol`, `deploy-prod` sont indépendants

### Par branche + environnement

```yaml
concurrency:
  group: deploy-${{ inputs.environment }}-${{ github.ref }}
  cancel-in-progress: false
```

Isoler par branche ET par environnement.

## Cancel-in-progress : Quand l'utiliser ?

### ✅ `cancel-in-progress: true`

**Cas d'usage :**
- Tests sur PR (nouveau push = anciens tests obsolètes)
- Builds de développement
- Vérifications pré-merge

```yaml
name: Tests PR

on:
  pull_request:
    branches: [develop]

concurrency:
  group: tests-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true  # Annuler les anciens tests

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

### ❌ `cancel-in-progress: false`

**Cas d'usage :**
- Déploiements (ne jamais interrompre)
- Releases (complétude critique)
- Migrations de base de données

```yaml
name: Déploiement Production

on:
  push:
    branches: [main]

concurrency:
  group: deploy-prod
  cancel-in-progress: false  # Ne JAMAIS annuler un déploiement

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Déploiement production"
```

## Patterns courants

### Pattern 1 : Tests sur PR (annulation autorisée)

```yaml
name: Tests

on:
  pull_request:
    types: [opened, synchronize]

concurrency:
  group: tests-${{ github.event.pull_request.head.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

**Comportement :**
- Push 1 sur la PR → Tests démarrent
- Push 2 sur la PR → Tests du Push 1 annulés, Tests du Push 2 démarrent

### Pattern 2 : Déploiements séquentiels (pas d'annulation)

```yaml
name: Déploiement

on:
  push:
    branches: [main]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

**Comportement :**
- Commit 1 → Déploiement démarre
- Commit 2 → Attend la fin du déploiement du Commit 1
- Puis déploie le Commit 2

### Pattern 3 : Multi-environnements isolés

```yaml
name: Déploiement multi-env

on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true

concurrency:
  group: deploy-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Déploiement vers ${{ inputs.environment }}"
```

**Comportement :**
- Déploiement dev et prod peuvent être simultanés
- Deux déploiements dev seront séquentiels

### Pattern 4 : Vérification serveur avec annulation

```yaml
name: Vérification serveur

on:
  pull_request:
    types: [opened, synchronize]
    branches: [develop]

concurrency:
  group: server-check-${{ github.event.pull_request.head.ref }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Tester connexion SFTP
        run: ssh user@server "echo 'Connexion OK'"
```

## Workflows réutilisables et concurrency

### Définir la concurrency dans le workflow réutilisable

```yaml
name: Déploiement réutilisable

on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true

concurrency:
  group: deploy-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Déploiement"
```

### Définir la concurrency dans le consommateur

```yaml
name: Appel déploiement

on:
  push:
    branches: [main]

jobs:
  deploy-prod:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: prod
    concurrency:
      group: deploy-prod-${{ github.ref }}
      cancel-in-progress: false
```

**⚠️ Attention :** Si les deux niveaux définissent une concurrency, **le consommateur l'emporte**.

## Déboguer les problèmes de concurrency

### Workflows en attente

**Symptôme :** Workflow reste en "Pending" longtemps.

**Cause :** Un autre workflow du même groupe est en cours.

**Solution :**
1. Vérifier l'onglet **Actions** pour les runs en cours
2. Vérifier que le groupe de concurrency est correct
3. Considérer `cancel-in-progress: true` si approprié

### Workflows annulés inopinément

**Symptôme :** Workflow annulé sans raison apparente.

**Cause :** `cancel-in-progress: true` et nouveau run dans le même groupe.

**Solution :**
1. Vérifier si `cancel-in-progress` est approprié
2. Affiner le groupe de concurrency (ex: ajouter le numéro de PR)

```yaml
# ❌ Trop large - annule tout
concurrency:
  group: tests
  cancel-in-progress: true

# ✅ Isolé par PR - n'annule que la même PR
concurrency:
  group: tests-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

## Bonnes pratiques

### ✅ À faire

1. **Toujours définir un groupe pour les déploiements**
   ```yaml
   concurrency:
     group: deploy-prod
     cancel-in-progress: false
   ```

2. **Annuler les tests obsolètes sur PR**
   ```yaml
   concurrency:
     group: pr-${{ github.event.pull_request.number }}
     cancel-in-progress: true
   ```

3. **Isoler par environnement**
   ```yaml
   concurrency:
     group: deploy-${{ inputs.environment }}
   ```

4. **Être explicite**
   ```yaml
   # ✅ Clair et intentionnel
   concurrency:
     group: deploy-prod-${{ github.ref }}
     cancel-in-progress: false  # Explicite
   ```

### ❌ À éviter

1. **Groupes trop larges**
   ```yaml
   # ❌ Bloque TOUS les workflows
   concurrency:
     group: default
   ```

2. **Annuler des déploiements**
   ```yaml
   # ❌ Risque d'état incohérent
   concurrency:
     group: deploy-prod
     cancel-in-progress: true
   ```

3. **Pas de concurrency sur les déploiements**
   ```yaml
   # ❌ Risque de déploiements simultanés
   # (pas de concurrency définie)
   ```

4. **Groupes non dynamiques quand nécessaire**
   ```yaml
   # ❌ Ne distingue pas les branches
   concurrency:
     group: tests
   
   # ✅ Isole par branche
   concurrency:
     group: tests-${{ github.ref }}
   ```

## Exemple complet : Système de déploiement

```yaml
name: Déploiement complet

on:
  push:
    branches: [develop, homol, main]

jobs:
  determine-env:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
    steps:
      - id: env
        run: |
          case "${{ github.ref_name }}" in
            develop) echo "environment=dev" >> $GITHUB_OUTPUT ;;
            homol)   echo "environment=homol" >> $GITHUB_OUTPUT ;;
            main)    echo "environment=prod" >> $GITHUB_OUTPUT ;;
          esac
  
  deploy:
    needs: determine-env
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: ${{ needs.determine-env.outputs.environment }}
    concurrency:
      # Un seul déploiement à la fois par environnement
      group: deploy-${{ needs.determine-env.outputs.environment }}
      cancel-in-progress: false
```

## Visualiser la concurrency

Dans l'interface GitHub Actions :
1. Aller dans **Actions** → Workflow
2. Voir les runs **"Queued"** ou **"In progress"**
3. Filtrer par branche pour voir les groupes actifs

## Ressources

- [Workflow syntax - concurrency](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#concurrency)
- [Using concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency)

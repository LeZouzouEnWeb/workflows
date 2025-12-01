# 🔒 Permissions GITHUB_TOKEN

## Qu'est-ce que GITHUB_TOKEN ?

`GITHUB_TOKEN` est un token d'authentification automatiquement créé par GitHub pour chaque workflow. Il permet d'interagir avec l'API GitHub (créer des issues, commenter des PR, etc.) sans nécessiter de Personal Access Token (PAT).

## Permissions par défaut

Depuis septembre 2023, GitHub recommande le mode **restrictif** par défaut :

```yaml
permissions:
  contents: read  # Lecture seule par défaut
```

### Mode permissif (legacy)

```yaml
permissions: write-all  # ⚠️ Non recommandé
```

## Permissions disponibles

| Permission | Scope | Actions autorisées |
|------------|-------|-------------------|
| `actions` | read/write | Gérer les workflows et artifacts |
| `checks` | read/write | Créer/modifier des checks |
| `contents` | read/write | Lire/modifier le code, créer des releases |
| `deployments` | read/write | Gérer les déploiements |
| `issues` | read/write | Créer/modifier des issues |
| `packages` | read/write | Publier des packages |
| `pages` | read/write | Déployer GitHub Pages |
| `pull-requests` | read/write | Créer/modifier des PR et commentaires |
| `repository-projects` | read/write | Gérer les projets du dépôt |
| `security-events` | read/write | Publier des résultats de scan de sécurité |
| `statuses` | read/write | Créer des statuts de commit |

## Configuration des permissions

### Au niveau du workflow

```yaml
name: Mon workflow

on: [push]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  my-job:
    runs-on: ubuntu-latest
    steps:
      - name: Commenter la PR
        run: echo "Commentaire"
```

### Au niveau du job

Les permissions au niveau du job **écrasent** celles du workflow :

```yaml
name: Multi-jobs

permissions:
  contents: read  # Par défaut pour tous les jobs

jobs:
  read-only:
    runs-on: ubuntu-latest
    # Hérite de contents: read
    steps:
      - uses: actions/checkout@v4
      
  write-access:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # Override pour ce job
      pull-requests: write
    steps:
      - name: Créer un tag
        run: git tag v1.0.0
```

## Permissions dans les workflows réutilisables

### Workflow réutilisable

```yaml
name: Workflow réutilisable

on:
  workflow_call:

# ⚠️ Les permissions ici sont IGNORÉES pour le workflow appelant
permissions:
  contents: read
  issues: write

jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - name: Commenter l'issue
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '✅ Traitement terminé'
            });
```

### Workflow consommateur

```yaml
name: Appel du workflow

on: [push]

permissions:
  contents: read
  issues: write      # ✅ Requis pour que le workflow appelé puisse commenter
  pull-requests: write

jobs:
  call-workflow:
    uses: owner/repo/.github/workflows/reusable.yml@v1
```

**🔑 Point important :** Les permissions doivent être définies dans le **workflow consommateur**, pas dans le workflow réutilisable.

## Cas d'usage courants

### Lecture seule (défaut sécurisé)

```yaml
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### Commenter une PR

```yaml
permissions:
  contents: read
  pull-requests: write

jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Tests réussis!'
            });
```

### Créer/modifier des issues

```yaml
permissions:
  contents: read
  issues: write

jobs:
  close-issue:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: 123,
              state: 'closed'
            });
```

### Créer une release

```yaml
permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Créer une release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v1.0.0
          release_name: Release v1.0.0
```

### Publier des résultats de sécurité (CodeQL, etc.)

```yaml
permissions:
  contents: read
  security-events: write

jobs:
  codeql:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
      - uses: github/codeql-action/analyze@v3
```

### Déployer sur GitHub Pages

```yaml
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
      - uses: actions/deploy-pages@v4
```

## Principe du moindre privilège

Toujours définir les permissions **minimales** nécessaires :

### ❌ Trop permissif

```yaml
permissions: write-all  # Donne tous les droits
```

### ✅ Juste ce qu'il faut

```yaml
permissions:
  contents: read         # Lire le code
  pull-requests: write   # Commenter les PR
  issues: write          # Modifier les issues
```

## Déboguer les problèmes de permissions

### Erreur courante : "Resource not accessible"

```
Error: Resource not accessible by integration
```

**Cause :** Permission manquante dans le workflow consommateur.

**Solution :**

```yaml
# Ajouter la permission manquante
permissions:
  contents: read
  issues: write       # ← Permission manquante
```

### Vérifier les permissions actives

```yaml
steps:
  - name: Afficher les permissions
    run: |
      echo "Token permissions:"
      curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
        https://api.github.com/repos/${{ github.repository }}
```

## Permissions et forked repositories

⚠️ **Attention :** Les PR depuis des forks ont des permissions **très limitées** par sécurité :

- `GITHUB_TOKEN` en **lecture seule** sur le dépôt cible
- Pas d'accès aux secrets du dépôt cible
- Recommandation : utiliser `pull_request_target` avec précaution

```yaml
on:
  pull_request_target:  # Attention : contexte du dépôt cible
    branches: [main]

permissions:
  contents: read
  pull-requests: write  # Permet de commenter même depuis un fork

jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - name: Commenter la PR
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '👋 Merci pour votre contribution!'
            });
```

## Bonnes pratiques

### ✅ À faire

1. **Définir explicitement les permissions**
   ```yaml
   permissions:
     contents: read
     pull-requests: write
   ```

2. **Permissions au niveau job si différentes**
   ```yaml
   jobs:
     read-job:
       permissions:
         contents: read
     write-job:
       permissions:
         contents: write
   ```

3. **Documenter pourquoi chaque permission est nécessaire**
   ```yaml
   permissions:
     contents: read        # Checkout du code
     pull-requests: write  # Commenter les résultats des tests
     issues: write         # Fermer l'issue liée après merge
   ```

4. **Tester avec permissions minimales**
   - Commencer avec `contents: read`
   - Ajouter les permissions au fur et à mesure des besoins

### ❌ À éviter

1. **`write-all` en production**
   ```yaml
   permissions: write-all  # ❌ Trop permissif
   ```

2. **Permissions non nécessaires**
   ```yaml
   permissions:
     contents: write      # ❌ Si on ne modifie pas le dépôt
     packages: write      # ❌ Si on ne publie pas de packages
   ```

3. **Oublier les permissions dans le consommateur**
   ```yaml
   # Workflow réutilisable avec actions/github-script
   # ❌ Sans permissions dans le consommateur, ça échouera
   ```

## Exemple complet

```yaml
name: PR Workflow complet

on:
  pull_request:
    types: [opened, synchronize]
    branches: [develop]

permissions:
  contents: read         # Checkout du code
  pull-requests: write   # Commenter la PR
  issues: write          # Lier/fermer les issues
  checks: write          # Créer des checks

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Vérifier tickets liés
        uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            const hasTicket = /#\d+/.test(pr.title) || /#\d+/.test(pr.body);
            
            if (!hasTicket) {
              await github.rest.issues.createComment({
                issue_number: pr.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: '⚠️ Aucun ticket détecté dans cette PR'
              });
              core.setFailed('Ticket manquant');
            }
```

## Ressources

- [Permissions for the GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [Workflow syntax - permissions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions)
- [Security hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

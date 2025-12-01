# 📞 Workflow Call - Workflows réutilisables

## Qu'est-ce qu'un workflow réutilisable ?

Un workflow réutilisable est un workflow GitHub Actions qui peut être appelé depuis d'autres workflows, comme une fonction réutilisable dans le code. Cela permet de :
- **Centraliser** la logique dans un dépôt unique
- **Standardiser** les processus à travers plusieurs projets
- **Maintenir** facilement : une modification se propage partout
- **Réduire** la duplication de code

## Syntaxe de base

### Workflow réutilisable (source)

```yaml
name: Mon workflow réutilisable

on:
  workflow_call:
    inputs:
      environment:
        description: 'Environnement cible (dev/homol/prod)'
        required: true
        type: string
      deploy:
        description: 'Effectuer le déploiement ?'
        required: false
        type: boolean
        default: true
    secrets:
      SSH_KEY:
        description: 'Clé SSH pour le déploiement'
        required: true
    outputs:
      deployment_url:
        description: 'URL du déploiement'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - name: Déploiement
        id: deploy
        run: |
          echo "Déploiement vers ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

### Workflow consommateur (appelant)

```yaml
name: Déploiement Dev

on:
  push:
    branches: [develop]

jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: dev
      deploy: true
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
```

## Déclencheurs supportés

Un workflow réutilisable **DOIT** utiliser `workflow_call` comme déclencheur. Vous pouvez combiner avec d'autres déclencheurs :

```yaml
on:
  workflow_call:
    # Configuration pour les appels externes
  push:
    branches: [main]
    # Permet aussi d'exécuter directement le workflow
  workflow_dispatch:
    # Permet l'exécution manuelle
```

⚠️ **Attention** : Si vous voulez UNIQUEMENT que le workflow soit appelable (pas d'exécution directe), utilisez **SEULEMENT** `workflow_call`.

## Références au workflow réutilisable

### Format de référence

```yaml
uses: {owner}/{repo}/.github/workflows/{filename}@{ref}
```

- `owner/repo` : dépôt contenant le workflow
- `filename` : nom du fichier YAML
- `ref` : branche, tag ou SHA

### Exemples de références

```yaml
# Référence par tag (recommandé pour la production)
uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@v1

# Référence par branche (pour le développement)
uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@main

# Référence par SHA (maximum de stabilité)
uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@a1b2c3d
```

### Bonnes pratiques de référencement

1. **Production** : utilisez des tags sémantiques (`@v1`, `@v1.2.3`)
2. **Développement** : utilisez `@main` ou `@develop`
3. **Audit/Compliance** : utilisez le SHA complet
4. **Mises à jour** : 
   - Tags majeurs (`v1`) : auto-update vers v1.x.x
   - Tags complets (`v1.2.3`) : version fixe

## Inputs et types supportés

### Types disponibles

```yaml
on:
  workflow_call:
    inputs:
      # Chaîne de caractères
      environment:
        type: string
        required: true
        
      # Booléen
      dry_run:
        type: boolean
        default: false
        
      # Nombre
      timeout:
        type: number
        default: 30
```

### Valeurs par défaut

```yaml
inputs:
  log_level:
    type: string
    default: 'info'
  retries:
    type: number
    default: 3
```

### Utilisation dans le workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Afficher l'environnement
        run: echo "Déploiement vers ${{ inputs.environment }}"
        
      - name: Mode dry-run
        if: ${{ inputs.dry_run }}
        run: echo "Mode simulation activé"
```

## Secrets

### Deux approches possibles

#### 1. Héritage complet (recommandé pour simplicité)

```yaml
# Consommateur
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets: inherit
```

Tous les secrets disponibles dans le workflow appelant sont transmis.

#### 2. Mapping explicite (recommandé pour sécurité)

```yaml
# Workflow réutilisable
on:
  workflow_call:
    secrets:
      SSH_KEY:
        required: true
      DB_PASSWORD:
        required: false

# Consommateur
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets:
      SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
      DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
```

### Quand utiliser chaque approche ?

- **`inherit`** : workflows internes, même organisation, confiance totale
- **Mapping explicite** : workflows publics, contrôle fin, sécurité renforcée

## Outputs

Les outputs permettent de retourner des valeurs au workflow appelant.

### Définition dans le workflow réutilisable

```yaml
on:
  workflow_call:
    outputs:
      deployment_url:
        description: "URL du déploiement"
        value: ${{ jobs.deploy.outputs.url }}
      status:
        description: "Statut du déploiement"
        value: ${{ jobs.deploy.outputs.status }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
      status: ${{ steps.deploy.outputs.status }}
    steps:
      - name: Déployer
        id: deploy
        run: |
          echo "url=https://example.com" >> $GITHUB_OUTPUT
          echo "status=success" >> $GITHUB_OUTPUT
```

### Utilisation dans le workflow consommateur

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    
  notify:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Envoyer notification
        run: |
          echo "Déployé sur ${{ needs.deploy.outputs.deployment_url }}"
          echo "Statut: ${{ needs.deploy.outputs.status }}"
```

## Limitations importantes

### Ce qui N'EST PAS supporté

❌ **Variables d'environnement au niveau workflow** : `env:` au niveau racine n'est pas transmis aux workflows réutilisables

❌ **Contexte `github.env`** : pas accessible dans le workflow réutilisable

❌ **Stratégies de matrice au niveau du job réutilisable** : la matrice doit être dans le consommateur

### Solutions de contournement

#### Transmettre des "env" comme inputs

```yaml
# Au lieu de
env:
  MY_VAR: value

# Utiliser
inputs:
  my_var:
    type: string
    default: 'value'
```

#### Matrice dans le consommateur

```yaml
# Consommateur
jobs:
  deploy:
    strategy:
      matrix:
        environment: [dev, homol, prod]
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: ${{ matrix.environment }}
```

## Bonnes pratiques

### ✅ À faire

1. **Documenter** : ajouter des descriptions claires pour inputs/secrets/outputs
2. **Versionner** : utiliser des tags sémantiques (`v1.0.0`)
3. **Valider** : vérifier la présence des inputs/secrets requis
4. **Tester** : créer des workflows de test avant le déploiement
5. **Simplifier** : un workflow = une responsabilité claire

### ❌ À éviter

1. Trop de paramètres : privilégier des conventions
2. Secrets non documentés : toujours les déclarer explicitement
3. Dépendances cachées : documenter les prérequis
4. Workflows trop génériques : trouver le bon équilibre
5. Pas de validation : toujours vérifier les inputs critiques

## Exemple complet

Voir [reusable-template.yml](../templates/reusable-template.yml) pour un template complet et commenté.

## Ressources

- [Documentation officielle - Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Guide de migration](../guides/01-converting-workflow.md)
- [Calling workflows](../guides/02-calling-workflow.md)

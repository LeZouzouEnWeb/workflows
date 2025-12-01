# 🔄 Convertir un workflow existant en réutilisable

Guide étape par étape pour transformer un workflow standard en workflow réutilisable.

## Processus de conversion

### Étape 1 : Analyser le workflow existant

Avant de commencer, identifiez :
- Les **déclencheurs** actuels (`on:`)
- Les **variables d'environnement** utilisées
- Les **secrets** requis
- Les **valeurs** qui pourraient être paramétrées
- Les **outputs** potentiellement utiles

### Étape 2 : Sauvegarder l'original

```bash
cp .github/workflows/deploy.yml .github/workflows/deploy.yml.backup
```

### Étape 3 : Modifier le déclencheur

#### Avant

```yaml
name: Déploiement Dev

on:
  push:
    branches: [develop]
  workflow_dispatch:
```

#### Après

```yaml
name: Déploiement Dev

# ──────────────────────────────────────────────────────────────
# Ancien déclencheur (commenté pour référence)
# ──────────────────────────────────────────────────────────────
# on:
#   push:
#     branches: [develop]
#   workflow_dispatch:

# ──────────────────────────────────────────────────────────────
# Nouveau : workflow réutilisable
# ──────────────────────────────────────────────────────────────
on:
  workflow_call:
```

💡 **Astuce** : Conserver les anciens déclencheurs en commentaire aide à créer les workflows consommateurs.

### Étape 4 : Identifier les variables à paramétrer

#### Avant

```yaml
env:
  REMOTE_ENV: dev
  REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
  ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Déployer
        run: echo "Déploiement vers dev"
```

#### Après

```yaml
on:
  workflow_call:
    inputs:
      remote_env:
        description: 'Environnement cible (dev/homol/prod)'
        type: string
        required: true
      ADRESSE_GLOBAL:
        description: 'Domaine principal (ex: corbisier.fr)'
        type: string
        required: true
      ADRESSE_LOCAL:
        description: 'Sous-dossier virtualhost (ex: web-git)'
        type: string
        required: true
    secrets:
      REMOTE_CHEMIN:
        description: 'Chemin racine sur le serveur'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Déployer
        run: echo "Déploiement vers ${{ inputs.remote_env }}"
        env:
          REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
          ADRESSE_GLOBAL: ${{ inputs.ADRESSE_GLOBAL }}
          ADRESSE_LOCAL: ${{ inputs.ADRESSE_LOCAL }}
```

### Étape 5 : Déclarer les secrets explicitement

#### Avant (secrets utilisés implicitement)

```yaml
jobs:
  deploy:
    steps:
      - uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

#### Après (secrets déclarés)

```yaml
on:
  workflow_call:
    secrets:
      SSH_PRIVATE_KEY:
        description: 'Clé privée SSH au format PEM'
        required: true
      SFTP_HOST:
        description: 'Hôte SFTP (ex: access-ssh.ionos.fr)'
        required: true
      SFTP_USER:
        description: 'Utilisateur SFTP'
        required: true
      SFTP_PORT:
        description: 'Port SSH (défaut: 22)'
        required: false

jobs:
  deploy:
    steps:
      - uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Étape 6 : Ajouter des outputs si nécessaire

#### Si le workflow produit des résultats utiles

```yaml
on:
  workflow_call:
    outputs:
      deployment_url:
        description: 'URL du déploiement'
        value: ${{ jobs.deploy.outputs.url }}
      deployment_time:
        description: 'Timestamp du déploiement'
        value: ${{ jobs.deploy.outputs.timestamp }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
      timestamp: ${{ steps.deploy.outputs.timestamp }}
    steps:
      - name: Déployer
        id: deploy
        run: |
          URL="https://${{ inputs.remote_env }}.example.com"
          TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
          echo "url=$URL" >> $GITHUB_OUTPUT
          echo "timestamp=$TIMESTAMP" >> $GITHUB_OUTPUT
```

### Étape 7 : Ajuster les permissions

Les permissions définies dans le workflow réutilisable **ne s'appliquent pas** au workflow appelant. Documentez-les pour que les consommateurs sachent quoi définir.

```yaml
# ──────────────────────────────────────────────────────────────
# PERMISSIONS REQUISES DANS LE WORKFLOW CONSOMMATEUR :
# ──────────────────────────────────────────────────────────────
# permissions:
#   contents: read
#   deployments: write
#   statuses: write

on:
  workflow_call:
    # ...

permissions:
  contents: read  # Utilisé uniquement si exécuté directement
```

### Étape 8 : Valider les inputs

Ajouter une étape de validation en début de workflow :

```yaml
jobs:
  validate:
    name: Validation des paramètres
    runs-on: ubuntu-latest
    steps:
      - name: Valider l'environnement
        run: |
          ENV="${{ inputs.remote_env }}"
          case "$ENV" in
            dev|homol|prod)
              echo "✅ Environnement valide: $ENV"
              ;;
            *)
              echo "::error title=Environnement invalide::Attendu: dev/homol/prod, reçu: $ENV"
              exit 1
              ;;
          esac
          
      - name: Valider les secrets
        run: |
          missing=0
          if [ -z "${{ secrets.SSH_PRIVATE_KEY }}" ]; then
            echo "::error::SSH_PRIVATE_KEY non défini"
            missing=1
          fi
          [ "$missing" -eq 0 ] || exit 1
```

## Exemple complet : Avant/Après

### AVANT : Workflow standard

```yaml
name: 🚀 Déploiement Dev

on:
  pull_request:
    branches: [develop]
    types: [closed]

permissions:
  contents: read

env:
  REMOTE_ENV: dev
  REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
  ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
  ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
  SFTP_HOST: ${{ secrets.SFTP_HOST }}
  SFTP_USER: ${{ secrets.SFTP_USER }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

jobs:
  deploy:
    if: ${{ github.event.pull_request.merged == true }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Préparer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ env.SSH_PRIVATE_KEY }}
      
      - name: Déployer
        run: |
          REMOTE_PATH="/${REMOTE_CHEMIN}/${ADRESSE_GLOBAL}/${ADRESSE_LOCAL}/${REMOTE_ENV}"
          rsync -az --delete ./ "$SFTP_USER@$SFTP_HOST:$REMOTE_PATH/"
```

### APRÈS : Workflow réutilisable

```yaml
name: 🚀 Déploiement réutilisable

# ──────────────────────────────────────────────────────────────
# Ancien déclencheur (pour référence lors de la création du consommateur)
# ──────────────────────────────────────────────────────────────
# on:
#   pull_request:
#     branches: [develop]
#     types: [closed]

on:
  workflow_call:
    inputs:
      remote_env:
        description: 'Environnement (dev/homol/prod)'
        type: string
        required: true
      ADRESSE_GLOBAL:
        description: 'Domaine principal'
        type: string
        required: true
      ADRESSE_LOCAL:
        description: 'Sous-dossier virtualhost'
        type: string
        required: true
    secrets:
      REMOTE_CHEMIN:
        description: 'Chemin racine serveur'
        required: true
      SFTP_HOST:
        description: 'Hôte SFTP'
        required: true
      SFTP_USER:
        description: 'Utilisateur SFTP'
        required: true
      SFTP_PORT:
        description: 'Port SSH'
        required: false
      SSH_PRIVATE_KEY:
        description: 'Clé SSH privée'
        required: true
    outputs:
      deployment_path:
        description: 'Chemin de déploiement'
        value: ${{ jobs.deploy.outputs.path }}

# ──────────────────────────────────────────────────────────────
# PERMISSIONS REQUISES DANS LE WORKFLOW CONSOMMATEUR :
# ──────────────────────────────────────────────────────────────
# permissions:
#   contents: read

jobs:
  validate:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Valider environnement
        run: |
          case "${{ inputs.remote_env }}" in
            dev|homol|prod) echo "✅ Valide" ;;
            *) echo "::error::Environnement invalide"; exit 1 ;;
          esac

  deploy:
    name: Déploiement vers ${{ inputs.remote_env }}
    needs: validate
    runs-on: ubuntu-latest
    outputs:
      path: ${{ steps.deploy.outputs.path }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Construire le chemin
        id: path
        run: |
          PATH_="/${{ secrets.REMOTE_CHEMIN }}/${{ inputs.ADRESSE_GLOBAL }}/${{ inputs.ADRESSE_LOCAL }}/${{ inputs.remote_env }}"
          PATH_=$(echo "$PATH_" | sed 's#//*#/#g')
          echo "path=$PATH_" >> $GITHUB_OUTPUT
          echo "::notice::Déploiement vers: $PATH_"
      
      - name: Préparer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      
      - name: Ajouter known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p "${{ secrets.SFTP_PORT || '22' }}" "${{ secrets.SFTP_HOST }}" >> ~/.ssh/known_hosts
      
      - name: Déployer
        id: deploy
        run: |
          rsync -az --delete \
            -e "ssh -p ${{ secrets.SFTP_PORT || '22' }}" \
            ./ "${{ secrets.SFTP_USER }}@${{ secrets.SFTP_HOST }}:${{ steps.path.outputs.path }}/"
          
          echo "✅ Déploiement terminé"
```

## Créer le workflow consommateur

Maintenant que le workflow est réutilisable, créez le workflow consommateur :

```yaml
name: 🚀 Déploiement Dev

on:
  pull_request:
    branches: [develop]
    types: [closed]

permissions:
  contents: read

jobs:
  deploy:
    if: ${{ github.event.pull_request.merged == true }}
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/deploy.yml@v1
    with:
      remote_env: dev
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      SFTP_HOST: ${{ secrets.SFTP_HOST }}
      SFTP_USER: ${{ secrets.SFTP_USER }}
      SFTP_PORT: ${{ secrets.SFTP_PORT }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

## Checklist de conversion

- [ ] Sauvegarder le workflow original
- [ ] Commenter les anciens déclencheurs
- [ ] Ajouter `workflow_call` comme déclencheur
- [ ] Identifier les variables à paramétrer → créer des `inputs`
- [ ] Identifier les secrets requis → déclarer dans `secrets`
- [ ] Remplacer `env:` au niveau racine par `inputs:` / `secrets:`
- [ ] Ajouter des `outputs` si pertinent
- [ ] Ajouter une validation des inputs
- [ ] Documenter les permissions requises
- [ ] Tester avec un workflow consommateur
- [ ] Taguer la version (`v1`)
- [ ] Mettre à jour la documentation

## Erreurs courantes

### ❌ Erreur 1 : Oublier de déclarer un secret

```yaml
# ❌ MAUVAIS
on:
  workflow_call:
    # secrets non déclarés

jobs:
  deploy:
    steps:
      - run: echo ${{ secrets.SSH_KEY }}  # Ne fonctionnera pas
```

```yaml
# ✅ BON
on:
  workflow_call:
    secrets:
      SSH_KEY:
        required: true

jobs:
  deploy:
    steps:
      - run: echo ${{ secrets.SSH_KEY }}  # Fonctionne
```

### ❌ Erreur 2 : Variables d'environnement au niveau racine

```yaml
# ❌ MAUVAIS - env: racine n'est pas transmis
env:
  MY_VAR: value

on:
  workflow_call:
```

```yaml
# ✅ BON - utiliser inputs
on:
  workflow_call:
    inputs:
      my_var:
        type: string
        default: 'value'
```

### ❌ Erreur 3 : Permissions non documentées

```yaml
# ❌ MAUVAIS - pas de documentation
on:
  workflow_call:

permissions:
  contents: write  # Ignoré par le consommateur
```

```yaml
# ✅ BON - documenter pour le consommateur
# ──────────────────────────────────────────────────────────────
# PERMISSIONS REQUISES DANS LE WORKFLOW CONSOMMATEUR :
# ──────────────────────────────────────────────────────────────
# permissions:
#   contents: write

on:
  workflow_call:
```

## Tester la conversion

```bash
# 1. Pousser le workflow réutilisable
git add .github/workflows/deploy-reusable.yml
git commit -m "feat: convert deploy workflow to reusable"
git push origin main

# 2. Créer un tag
git tag v1
git push origin v1

# 3. Créer le workflow consommateur dans un dépôt de test
# Créer .github/workflows/deploy-consumer.yml

# 4. Créer une PR de test pour vérifier
```

## Bonnes pratiques

### ✅ À faire

1. **Commenter les anciens déclencheurs** pour référence
2. **Valider les inputs** en début de workflow
3. **Documenter les permissions** requises
4. **Tester avant de taguer** la version
5. **Versionner sémantiquement** (`v1.0.0`)

### ❌ À éviter

1. **Supprimer l'historique** des anciens déclencheurs
2. **Oublier la validation** des inputs critiques
3. **Taguer sans tester** le workflow
4. **Casser la rétrocompatibilité** sans prévenir
5. **Ne pas documenter** les changements

## Ressources

- [Template de workflow réutilisable](../templates/reusable-template.yml)
- [Template de workflow consommateur](../templates/consumer-template.yml)
- [Fondamentaux - workflow_call](../fondamentaux/01-workflow-call.md)

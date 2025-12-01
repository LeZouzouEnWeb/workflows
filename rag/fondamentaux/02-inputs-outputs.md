# 📥📤 Inputs & Outputs

## Inputs - Paramètres d'entrée

Les inputs permettent de passer des paramètres à un workflow réutilisable.

### Types d'inputs supportés

```yaml
on:
  workflow_call:
    inputs:
      # Chaîne de caractères
      environment:
        description: 'Environnement cible'
        type: string
        required: true
        
      # Booléen
      dry_run:
        description: 'Mode simulation sans déploiement réel'
        type: boolean
        required: false
        default: false
        
      # Nombre (entier ou décimal)
      timeout:
        description: 'Timeout en secondes'
        type: number
        required: false
        default: 300
```

### Propriétés des inputs

| Propriété | Obligatoire | Description |
|-----------|-------------|-------------|
| `type` | ✅ Oui | Type de donnée (`string`, `boolean`, `number`) |
| `description` | ❌ Non | Description affichée dans l'UI GitHub |
| `required` | ❌ Non | Si `true`, l'input doit être fourni (défaut: `false`) |
| `default` | ❌ Non | Valeur par défaut si non fourni |

### Utilisation dans le workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Afficher l'environnement
        run: echo "Déploiement vers ${{ inputs.environment }}"
        
      - name: Vérifier le mode dry-run
        if: ${{ inputs.dry_run }}
        run: echo "Mode simulation activé"
        
      - name: Configurer le timeout
        run: echo "Timeout configuré à ${{ inputs.timeout }}s"
```

### Validation des inputs

Il est recommandé de valider les inputs critiques en début de workflow :

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Valider l'environnement
        run: |
          ENV="${{ inputs.environment }}"
          case "$ENV" in
            dev|homol|prod)
              echo "✅ Environnement valide: $ENV"
              ;;
            *)
              echo "::error title=Environnement invalide::Attendu: dev/homol/prod, reçu: $ENV"
              exit 1
              ;;
          esac
```

## Outputs - Valeurs de retour

Les outputs permettent au workflow réutilisable de retourner des valeurs au workflow appelant.

### Définir des outputs

```yaml
on:
  workflow_call:
    outputs:
      deployment_url:
        description: "URL du déploiement effectué"
        value: ${{ jobs.deploy.outputs.url }}
      deployment_time:
        description: "Timestamp du déploiement"
        value: ${{ jobs.deploy.outputs.timestamp }}
      status:
        description: "Statut du déploiement (success/failure)"
        value: ${{ jobs.deploy.outputs.status }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
      timestamp: ${{ steps.deploy.outputs.timestamp }}
      status: ${{ steps.deploy.outputs.status }}
    steps:
      - name: Déployer
        id: deploy
        run: |
          # Effectuer le déploiement
          URL="https://${{ inputs.environment }}.example.com"
          TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
          
          # Définir les outputs du step
          echo "url=$URL" >> $GITHUB_OUTPUT
          echo "timestamp=$TIMESTAMP" >> $GITHUB_OUTPUT
          echo "status=success" >> $GITHUB_OUTPUT
```

### Utiliser les outputs

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      environment: dev
      
  notify:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Notifier le déploiement
        run: |
          echo "Déployé sur: ${{ needs.deploy.outputs.deployment_url }}"
          echo "À: ${{ needs.deploy.outputs.deployment_time }}"
          echo "Statut: ${{ needs.deploy.outputs.status }}"
          
      - name: Créer un tag
        if: ${{ needs.deploy.outputs.status == 'success' }}
        run: |
          git tag "deployed-${{ needs.deploy.outputs.deployment_time }}"
```

## Chaîner plusieurs workflows réutilisables

Vous pouvez chaîner des workflows et transmettre les outputs :

```yaml
jobs:
  build:
    uses: owner/repo/.github/workflows/build.yml@v1
    with:
      version: 1.2.3
      
  deploy:
    needs: build
    uses: owner/repo/.github/workflows/deploy.yml@v1
    with:
      artifact_url: ${{ needs.build.outputs.artifact_url }}
      environment: prod
      
  notify:
    needs: [build, deploy]
    uses: owner/repo/.github/workflows/notify.yml@v1
    with:
      build_version: ${{ needs.build.outputs.version }}
      deployment_url: ${{ needs.deploy.outputs.deployment_url }}
```

## Patterns avancés

### Input avec valeurs contraintes

```yaml
jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Valider l'environnement
        run: |
          ENV="${{ inputs.environment }}"
          ALLOWED="dev homol prod"
          
          if ! echo "$ALLOWED" | grep -qw "$ENV"; then
            echo "::error::Environnement '$ENV' non autorisé. Valeurs acceptées: $ALLOWED"
            exit 1
          fi
          
      - name: Déployer
        run: echo "Déploiement vers ${{ inputs.environment }}"
```

### Input conditionnel

```yaml
on:
  workflow_call:
    inputs:
      enable_notifications:
        type: boolean
        default: false
      slack_channel:
        type: string
        required: false  # requis seulement si enable_notifications=true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Valider la configuration
        if: ${{ inputs.enable_notifications }}
        run: |
          if [ -z "${{ inputs.slack_channel }}" ]; then
            echo "::error::slack_channel requis quand enable_notifications=true"
            exit 1
          fi
```

### Outputs multiples depuis plusieurs jobs

```yaml
on:
  workflow_call:
    outputs:
      build_artifact:
        value: ${{ jobs.build.outputs.artifact }}
      test_report:
        value: ${{ jobs.test.outputs.report }}
      deploy_url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  build:
    outputs:
      artifact: ${{ steps.build.outputs.artifact }}
    steps:
      - id: build
        run: echo "artifact=app-v1.2.3.tar.gz" >> $GITHUB_OUTPUT
        
  test:
    needs: build
    outputs:
      report: ${{ steps.test.outputs.report }}
    steps:
      - id: test
        run: echo "report=tests-passed-100%" >> $GITHUB_OUTPUT
        
  deploy:
    needs: [build, test]
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - id: deploy
        run: echo "url=https://app.example.com" >> $GITHUB_OUTPUT
```

## Transmettre des objets complexes

GitHub Actions ne supporte que des types simples. Pour des données complexes :

### Option 1 : JSON en string

```yaml
# Workflow réutilisable
steps:
  - id: config
    run: |
      CONFIG='{"host":"example.com","port":443,"ssl":true}'
      echo "config=$CONFIG" >> $GITHUB_OUTPUT

# Workflow consommateur
steps:
  - name: Parser la config
    run: |
      CONFIG='${{ needs.deploy.outputs.config }}'
      HOST=$(echo "$CONFIG" | jq -r '.host')
      PORT=$(echo "$CONFIG" | jq -r '.port')
      echo "Connexion à $HOST:$PORT"
```

### Option 2 : Multiples inputs

```yaml
on:
  workflow_call:
    inputs:
      sftp_host:
        type: string
        required: true
      sftp_port:
        type: number
        default: 22
      sftp_user:
        type: string
        required: true
```

## Bonnes pratiques

### ✅ À faire

1. **Nommer clairement** : `deployment_url` plutôt que `url`
2. **Documenter** : toujours ajouter une description
3. **Valider tôt** : vérifier les inputs en début de workflow
4. **Valeurs par défaut sensées** : éviter les surprises
5. **Outputs explicites** : retourner des valeurs utiles pour le chaînage

### ❌ À éviter

1. Trop d'inputs : privilégier les conventions
2. Inputs sans description : rendre le workflow documenté
3. Pas de validation : toujours vérifier les valeurs critiques
4. Outputs non documentés : clarifier leur utilité
5. Dépendances implicites : documenter les prérequis

## Exemples pratiques

### Workflow de déploiement paramétrable

```yaml
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
      version:
        type: string
        required: true
      dry_run:
        type: boolean
        default: false
    secrets:
      SSH_KEY:
        required: true
    outputs:
      deployment_url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - name: Valider inputs
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Dry-run: ${{ inputs.dry_run }}"
          
      - name: Déployer
        id: deploy
        run: |
          if [ "${{ inputs.dry_run }}" = "true" ]; then
            echo "Mode simulation - pas de déploiement réel"
          else
            echo "Déploiement de la version ${{ inputs.version }}"
          fi
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

## Ressources

- [Documentation officielle - Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Reusable workflows inputs](https://docs.github.com/en/actions/using-workflows/reusing-workflows#using-inputs-and-secrets-in-a-reusable-workflow)

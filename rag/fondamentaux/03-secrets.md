# 🔐 Gestion des Secrets

## Qu'est-ce qu'un secret ?

Les secrets sont des informations sensibles (clés API, mots de passe, tokens) stockées de manière sécurisée dans GitHub et accessibles dans les workflows.

## Niveaux de secrets

### 1. Secrets d'Organisation
- Partagés entre tous les dépôts de l'organisation
- Configurables pour tous les dépôts ou seulement certains (selected repositories)
- **Recommandé pour** : credentials infrastructure, services partagés

### 2. Secrets de Dépôt
- Spécifiques à un seul dépôt
- **Recommandé pour** : credentials spécifiques au projet

### 3. Secrets d'Environnement
- Associés à un environnement GitHub (dev/staging/prod)
- Protection supplémentaire avec approbations requises
- **Recommandé pour** : credentials de production sensibles

## Utilisation dans les workflows réutilisables

### Approche 1 : Héritage complet (`inherit`)

Le workflow réutilisable hérite de **tous** les secrets disponibles dans le workflow appelant.

#### Workflow réutilisable

```yaml
name: Déploiement réutilisable

on:
  workflow_call:
    # Pas besoin de déclarer les secrets

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Utiliser les secrets
        run: |
          echo "Host: ${{ secrets.SFTP_HOST }}"
          # Tous les secrets du workflow appelant sont disponibles
```

#### Workflow consommateur

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets: inherit  # Transmet tous les secrets
```

**✅ Avantages :**
- Simple et rapide à mettre en place
- Moins de code répétitif
- Idéal pour workflows internes

**❌ Inconvénients :**
- Moins de contrôle sur ce qui est transmis
- Peut exposer des secrets non nécessaires
- Documentation implicite

### Approche 2 : Mapping explicite

Le workflow réutilisable déclare explicitement les secrets requis.

#### Workflow réutilisable

```yaml
name: Déploiement réutilisable

on:
  workflow_call:
    secrets:
      SFTP_HOST:
        description: 'Serveur SFTP pour le déploiement'
        required: true
      SFTP_USER:
        description: 'Utilisateur SFTP'
        required: true
      SSH_PRIVATE_KEY:
        description: 'Clé SSH privée'
        required: true
      SFTP_PORT:
        description: 'Port SFTP (optionnel)'
        required: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Connexion SFTP
        run: |
          echo "Connexion à ${{ secrets.SFTP_HOST }}"
          echo "Utilisateur: ${{ secrets.SFTP_USER }}"
          echo "Port: ${{ secrets.SFTP_PORT || '22' }}"
```

#### Workflow consommateur

```yaml
jobs:
  deploy:
    uses: owner/repo/.github/workflows/deploy.yml@v1
    secrets:
      SFTP_HOST: ${{ secrets.IONOS_SFTP_HOST }}
      SFTP_USER: ${{ secrets.IONOS_SFTP_USER }}
      SSH_PRIVATE_KEY: ${{ secrets.IONOS_SSH_KEY }}
      SFTP_PORT: ${{ secrets.IONOS_SFTP_PORT }}
```

**✅ Avantages :**
- Contrôle précis des secrets transmis
- Documentation explicite
- Meilleure sécurité (principe du moindre privilège)
- Permet le remapping de noms

**❌ Inconvénients :**
- Plus verbeux
- Duplication si beaucoup de secrets

## Configuration des secrets

### Secrets d'Organisation

1. **Settings** (organisation) → **Secrets and variables** → **Actions**
2. **New organization secret**
3. Choisir les dépôts autorisés :
   - **All repositories** : tous les dépôts peuvent utiliser ce secret
   - **Selected repositories** : seulement certains dépôts (recommandé)

```yaml
# Exemple de secrets d'Organisation recommandés
REMOTE_CHEMIN       # Chemin racine sur le serveur
SFTP_HOST           # Hôte SFTP/SSH
SFTP_PORT           # Port SSH (22)
SFTP_USER           # Utilisateur SSH
SSH_PRIVATE_KEY     # Clé privée SSH
SSH_KNOWN_HOSTS     # Fingerprints des hôtes connus
```

### Secrets de Dépôt

1. **Settings** (dépôt) → **Secrets and variables** → **Actions**
2. **Repository secrets** → **New repository secret**

```yaml
# Exemple de secrets de Dépôt
GITHUB_TOKEN              # Automatiquement fourni par GitHub
SLACK_WEBHOOK_URL         # Webhook spécifique au projet
DATABASE_URL              # URL de base de données du projet
```

### Secrets d'Environnement

1. **Settings** (dépôt) → **Environments** → Créer/sélectionner un environnement
2. **Environment secrets** → **Add secret**

```yaml
# Exemple de secrets d'Environnement
PROD_DATABASE_URL         # URL de la base prod
PROD_API_KEY              # Clé API production
```

## Bonnes pratiques

### ✅ À faire

1. **Principe du moindre privilège**
   ```yaml
   # Déclarer explicitement ce qui est nécessaire
   secrets:
     DEPLOY_KEY:
       required: true
   ```

2. **Documenter les secrets**
   ```yaml
   secrets:
     SSH_KEY:
       description: 'Clé SSH privée au format PEM pour connexion SFTP'
       required: true
   ```

3. **Validation de présence**
   ```yaml
   steps:
     - name: Vérifier secrets requis
       run: |
         if [ -z "${{ secrets.SSH_KEY }}" ]; then
           echo "::error::SSH_KEY non défini"
           exit 1
         fi
   ```

4. **Rotation régulière**
   - Changer les secrets périodiquement
   - Utiliser des clés SSH avec expiration

5. **Secrets d'Organisation pour infrastructure**
   - Credentials serveurs → Organisation
   - Credentials projet → Dépôt
   - Credentials prod sensibles → Environnement

### ❌ À éviter

1. **Secrets en clair dans les logs**
   ```yaml
   # ❌ MAUVAIS
   - run: echo "Password: ${{ secrets.PASSWORD }}"
   
   # ✅ BON
   - run: echo "Connexion avec credentials"
   ```

2. **Secrets dans le code**
   ```yaml
   # ❌ MAUVAIS
   env:
     API_KEY: "sk_live_123456789"
   
   # ✅ BON
   env:
     API_KEY: ${{ secrets.API_KEY }}
   ```

3. **Trop de secrets héritables**
   ```yaml
   # ❌ Éviter si possible pour workflows publics
   secrets: inherit
   
   # ✅ Préférer le mapping explicite
   secrets:
     DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
   ```

4. **Pas de validation**
   ```yaml
   # ❌ MAUVAIS - pas de vérification
   - run: ssh ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}
   
   # ✅ BON - vérification préalable
   - name: Valider secrets
     run: |
       for secret in SSH_USER SSH_HOST SSH_KEY; do
         if [ -z "${!secret}" ]; then
           echo "::error::$secret manquant"
           exit 1
         fi
       done
   ```

## Patterns avancés

### Secret conditionnel

```yaml
on:
  workflow_call:
    inputs:
      enable_slack:
        type: boolean
        default: false
    secrets:
      SLACK_WEBHOOK:
        required: false  # requis seulement si enable_slack=true

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Valider configuration Slack
        if: ${{ inputs.enable_slack }}
        run: |
          if [ -z "${{ secrets.SLACK_WEBHOOK }}" ]; then
            echo "::error::SLACK_WEBHOOK requis quand enable_slack=true"
            exit 1
          fi
```

### Secrets par environnement

```yaml
jobs:
  deploy-dev:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: dev
    secrets:
      DEPLOY_KEY: ${{ secrets.DEV_DEPLOY_KEY }}
      
  deploy-prod:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: prod
    secrets:
      DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
```

### Construire des secrets composites

```yaml
steps:
  - name: Créer configuration SSH
    run: |
      mkdir -p ~/.ssh
      echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
      chmod 600 ~/.ssh/id_rsa
      echo "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
      
  - name: Créer fichier .env
    run: |
      cat > .env << EOF
      DATABASE_URL=${{ secrets.DATABASE_URL }}
      API_KEY=${{ secrets.API_KEY }}
      REDIS_URL=${{ secrets.REDIS_URL }}
      EOF
      chmod 600 .env
```

## Exemple complet

```yaml
name: Déploiement SFTP sécurisé

on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      REMOTE_CHEMIN:
        description: 'Chemin racine sur le serveur (/homepages/XX/...)'
        required: true
      SFTP_HOST:
        description: 'Hôte SFTP (access-ssh.ionos.fr)'
        required: true
      SFTP_USER:
        description: 'Utilisateur SFTP'
        required: true
      SFTP_PORT:
        description: 'Port SSH (défaut 22)'
        required: false
      SSH_PRIVATE_KEY:
        description: 'Clé privée SSH au format PEM'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Vérifier secrets requis
        run: |
          missing=0
          for secret in REMOTE_CHEMIN SFTP_HOST SFTP_USER SSH_PRIVATE_KEY; do
            var_name="${secret}"
            case "$var_name" in
              REMOTE_CHEMIN)
                value="${{ secrets.REMOTE_CHEMIN }}"
                ;;
              SFTP_HOST)
                value="${{ secrets.SFTP_HOST }}"
                ;;
              SFTP_USER)
                value="${{ secrets.SFTP_USER }}"
                ;;
              SSH_PRIVATE_KEY)
                value="${{ secrets.SSH_PRIVATE_KEY }}"
                ;;
            esac
            
            if [ -z "$value" ]; then
              echo "::error title=Secret manquant::$var_name n'est pas défini"
              missing=1
            fi
          done
          
          [ "$missing" -eq 0 ] || exit 1
          
      - name: Configurer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
          
      - name: Déployer
        run: |
          rsync -az \
            -e "ssh -p ${{ secrets.SFTP_PORT || '22' }}" \
            ./ ${{ secrets.SFTP_USER }}@${{ secrets.SFTP_HOST }}:${{ secrets.REMOTE_CHEMIN }}/${{ inputs.environment }}/
```

## Ressources

- [Encrypted secrets documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Using secrets in workflows](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idsecrets)
- [Security hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

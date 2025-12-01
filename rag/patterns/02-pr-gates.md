# 🚪 Gates de validation de PR

## Qu'est-ce qu'un gate ?

Un gate est un workflow qui **bloque** une PR si certaines conditions ne sont pas remplies. Ils sont essentiels pour maintenir la qualité et l'intégrité du code.

## Types de gates

### 1. Gate de source de branche

Vérifie que la PR provient de la bonne branche.

```yaml
name: 🔒 Gate source de PR

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
        run: |
          if [ "${{ github.base_ref }}" = "homol" ] && [ "${{ github.head_ref }}" != "develop" ]; then
            echo "::error::Seules les PR de develop vers homol sont autorisées"
            echo "### ❌ PR refusée" >> $GITHUB_STEP_SUMMARY
            echo "Source: \`${{ github.head_ref }}\`" >> $GITHUB_STEP_SUMMARY
            echo "Cible: \`${{ github.base_ref }}\`" >> $GITHUB_STEP_SUMMARY
            exit 1
          fi
          
          echo "### ✅ PR valide" >> $GITHUB_STEP_SUMMARY
          echo "Source: \`${{ github.head_ref }}\` → Cible: \`${{ github.base_ref }}\`" >> $GITHUB_STEP_SUMMARY

      - name: Commenter si invalide
        if: ${{ failure() }}
        uses: peter-evans/create-or-update-comment@v4
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            🚫 **PR refusée**
            
            - **Source** : `${{ github.head_ref }}`
            - **Cible** : `${{ github.base_ref }}`
            
            Seules les PR provenant de **`develop`** peuvent être fusionnées dans **`homol`**.
            Merci de créer une PR **develop → homol**.
```

### 2. Gate de lien avec ticket/issue

Force la présence d'un ticket lié à chaque PR.

```yaml
name: 🔗 Gate ticket requis

on:
  pull_request:
    types: [opened, edited, synchronize, ready_for_review, reopened]
    branches: [develop]

permissions:
  contents: read
  pull-requests: write

jobs:
  gate:
    name: Vérifier la présence d'un ticket
    runs-on: ubuntu-latest
    steps:
      - name: Détecter les tickets
        uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            const branch = pr.head.ref || '';
            const title = pr.title || '';
            const body = pr.body || '';
            
            // Récupérer les commits
            const commits = await github.paginate(
              github.rest.pulls.listCommits,
              { owner: context.repo.owner, repo: context.repo.repo, pull_number: pr.number }
            );
            const commitMessages = commits.map(c => c.commit.message).join('\n');
            
            const haystack = [branch, title, body, commitMessages].join('\n');
            
            // Regex pour détecter les tickets
            const patterns = [
              /#(\d+)/g,                          // #123
              /ticket[-_ ]?(\d+)/ig,              // ticket-123, ticket 123
              /\b[A-Z][A-Z0-9_]+-(\d+)\b/g,       // PROJ-123
              /(?:close[sd]?|fix(?:es|ed)?|resolve[sd]?)\s+#(\d+)/ig  // Fixes #123
            ];
            
            const found = new Set();
            for (const pattern of patterns) {
              for (const match of haystack.matchAll(pattern)) {
                const num = Number(match[1]);
                if (Number.isInteger(num)) found.add(num);
              }
            }
            
            if (found.size === 0) {
              core.setFailed(
                'Cette PR n\'est liée à aucun ticket/issue.\n' +
                'Formats acceptés : #123 | ticket-123 | PROJ-123 | Fixes #123'
              );
              return;
            }
            
            console.log(`✅ Tickets détectés: ${[...found].join(', ')}`);
```

### 3. Gate de vérification serveur

Vérifie que la connexion au serveur fonctionne avant de permettre le merge.

```yaml
name: 🔳 Gate serveur

on:
  pull_request:
    types: [opened, synchronize]
    branches: [develop]

permissions:
  contents: read

concurrency:
  group: server-check-${{ github.event.pull_request.head.ref }}
  cancel-in-progress: true

env:
  SFTP_HOST: ${{ secrets.SFTP_HOST }}
  SFTP_USER: ${{ secrets.SFTP_USER }}
  SFTP_PORT: ${{ secrets.SFTP_PORT }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

jobs:
  gate:
    name: Vérification serveur
    runs-on: ubuntu-latest
    steps:
      - name: Vérifier secrets requis
        run: |
          missing=0
          for var in SFTP_HOST SFTP_USER SSH_PRIVATE_KEY; do
            if [ -z "${!var}" ]; then
              echo "::error::$var non défini"
              missing=1
            fi
          done
          [ "$missing" -eq 0 ] || exit 1

      - name: Préparer SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ env.SSH_PRIVATE_KEY }}

      - name: Tester connexion
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p "${SFTP_PORT:-22}" "$SFTP_HOST" >> ~/.ssh/known_hosts
          ssh -p "${SFTP_PORT:-22}" "$SFTP_USER@$SFTP_HOST" "echo 'Connexion SSH OK'"
```

### 4. Gate de format de titre de PR

Impose un format spécifique pour le titre des PR.

```yaml
name: 📝 Gate format titre PR

on:
  pull_request:
    types: [opened, edited, synchronize]
    branches: [develop]

permissions:
  contents: read
  pull-requests: write

jobs:
  gate:
    name: Vérifier format titre
    runs-on: ubuntu-latest
    steps:
      - name: Valider le format
        uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            const title = pr.title;
            
            // Format attendu : [#123] type - description
            // Exemples: [#45] feat - nouvelle feature
            //           [ticket-78] fix - correction bug
            const validFormat = /^\[(?:#\d+|ticket[-_ ]?\d+|[A-Z]+-\d+)\]\s+\w+\s+-\s+.+/i;
            
            if (!validFormat.test(title)) {
              core.setFailed(
                'Le titre de la PR ne respecte pas le format requis.\n\n' +
                'Format attendu : `[#123] type - description`\n' +
                'Exemple : `[#45] feat - ajout nouveau module`\n\n' +
                'Types valides : feat, fix, docs, refactor, test, chore, etc.'
              );
              
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: pr.number,
                body: '⚠️ **Format de titre invalide**\n\n' +
                      'Format attendu : `[#123] type - description`\n' +
                      'Exemple : `[#45] feat - ajout nouveau module`'
              });
              
              return;
            }
            
            console.log('✅ Format de titre valide');
```

### 5. Gate de tests

Impose que les tests passent avant le merge.

```yaml
name: ✅ Gate tests

on:
  pull_request:
    types: [opened, synchronize]
    branches: [develop, homol, main]

permissions:
  contents: read

jobs:
  gate:
    name: Exécuter les tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Installer les dépendances
        run: npm ci
        
      - name: Lancer les tests
        run: npm test
        
      - name: Vérifier la couverture
        run: |
          COVERAGE=$(npm test -- --coverage --json | jq '.coverageMap | length')
          if [ "$COVERAGE" -lt 80 ]; then
            echo "::error::Couverture insuffisante: ${COVERAGE}%"
            exit 1
          fi
```

## Configuration des gates comme checks requis

### Dans GitHub

1. **Settings** → **Branches** → **Branch protection rules**
2. Sélectionner la branche (`develop`, `homol`, `main`)
3. Activer **Require status checks to pass before merging**
4. Sélectionner les gates dans **Status checks that are required** :
   - `Vérifier la source de la PR`
   - `Vérifier la présence d'un ticket`
   - `Vérification serveur`
   - `Vérifier format titre`
   - `Exécuter les tests`

⚠️ **Important :** Les checks n'apparaissent dans la liste que s'ils ont été exécutés au moins une fois. Créez une PR de test pour les faire apparaître.

## Combinaison de gates

Vous pouvez combiner plusieurs gates dans un seul workflow :

```yaml
name: 🚪 Gates de validation PR

on:
  pull_request:
    types: [opened, edited, synchronize, ready_for_review, reopened]
    branches: [develop]

permissions:
  contents: read
  pull-requests: write

jobs:
  gate-ticket:
    name: ✅ Ticket requis
    runs-on: ubuntu-latest
    steps:
      # ... vérification ticket
      
  gate-format:
    name: ✅ Format titre
    runs-on: ubuntu-latest
    steps:
      # ... vérification format
      
  gate-tests:
    name: ✅ Tests passent
    runs-on: ubuntu-latest
    steps:
      # ... exécution tests
      
  gate-server:
    name: ✅ Serveur accessible
    runs-on: ubuntu-latest
    steps:
      # ... vérification serveur
```

Tous ces jobs doivent passer pour que la PR soit mergeable.

## Gates conditionnels

Certains gates ne s'appliquent que dans certains contextes :

```yaml
jobs:
  gate-source:
    name: Vérifier source PR
    runs-on: ubuntu-latest
    if: ${{ github.base_ref == 'homol' || github.base_ref == 'main' }}
    steps:
      - name: Valider source
        run: |
          case "${{ github.base_ref }}" in
            homol)
              [ "${{ github.head_ref }}" = "develop" ] || exit 1
              ;;
            main)
              [ "${{ github.head_ref }}" = "homol" ] || exit 1
              ;;
          esac
```

## Notifications des échecs

Ajouter des commentaires informatifs quand un gate échoue :

```yaml
- name: Commenter l'échec
  if: ${{ failure() }}
  uses: peter-evans/create-or-update-comment@v4
  with:
    issue-number: ${{ github.event.pull_request.number }}
    body: |
      ❌ **Gate de validation échoué**
      
      **Problème :** ${{ steps.check.outputs.error }}
      
      **Solution :** ${{ steps.check.outputs.fix }}
      
      ---
      
      Pour plus d'informations, consultez les logs du workflow.
```

## Bonnes pratiques

### ✅ À faire

1. **Messages d'erreur clairs**
   ```yaml
   echo "::error title=Ticket manquant::Ajoutez un ticket (#123) dans le titre ou la description"
   ```

2. **Step summary pour visibilité**
   ```yaml
   echo "### ❌ Gate échoué" >> $GITHUB_STEP_SUMMARY
   echo "Raison: ticket manquant" >> $GITHUB_STEP_SUMMARY
   ```

3. **Commenter les PR pour guider**
   ```yaml
   - uses: peter-evans/create-or-update-comment@v4
   ```

4. **Conditions claires**
   ```yaml
   if: ${{ github.base_ref == 'homol' }}
   ```

5. **Tests avant déploiement**
   - Gate serveur avant merge
   - Déploiement après merge

### ❌ À éviter

1. **Gates trop stricts** qui bloquent le travail légitime
2. **Messages d'erreur vagues** ("Échec", sans explication)
3. **Pas de documentation** sur comment résoudre l'échec
4. **Gates sans bypass** pour les cas d'urgence
5. **Trop de gates** qui ralentissent le workflow

## Bypass des gates

Pour les administrateurs, en cas d'urgence :

### Option 1 : Bypass via GitHub UI

**Settings** → **Branches** → **Branch protection rules**
- Décocher temporairement **Require status checks**
- Merger la PR
- Réactiver les checks

### Option 2 : Bypass dans le code

```yaml
jobs:
  gate:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'bypass-gates') }}
    runs-on: ubuntu-latest
    steps:
      # ... validations
```

Ajouter le label `bypass-gates` à la PR pour ignorer les gates.

⚠️ **À utiliser avec précaution !**

## Exemple complet : Système de gates

```yaml
name: 🚪 Gates de validation complète

on:
  pull_request:
    types: [opened, edited, synchronize, ready_for_review, reopened]
    branches: [develop, homol, main]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  # Gate 1 : Source de branche
  gate-source:
    name: ✅ Source de branche valide
    if: ${{ github.base_ref == 'homol' || github.base_ref == 'main' }}
    runs-on: ubuntu-latest
    steps:
      - name: Valider source
        run: |
          case "${{ github.base_ref }}" in
            homol)
              if [ "${{ github.head_ref }}" != "develop" ]; then
                echo "::error::PR vers homol doit provenir de develop"
                exit 1
              fi
              ;;
            main)
              if [ "${{ github.head_ref }}" != "homol" ]; then
                echo "::error::PR vers main doit provenir de homol"
                exit 1
              fi
              ;;
          esac
          
  # Gate 2 : Ticket lié
  gate-ticket:
    name: ✅ Ticket lié
    if: ${{ github.base_ref == 'develop' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            const haystack = [pr.head.ref, pr.title, pr.body].join(' ');
            const hasTicket = /#\d+|ticket[-_ ]?\d+/i.test(haystack);
            
            if (!hasTicket) {
              core.setFailed('Aucun ticket détecté');
            }
            
  # Gate 3 : Tests
  gate-tests:
    name: ✅ Tests passent
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

## Ressources

- [Branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Required status checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging)
- [actions/github-script](https://github.com/actions/github-script)

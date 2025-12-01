# 📚 RAG - Base de connaissances pour workflows GitHub réutilisables

Ce dossier contient toute la documentation nécessaire pour **créer, maintenir et réutiliser** des workflows GitHub Actions dans vos projets.

## 🗂️ Structure de la documentation

### 📖 Fondamentaux
- **[01-workflow-call.md](./fondamentaux/01-workflow-call.md)** : Comprendre `workflow_call` et rendre un workflow réutilisable
- **[02-inputs-outputs.md](./fondamentaux/02-inputs-outputs.md)** : Définir et utiliser les inputs/outputs
- **[03-secrets.md](./fondamentaux/03-secrets.md)** : Gérer les secrets (inherit vs mapping explicite)
- **[04-permissions.md](./fondamentaux/04-permissions.md)** : Configurer les permissions GITHUB_TOKEN
- **[05-concurrency.md](./fondamentaux/05-concurrency.md)** : Contrôler l'exécution concurrente des workflows

### 🏗️ Patterns & Bonnes pratiques
- **[01-deployment-pattern.md](./patterns/01-deployment-pattern.md)** : Pattern de déploiement SFTP (dev/homol/prod)
- **[02-pr-gates.md](./patterns/02-pr-gates.md)** : Gates de validation de PR (vérifications avant merge)
- **[03-version-gates.md](./patterns/03-version-gates.md)** : Gates de versioning (pré-release, release)
- **[04-issue-linking.md](./patterns/04-issue-linking.md)** : Lier automatiquement les tickets/issues aux PR
- **[05-branch-protection.md](./patterns/05-branch-protection.md)** : Configuration des protections de branches
- **[06-matrix-strategy.md](./patterns/06-matrix-strategy.md)** : Utiliser les matrices pour multi-environnements

### 🔧 Guides pratiques
- **[01-converting-workflow.md](./guides/01-converting-workflow.md)** : Convertir un workflow existant en réutilisable
- **[02-calling-workflow.md](./guides/02-calling-workflow.md)** : Appeler un workflow réutilisable depuis un dépôt consommateur
- **[03-versioning-workflows.md](./guides/03-versioning-workflows.md)** : Versionner et taguer vos workflows
- **[04-organization-secrets.md](./guides/04-organization-secrets.md)** : Configurer les secrets au niveau Organisation
- **[05-debugging.md](./guides/05-debugging.md)** : Déboguer les workflows (logs, step summary, etc.)

### 📝 Templates & Exemples
- **[reusable-template.yml](./templates/reusable-template.yml)** : Template de base pour un workflow réutilisable
- **[consumer-template.yml](./templates/consumer-template.yml)** : Template pour appeler un workflow réutilisable
- **[deployment-example.yml](./templates/deployment-example.yml)** : Exemple complet de workflow de déploiement
- **[gate-example.yml](./templates/gate-example.yml)** : Exemple de gate de validation

### 🔍 Références
- **[glossaire.md](./references/glossaire.md)** : Terminologie et concepts clés
- **[actions-utilisees.md](./references/actions-utilisees.md)** : Catalogue des actions GitHub utilisées
- **[troubleshooting.md](./references/troubleshooting.md)** : Solutions aux problèmes courants

## 🚀 Démarrage rapide

### Pour créer un nouveau workflow réutilisable
1. Lire **[01-workflow-call.md](./fondamentaux/01-workflow-call.md)**
2. Utiliser **[reusable-template.yml](./templates/reusable-template.yml)** comme point de départ
3. Consulter **[01-converting-workflow.md](./guides/01-converting-workflow.md)** si vous convertissez un workflow existant

### Pour utiliser un workflow réutilisable
1. Lire **[02-calling-workflow.md](./guides/02-calling-workflow.md)**
2. Utiliser **[consumer-template.yml](./templates/consumer-template.yml)** comme exemple
3. Configurer les secrets selon **[04-organization-secrets.md](./guides/04-organization-secrets.md)**

### Pour mettre en place un système de déploiement
1. Étudier **[01-deployment-pattern.md](./patterns/01-deployment-pattern.md)**
2. Adapter **[deployment-example.yml](./templates/deployment-example.yml)**
3. Configurer les protections selon **[05-branch-protection.md](./patterns/05-branch-protection.md)**

## 📋 Checklist de mise en place

- [ ] Créer le dépôt central de workflows (`corbidev-actions-central`)
- [ ] Définir les secrets au niveau Organisation
- [ ] Convertir les workflows existants en réutilisables
- [ ] Taguer la première version (`v1`)
- [ ] Créer les workflows consommateurs dans les dépôts
- [ ] Configurer les protections de branches
- [ ] Tester les déploiements sur chaque environnement

## 🔗 Liens utiles

- [Documentation officielle GitHub Actions](https://docs.github.com/en/actions)
- [Reusable workflows documentation](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

---

💡 **Astuce** : Cette documentation est structurée pour être parcourue de manière progressive. Commencez par les fondamentaux, puis explorez les patterns selon vos besoins.

# GitHub Workflows – IONOS (Dev / Homol / Prod)

Ce dépôt fournit un jeu complet de workflows GitHub Actions pour piloter les déploiements SFTP chez IONOS, sécuriser les PR et automatiser quelques tâches de maintenance.

## Aperçu rapide

| Workflow                               | Fichier                                           | Déclenchement                           | Résumé                                                             |
| -------------------------------------- | ------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------ |
| 🔳 Vérification serveur Dev            | `.github/workflows/dev-server-check-pr.yml`       | PR → `develop` (open/sync)              | Teste la connexion SFTP et crée le dossier `/dev` si besoin.       |
| 🚀 Déploiement Dev                     | `.github/workflows/dev-deploy.yml`                | PR → `develop` (merged)                 | Déploie sur l'environnement **dev** via `rsync` après merge.       |
| 🔒 PR de develop vers homol            | `.github/workflows/homol-check-pr-depuis-dev.yml` | PR → `homol`                            | Refuse les PR qui ne proviennent pas de `develop`.                 |
| 🔳 Vérification serveur Homol          | `.github/workflows/homol-server-check-pr.yml`     | PR → `homol` (open/sync)                | Vérifie l'accès SFTP et prépare le dossier `/homol`.               |
| 🚀 Déploiement Homol                   | `.github/workflows/homol-deploy.yml`              | PR → `homol` (merged)                   | Aligne `homol` sur `develop` puis déploie en SFTP.                 |
| 🔳 Vérification serveur Prod           | `.github/workflows/prod-server-check-pr.yml`      | PR → `main` (open/sync)                 | Vérifie SFTP et prépare le dossier `/prod`.                        |
| 🚀 Déploiement Prod                    | `.github/workflows/prod-deploy.yml`               | PR → `main` (merged)                    | Aligne `main` sur `homol` puis déploie en SFTP.                    |
| 🔗 Lier les tickets à la PR            | `.github/workflows/link-issues-in-pr.yml`         | PR → `develop`                          | Force la présence d'un ticket (#123, ticket-123, ABC-123…).        |
| 🔄 Mettre à jour le titre de la PR     | `.github/workflows/update-pr-title.yml`           | PR → `develop`                          | Normalise le titre de la PR depuis le nom de branche.              |
| 📝 Commenter les commits sur le ticket puis le cloturer | `.github/workflows/comment-and-close-ticket.yml`  | PR → `develop` (merged)                 | Commente et ferme l'issue liée en listant les commits mergés.      |
| 🗑️ Supprimer la branche après fusion   | `.github/workflows/delete-branch-after-merge.yml` | Toute PR fermée                         | Supprime automatiquement la branche source (hors liste interdite). |
| 🔣 Analyse CodeQL                      | `.github/workflows/CodeQL.yml`                    | Push/PR `main`,`homol`,`develop` + cron | Analyse statique JS/TS si du code est détecté.                     |
|   |          |            |            |

> ℹ️ Les intitulés affichés dans GitHub correspondent aux noms de jobs (ex. `Vérification connexion & dossier serveur dev`). Ouvre une PR de test pour qu'ils apparaissent avant de les marquer comme checks obligatoires.

## Installation

Les fichiers doivent être présents dans le dossier `.github/workflows` de ton dépôt :

```plaintext
.github/
└─ workflows/
   ├─ CodeQL.yml
   ├─ comment-and-close-ticket.yml
   ├─ delete-branch-after-merge.yml
   ├─ dev-deploy.yml
   ├─ dev-server-check-pr.yml
   ├─ homol-check-pr-depuis-dev.yml
   ├─ homol-deploy.yml
   ├─ homol-server-check-pr.yml
   ├─ link-issues-in-pr.yml
   ├─ prod-deploy.yml
   ├─ prod-server-check-pr.yml
   └─ update-pr-title.yml
```

## Variables & secrets requis

Configurer ces valeurs dans **Settings → Secrets and variables → Actions** du dépôt.

### Variables (Repository variables)

- `ADRESSE_GLOBAL` : domaine maître IONOS (ex. `corbisier.fr`).
- `ADRESSE_LOCAL` : répertoire virtualhost (ex. `web-git`).

### Secrets communs

- `REMOTE_CHEMIN` : racine hôte (ex. `/homepages/XX/dXXXXXXXX/htdocs`).
- `SFTP_HOST` : hôte SSH/SFTP (ex. `ssh.strato.com` ou `access-ssh.ionos.fr`).
- `SFTP_USER` : utilisateur SFTP autorisé.
- `SFTP_PORT` : port SSH (défaut `22`, optionnel).
- `SSH_PRIVATE_KEY` : clé privée correspondant à la clé publique installée dans `~/.ssh/authorized_keys` côté IONOS.

Ces workflows n'utilisent plus FTPS ; inutile de définir `FTP_SERVER`, `FTP_USERNAME` ou `FTP_PASSWORD`.

## Workflows par environnement

### Branche `develop`

- **🔳 Vérification serveur Dev** : exécute une connexion SSH, valide/initialise le dossier distant `.../ADRESSE_GLOBAL/dev/ADRESSE_LOCAL`.
- **🔗 Lier les tickets à la PR** : bloque la PR si aucun ticket n'est détecté (titre, body, nom de branche ou messages de commit).
- **🔄 Mettre à jour le titre de la PR** : reformate automatiquement le titre (`[#123] type - libellé`) à partir du nom de branche.
- **📝 Commenter les commits sur le ticket puis le cloturer** : après merge, commente l'issue détectée avec la liste des commits et la ferme.

### Branche `homol`

- **🔒 PR de develop vers homol** : si la source de la PR n'est pas `develop`, le workflow échoue et ajoute un commentaire.
- **🔳 Vérification serveur Homol** : même logique que pour `dev`, mais sur `/homol/`.
- **🚀 Déploiement Homol** : après merge d'une PR `develop → homol`, aligne la branche `homol` sur `develop` (`git reset --hard` + `force-with-lease`) puis déploie via `rsync`.

### Branche `main`

- **🔳 Vérification serveur Prod** : vérifie la présence/création du dossier distant de production.
- **🚀 Déploiement Prod** : après merge d'une PR `homol → main`, aligne `main` sur `homol` et déploie l'environnement `prod`.

## Automatisations transverses

- **🗑️ Supprimer la branche après fusion** : supprime la branche source d'une PR fusionnée, sauf si elle figure dans `FORBIDDEN_BRANCHES` (par défaut `develop,homol`).

- **🔣 Analyse CodeQL** : lance l'analyse CodeQL JS/TS sur chaque push/PR et hebdomadairement (`cron`). Le job s'exécute uniquement si des fichiers JS/TS existent.

## Protection de branches recommandée

1. **Settings → Branches → Branch protection rules → New rule**

2. Règles suggérées :
   - `develop` : activer _Require a pull request before merging_, puis marquer comme checks obligatoires `Vérification connexion & dossier serveur dev` et (si pertinent) `codeql`.
   - `homol` : même configuration, ajouter `Vérification connexion & dossier serveur homol` et `Vérifier la source de la PR` (job du workflow `homol-check-pr-depuis-dev`).
   - `main` : exiger une PR, cocher `Vérification connexion & dossier serveur prod` et `codeql`.
3. Les workflows de déploiement (`Déploiement Dev/Homol/Prod`) s'exécutent **après** le merge et ne peuvent pas être requis comme checks.
4. Pour qu'un job apparaisse dans la liste, exécute-le au moins une fois (PR de test).

## Personnalisation

- Ajoute un fichier `.rsync-ignore` à la racine pour exclure des répertoires/fichiers lors des déploiements.
- Adapte `FORBIDDEN_BRANCHES` dans `delete-branch-after-merge.yml` si d'autres branches ne doivent jamais être supprimées.
- Modifie la logique de normalisation des titres dans `update-pr-title.yml` pour inclure d'autres préfixes ou formats de ticket.
- Ajuste la configuration `SFTP_PORT` ou les options `ssh-keyscan` selon l'hébergeur.

## Dépannage

- **Échec de connexion SFTP** : vérifier `SFTP_HOST`, `SFTP_USER`, `SSH_PRIVATE_KEY`, `SFTP_PORT` et que la clé publique est bien installée chez IONOS.
- **Dossier distant introuvable** : lancer le workflow `Vérification serveur` correspondant pour créer les répertoires (`mkdir -p`), ou vérifier le chemin via `REMOTE_CHEMIN`, `ADRESSE_GLOBAL`, `ADRESSE_LOCAL`.
- **PR bloquée par Lier les tickets à la PR** : ajouter une référence explicite (`Fixes #123`, `ticket-123`, `ABC-123`) dans la branche, le titre ou la description.
- **Branche non supprimée** : consulter le commentaire automatique pour connaître la raison (branche interdite, dépôt forké, etc.).

Bon déploiement ! 🚀

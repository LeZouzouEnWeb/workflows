# corbidev-actions-central

Centralisation de **TES workflows** en versions **réutilisables** (`on: workflow_call`) quand nécessaire.

## Contenu converti

- `CodeQL.yml` → converti
- `comment-and-close-ticket.yml` → converti
- `delete-branch-after-merge.yml` → converti
- `dev-deploy.yml` → converti
- `dev-server-check-pr.yml` → converti
- `homol-check-pr-depuis-dev.yml` → converti
- `homol-deploy.yml` → converti
- `homol-server-check-pr.yml` → converti
- `link-issues-in-pr.yml` → converti
- `prod-deploy.yml` → converti
- `prod-server-check-pr.yml` → converti
- `update-pr-title.yml` → converti
- `link-project-in-pr.yml` → converti
- `prod-check-pr-depuis-homol.yml` → converti

## Utilisation côté dépôts consommateurs

### Appeler un workflow partagé

```yaml
jobs:
  my_job:
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/NAME.yml@v1
    # with:    # renseigner les inputs si le workflow en expose
    #   key: value
    # secrets: # soit inherit, soit liste contrôlée
    #   MY_SECRET: ${ secrets.MY_SECRET }
```

> Remplace `NAME.yml` par le fichier à appeler (voir la liste ci-dessus).

### Tagging

Après avoir poussé sur `main`, crée un tag stable :

```bash
git tag v1
git push origin v1
```

### Notes

- Les workflows **déjà** réutilisables sont conservés.
- Ceux sans `workflow_call` ont été **convertis**. Leurs **jobs** d'origine sont préservés ; les anciens triggers sont placés en **commentaire** en tête du fichier.
- Si certains workflows attendent des **inputs/secrets**, on peut les déclarer finement dans la section `on.workflow_call` au besoin.

## Convention chemins (déploiement)

`REMOTE_CHEMIN / ADRESSE_GLOBAL / $ENV / ADRESSE_LOCAL`

- **REMOTE_CHEMIN** : _secret commun (Organisation)_
- **ADRESSE_GLOBAL** : _var par dépôt_
- **ADRESSE_LOCAL** : _var par dépôt_
- **$ENV** : `dev|homol|prod` (input `remote_env`)

## Secrets communs (Organisation)

Tous les secrets `REMOTE*`, `SSH_*`, `FTPS_*` sont définis **au niveau Organisation** (option _Selected repositories_ possible) :  
`REMOTE_CHEMIN`, `REMOTE_SSH`, `SSH_PRIVATE_KEY`, `SSH_KNOWN_HOSTS`, `FTPS_HOST`, `FTPS_USER`, `FTPS_PASSWORD`, etc.

## Inputs requis (réutilisables)

- `remote_env` (string; `dev|homol|prod`)
- `ADRESSE_GLOBAL` (string; var repo)
- `ADRESSE_LOCAL` (string; var repo)

## Exemple d'appel (mapping strict)

```yaml
jobs:
  deploy:
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/<workflow>.yml@v1
    with:
      remote_env: ${{ matrix.remote_env }}
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    secrets:
      REMOTE_CHEMIN: ${{ secrets.REMOTE_CHEMIN }}
      REMOTE_SSH: ${{ secrets.REMOTE_SSH }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

## Exemple d'appel (héritage global des secrets)

```yaml
jobs:
  deploy:
    uses: LeZouzouEnWeb/corbidev-actions-central/.github/workflows/<workflow>.yml@v1
    with:
      remote_env: ${{ matrix.remote_env }}
      ADRESSE_GLOBAL: ${{ vars.ADRESSE_GLOBAL }}
      ADRESSE_LOCAL: ${{ vars.ADRESSE_LOCAL }}
    secrets: inherit
```

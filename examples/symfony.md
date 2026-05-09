Symfony à la racine
YAML
name: Tests

on:
  push:
    branches:
      - main

  pull_request:

jobs:
  tests:
    uses: my-org/github-workflows/.github/workflows/symfony.yml@main
Symfony dans /api
YAML
name: Tests

on:
  push:
    branches:
      - main

  pull_request:

jobs:
  tests:
    uses: my-org/github-workflows/.github/workflows/symfony.yml@main

    with:
      symfony-dir: api
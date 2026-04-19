# Projet CI/CD - Application Python

**Noms des membres du groupe :** [À REMPLIR]

---

## 1. Comment mettre en place le projet (Guide de Configuration)

Voici les étapes à suivre pour configurer correctement ce projet sur GitHub et Docker Hub :

### Étape 1 : Préparation du Dépôt GitHub
1. Initialisez un dépôt Git local dans ce dossier si ce n'est pas déjà fait :
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   ```
2. Créez un nouveau dépôt public sur GitHub.
3. Liez votre dépôt local au dépôt GitHub et poussez le code :
   ```bash
   git branch -M main
   git remote add origin https://github.com/VOTRE_UTILISATEUR/VOTRE_DEPOT.git
   git push -u origin main
   ```

### Étape 2 : Configuration de Docker Hub
1. Connectez-vous à votre compte [Docker Hub](https://hub.docker.com/).
2. Allez dans **Account Settings** -> **Personal Access Tokens**.
3. Cliquez sur **New Access Token**, donnez-lui un nom (ex: `github-actions`), donnez-lui les permissions de lecture et écriture (Read & Write), puis copiez le token généré.

### Étape 3 : Ajout des Secrets dans GitHub
Pour que le pipeline CD puisse se connecter à Docker Hub et pousser l'image, vous devez configurer les secrets :
1. Allez sur votre dépôt GitHub, cliquez sur l'onglet **Settings**.
2. Dans le menu de gauche, déroulez **Secrets and variables** puis cliquez sur **Actions**.
3. Cliquez sur le bouton vert **New repository secret**.
4. Ajoutez un premier secret :
   - **Name :** `DOCKER_USERNAME`
   - **Secret :** Votre nom d'utilisateur Docker Hub (ex: `mon_user_docker`)
5. Ajoutez un deuxième secret :
   - **Name :** `DOCKER_PASSWORD`
   - **Secret :** Le Personal Access Token copié à l'étape 2.

### Étape 4 : Activation de la protection et des permissions GitHub Actions (Optionnel mais recommandé)
1. Allez dans **Settings** -> **Actions** -> **General**.
2. Assurez-vous que l'option **Allow all actions and reusable workflows** est cochée.
3. Descendez jusqu'à **Workflow permissions** et sélectionnez **Read and write permissions** si nécessaire.

### Étape 5 : Déclenchement du Pipeline
- Le pipeline CI est configuré pour se lancer à chaque `push` sur la branche `main` (ou `master`).
- Vous pouvez vérifier l'état d'avancement dans l'onglet **Actions** de votre dépôt GitHub.
- Une fois le pipeline **CI** (tests, flake8, trivy) terminé avec succès, le pipeline **CD** se déclenchera automatiquement pour construire et pousser l'image Docker.

---

## 2. Ce qu'il faut soumettre (À remplir pour l'examen)

**1. URL du dépôt GitHub public :**
[https://github.com/VOTRE_UTILISATEUR/VOTRE_DEPOT](https://github.com/VOTRE_UTILISATEUR/VOTRE_DEPOT)

**2. Capture d'écran du pipeline CI réussi dans GitHub Actions :**
*(Ajoutez votre image ici, ex: `![CI Pipeline](lien_image)`)*

**3. Capture d'écran du pipeline CD réussi :**
*(Ajoutez votre image ici, ex: `![CD Pipeline](lien_image)`)*

**4. Capture d'écran du dépôt Docker Hub montrant l'image poussée :**
*(Ajoutez votre image ici, ex: `![Docker Hub](lien_image)`)*

**5. Fichiers créés :**

**Fichier `.github/workflows/ci.yml` :**
```yaml
name: CI

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

permissions:
  contents: read
  security-events: write
  actions: read

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.8", "3.9", "3.10"]

    steps:
    - name: checkout
      uses: actions/checkout@v5
    
    - name: Python ${{ matrix.python-version }}
      uses: actions/setup-python@v6
      with:
        python-version: ${{ matrix.python-version }}
        
    - name: dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8 pytest
        
    - name: flake8
      run: |
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --exit-zero --statistics
        
    - name: pytest
      run: |
        pytest tests/

  trivy-scan:
    runs-on: ubuntu-latest
    steps:
    - name: checkout
      uses: actions/checkout@v5
      
    - name: trivy FS mode
      uses: aquasecurity/trivy-action@0.33.1
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'results.sarif'
        severity: 'CRITICAL,HIGH'
        
    - name: upload
      uses: github/codeql-action/upload-sarif@v4
      with:
        sarif_file: 'results.sarif'
```

**Fichier `.github/workflows/cd.yml` :**
```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    types:
      - completed

jobs:
  build:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: checkout
        uses: actions/checkout@v5

      - name: login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: build and push
        id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/calculator:latest
```

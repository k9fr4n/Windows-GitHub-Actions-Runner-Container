# Windows GitHub Actions Runner Container

Image Docker basée sur Windows Server Core LTSC 2025, qui configure au démarrage un runner GitHub Actions auto-hébergé. Elle installe le runner officiel `actions/runner` et Git for Windows ; aucun jeton n'est inclus dans l'image.

## Prérequis

- Un hôte Windows Server 2025 avec Docker Engine configuré en mode conteneurs Windows.
- Une version du système hôte compatible avec l'image `ltsc2025`. Les conteneurs Windows sont liés à la version de Windows de l'hôte ; consultez la [matrice de compatibilité Microsoft](https://learn.microsoft.com/virtualization/windowscontainers/deploy-containers/version-compatibility) avant le déploiement.
- Docker Compose v2 si vous utilisez l'exemple Compose.
- Un jeton d'inscription de runner GitHub temporaire pour le dépôt ou l'organisation ciblés.

## Construire l'image

Depuis un hôte Windows configuré pour les conteneurs Windows :

```powershell
docker build -t windows-github-runner .
```

Les versions du runner et de Git sont épinglées dans les `ARG` du Dockerfile. Pour les mettre à jour, passez `--build-arg RUNNER_VERSION=... --build-arg GIT_VERSION=...` à `docker build`, ou renseignez ces valeurs dans `.env` pour Compose, puis reconstruisez l'image. La CI vérifie le build sur `windows-2025`.

## Démarrer un runner

Créez un jeton d'inscription temporaire dans GitHub sous **Settings → Actions → Runners → New self-hosted runner**. Le jeton est spécifique au dépôt ou à l'organisation et expire rapidement ; fournissez-en un valide au démarrage :

```powershell
docker run --rm `
  -e GITHUB_URL="https://github.com/my-org/my-repository" `
  -e RUNNER_TOKEN="<jeton-d-inscription>" `
  -e RUNNER_NAME="windows-runner-01" `
  -e RUNNER_LABELS="windows,windows-2025" `
  windows-github-runner
```

`GITHUB_URL` et `RUNNER_TOKEN` sont obligatoires. Le nom par défaut est le nom d'ordinateur du conteneur, les labels par défaut sont `windows,windows-2025`, et le répertoire de travail est `_work`. GitHub ajoute les labels par défaut du runner (`self-hosted`, le système d'exploitation et l'architecture) automatiquement. Ne fournissez pas `self-hosted` dans `RUNNER_LABELS`.

| Variable | Description | Valeur par défaut |
| --- | --- | --- |
| `GITHUB_URL` | URL du dépôt ou de l'organisation GitHub | Obligatoire |
| `RUNNER_TOKEN` | Jeton temporaire d'inscription | Obligatoire |
| `RUNNER_NAME` | Nom enregistré du runner | Nom de l'ordinateur |
| `RUNNER_LABELS` | Labels personnalisés séparés par des virgules | `windows,windows-2025` |
| `RUNNER_WORKDIR` | Répertoire de travail du runner | `_work` |
| `RUNNER_REMOVE_TOKEN` | Jeton temporaire de suppression du runner | Aucun |

## Docker Compose et plusieurs runners

Copiez `.env.example` vers `.env`, renseignez les valeurs localement puis lancez :

```powershell
docker compose up -d --build
```

Le fichier `.env` est ignoré par Git et exclu du contexte de build. Ne le committez jamais. Pour lancer plusieurs runners, chacun doit recevoir son propre jeton d'inscription valide ; un nom non renseigné est déduit du nom d'ordinateur propre au conteneur :

```powershell
docker compose up -d --build --scale github-runner=3
```

Les runners partagent le même ensemble de labels et la même configuration Compose. Pour des noms, labels ou jetons distincts, déployez plusieurs services configurés séparément.

## Désenregistrement et arrêt

Le jeton d'inscription ne donne pas le droit de retirer un runner. Pour que l'entrypoint puisse appeler `config.cmd remove` après l'arrêt du runner, fournissez également `RUNNER_REMOVE_TOKEN`, généré avec l'endpoint GitHub de suppression correspondant au dépôt ou à l'organisation. Ce jeton est lui aussi temporaire : s'il expire avant l'arrêt du conteneur, la suppression échouera. Sans ce jeton valide, le runner démarre normalement, mais l'entrée GitHub peut rester hors ligne et nécessiter une suppression manuelle.

`docker stop` laisse au runner une période d'arrêt avant de forcer l'arrêt du conteneur. L'entrypoint tente la suppression uniquement si la configuration a réussi et si un jeton de suppression est disponible. Le runner n'est pas configuré en mode éphémère : il peut donc recevoir plusieurs jobs pendant la durée de vie du conteneur.

## Utilisation dans un workflow

```yaml
jobs:
  build:
    runs-on:
      - self-hosted
      - windows
      - windows-2025
    steps:
      - uses: actions/checkout@v4
      - name: Build
        shell: powershell
        run: Write-Host "Running in a Windows container"
```

## Limites

- L'image fournit Git et le runner, pas une copie complète des outils des runners hébergés par GitHub. Ajoutez explicitement les outils nécessaires aux workflows.
- L'accès au moteur Docker de l'hôte depuis un conteneur Windows n'est pas configuré. Docker-in-Docker Windows n'est pas pris en charge par défaut ni garanti par ce projet.
- Les conteneurs Windows exigent un hôte Windows compatible et ne peuvent pas être construits ou exécutés sur un hôte Linux standard.
- Évitez de passer des secrets dans des images, des arguments de build ou des fichiers suivis par Git. Préférez un gestionnaire de secrets ou injectez les variables au runtime. Les variables d'environnement restent accessibles aux processus du conteneur.

## CI

`.github/workflows/build.yml` construit l'image sur un runner GitHub hébergé `windows-2025`. Il ne publie pas d'image dans un registry. Le build Windows requiert un hôte et un moteur Docker compatibles avec les conteneurs Windows.

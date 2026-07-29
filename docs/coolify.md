# Déployer Open-Meteo sur Coolify

Ce guide explique comment héberger votre propre instance de l'API Open-Meteo avec
[Coolify](https://coolify.io). Le fichier [`docker-compose.coolify.yml`](../docker-compose.coolify.yml)
à la racine du dépôt est prêt à l'emploi.

## Prérequis

- Un serveur connecté à Coolify avec au minimum :
  - 4 Go de RAM (8 Go recommandés)
  - 20 Go de disque libre pour commencer (voir la section « Espace disque »)
  - un CPU x86-64 avec AVX2, ou ARM64
- Le dépôt poussé sur un Git accessible par Coolify (GitHub, GitLab, Gitea…),
  ou un déploiement direct depuis l'image publique.

## Déploiement

1. Dans Coolify : **+ New Resource** → **Docker Compose** (ou *Public Repository*
   / *Private Repository* selon votre source), puis sélectionnez le dépôt et la
   branche.
2. Dans les réglages de la ressource, définissez **Docker Compose Location** sur :

   ```
   /docker-compose.coolify.yml
   ```

3. Onglet **Environment Variables** : ajoutez les variables souhaitées (voir le
   tableau ci-dessous). Toutes ont une valeur par défaut, vous pouvez déployer
   sans rien configurer.
4. Onglet **Domains** : Coolify génère automatiquement un domaine pour le
   service `openmeteo-api` grâce à la variable magique `SERVICE_FQDN_OPENMETEOAPI_8080`.
   Remplacez-le par votre propre domaine si besoin, le certificat TLS est géré
   automatiquement.
5. **Deploy**.

Une fois le déploiement terminé :

```bash
curl "https://votre-domaine.tld/v1/forecast?latitude=47.1&longitude=8.4&models=ecmwf_ifs025&hourly=temperature_2m"
```

Les premières requêtes renverront une erreur tant que le service `openmeteo-sync`
n'a pas terminé son premier téléchargement (quelques minutes pour le jeu de
données par défaut). Suivez sa progression dans les logs du conteneur
`openmeteo-sync`.

## Variables d'environnement

| Variable | Défaut | Description |
| --- | --- | --- |
| `LOG_LEVEL` | `info` | Niveau de log (`trace`, `debug`, `info`, `notice`, `warning`, `error`) |
| `SYNC_DOMAINS` | `ecmwf_ifs025` | Modèles météo à synchroniser, séparés par des virgules (ex. `dwd_icon,ncep_gfs013`) |
| `SYNC_VARIABLES` | `temperature_2m` | Variables météo à synchroniser, séparées par des virgules |
| `SYNC_PAST_DAYS` | `3` | Ancienneté maximale des fichiers synchronisés |
| `SYNC_REPEAT_INTERVAL` | `10` | Intervalle de vérification, en minutes |
| `SYNC_CONCURRENT` | `2` | Téléchargements simultanés par modèle |
| `LOCATIONS_LIMIT` | `1000` | Nombre maximal de coordonnées par requête API |
| `REMOTE_DATA_DIRECTORY` | *(vide)* | Mode « sans base locale », voir plus bas |
| `CACHE_SIZE` | `10GB` | Taille du cache disque, uniquement utilisé avec `REMOTE_DATA_DIRECTORY` |

La liste des modèles et variables disponibles est décrite dans le
[tutoriel Weather API](https://github.com/open-meteo/open-data/tree/main/tutorial_weather_api)
et dans [downloading-datasets.md](./downloading-datasets.md).

## Espace disque

Chaque couple modèle × variable ajoute des données. Ordres de grandeur :

- `ecmwf_ifs025` + `temperature_2m` : ~150 Mo par run
- un jeu complet de variables de surface pour un modèle : quelques Go
- accès complet à toutes les prévisions : 150 Go et plus

Commencez petit et ajoutez des variables progressivement. Le volume Docker
`open-meteo-data` est persistant entre les déploiements et partagé entre l'API
et le service de synchronisation.

Pour purger les anciennes données, ajoutez une **Scheduled Task** dans Coolify
(conteneur `openmeteo-sync`) :

```bash
# Données en niveaux de pression au-delà de 10 jours
find /app/data/ -type f -name "chunk_*" -wholename "*hPa*" -mtime +10 -delete

# Données de surface au-delà de 90 jours
find /app/data/ -type f -name "chunk_*" -mtime +90 -delete
```

## Variante légère : lire la base directement depuis S3

L'API peut lire la base Open-Meteo directement depuis le bucket open-data AWS,
avec un cache local, ce qui évite d'héberger la base. C'est adapté à un petit
VPS, au prix d'une latence plus élevée sur les requêtes non mises en cache.

Dans Coolify, définissez :

```
REMOTE_DATA_DIRECTORY=https://openmeteo.s3.amazonaws.com/data/
CACHE_SIZE=2GB
```

(la barre oblique finale est obligatoire), puis mettez le service
`openmeteo-sync` à l'arrêt — ou supprimez-le du fichier compose. Le fichier de
cache est créé dans `/app/data/cache.bin` à la taille indiquée par `CACHE_SIZE`,
assurez-vous d'avoir l'espace disque correspondant.

Cette option n'est pas documentée en amont ; testez-la avant de l'utiliser en
production.

## Compiler depuis les sources

Par défaut le fichier compose utilise l'image officielle
`ghcr.io/open-meteo/open-meteo:latest`. Pour déployer une version modifiée du
code, éditez [`docker-compose.coolify.yml`](../docker-compose.coolify.yml) :
commentez la ligne `image:` du service `openmeteo-api` et décommentez le bloc
`build:`.

Attention : la compilation Swift d'Open-Meteo est lourde (environ 16 Go de RAM
et 30 à 60 minutes). Sur un serveur modeste, le build échouera par manque de
mémoire. Deux alternatives :

- construire l'image ailleurs (CI, machine locale) et pointer `image:` vers
  votre registre ;
- utiliser un serveur de build dédié dans Coolify.

Le `Dockerfile` compile avec `-march=skylake`. Sur un CPU plus ancien que
Skylake, retirez `MARCH_SKYLAKE=TRUE` de la commande `swift build` du
[Dockerfile](../Dockerfile).

## Vérifications et dépannage

- **Le déploiement reste « unhealthy »** : le healthcheck interroge
  `/robots.txt` sur le port 8080. Vérifiez les logs du conteneur `openmeteo-api`.
- **L'API répond `Cannot open file`** : la variable demandée n'a pas encore été
  synchronisée. Vérifiez `SYNC_DOMAINS` / `SYNC_VARIABLES` et les logs de
  `openmeteo-sync`.
- **Erreurs de permission sur `/app/data`** : le conteneur tourne sous
  l'utilisateur `openmeteo`. Si vous remplacez le volume nommé par un bind mount
  vers un dossier de l'hôte, ajoutez `user: "0"` aux deux services ou ajustez le
  propriétaire du dossier.
- **Métriques** : l'endpoint `/metrics` (format Prometheus) est exposé par
  l'API.

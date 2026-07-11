# Proxy partagé (multi-sites)

Permet d'héberger plusieurs projets FrankenPHP sur un seul serveur. Le proxy
Caddy possède les ports 80/443 et le TLS (Let's Encrypt) ; chaque app tourne en
HTTP interne sur le réseau Docker partagé `web` et n'expose aucun port.

```
Internet :80/:443
      │
   [ proxy Caddy ]              ← ce dossier (80/443 + Let's Encrypt)
   ┌────┴───────────┐
amui.dom       chezmarcel.dom
   │                 │
[amui-app]      [chezmarcel-app]   ← FrankenPHP HTTP interne (réseau « web »)
   │                 │
[mysql amuï]    [mysql chez-marcel]  ← une base par projet, isolée
```

## Prérequis

- Un domaine (ou sous-domaine) par projet, pointant en DNS (A/AAAA) sur l'IP du serveur.
- Ports 80 et 443 ouverts sur le serveur.

## Mise en route (une seule fois)

```bash
# 1) Réseau partagé entre le proxy et toutes les apps
docker network create web

# 2) Config du proxy
cd proxy
cp .env.dist .env      # renseigner ACME_EMAIL + les domaines
docker compose up -d
```

## Démarrer chaque site (mode « derrière proxy »)

Dans chaque projet, on utilise `compose.proxy.yaml` (pas `compose.prod.yaml`) et on
met `SERVER_NAME=:80` + `DEFAULT_URI=https://<domaine>` dans son `.env.prod` :

```bash
cd ../amui-studio
docker compose --env-file .env.prod -f compose.proxy.yaml up -d --build

cd ../chez-marcel
docker compose --env-file .env.prod -f compose.proxy.yaml up -d --build
```

Le proxy détecte les apps par leur alias réseau (`amui-app`, `chezmarcel-app`) et
route chaque domaine. Les certificats sont obtenus automatiquement au premier accès.

## Ajouter un projet

1. Ajouter un bloc dans `Caddyfile` (`{$NOUVEAU_DOMAIN} { reverse_proxy nouveau-app:80 }`)
   et la variable dans `.env`.
2. `docker compose up -d` (recharge le proxy).
3. Créer un `compose.proxy.yaml` dans le nouveau projet avec l'alias réseau `nouveau-app`.

## Notes

- Chaque projet garde sa propre base MySQL + ses volumes (préfixés par le nom du
  dossier : `amui-studio_db_data`, `chez-marcel_db_data`…), donc aucune collision.
- Lancer chaque projet depuis un dossier au nom distinct (le nom du dossier = nom
  du projet Compose, qui isole conteneurs/volumes/réseau interne).
- `compose.prod.yaml` (autonome, TLS géré par l'app, ports 80/443 publiés) reste
  disponible pour un déploiement mono-site sans proxy.

*[English version](README.md)*

# Infisical — gestionnaire de secrets centralisé (homelab self-hosted)

## Vue d'ensemble

Instance auto-hébergée d'[Infisical](https://infisical.com/) sur un Raspberry Pi 5 faisant office de homelab, destinée à devenir le gestionnaire de secrets centralisé pour l'ensemble des projets (à la place de fichiers `.env` épars par projet).

- **État** : déployé et opérationnel, compte administrateur créé.
- **Accès** : via un tunnel Tailscale privé, sur un port dédié — pas d'exposition publique.

## Architecture

Deux conteneurs, réseau Docker par défaut (pas de réseau custom, pas de reverse proxy pour l'instant) :

| Service | Image | Port hôte | Rôle |
|---|---|---|---|
| `infisical` | `infisical/infisical:latest` | exposé uniquement en interne au tunnel privé | Application principale |
| `infisical-redis` | `redis:7-alpine` | *(aucun, interne uniquement)* | Cache / queue interne |

- **Base de données** : PostgreSQL **mutualisé**, déjà en place sur l'hôte (pas de conteneur Postgres dédié), base et user applicatif dédiés.
  - Connexion via la passerelle du bridge Docker par défaut (pas de réseau custom, donc pas besoin d'`extra_hosts` / réseau dédié).
  - ⚠️ À vérifier : règle firewall autorisant le sous-réseau du bridge Docker par défaut vers le port Postgres, sinon la connexion timeout silencieusement.
- **Redis** : pas de port exposé côté hôte, joignable uniquement par `infisical` via le réseau interne du compose.
- **Volumes** : aucun volume déclaré — cohérent car tout l'état persistant (secrets, configuration) vit dans le Postgres externe ; Redis n'est qu'un cache, sa perte au redémarrage n'est pas un problème.

## Réseau / routage

- Actuellement accès direct via Tailscale uniquement, sans reverse proxy.
- Le compose contient un bloc commenté prêt à activer pour router plus tard via un reverse proxy (Traefik ou équivalent), avec un sous-domaine interne dédié.

## Variables d'environnement (`.env`, non commité — voir `.gitignore` et `.env.example`)

| Variable | Description |
|---|---|
| `DB_PASSWORD` | Mot de passe du user Postgres applicatif |
| `DB_HOST` | Adresse de connexion au Postgres mutualisé |
| `ENCRYPTION_KEY` | Clé de chiffrement Infisical (secrets au repos) — **critique, ne jamais perdre ni faire fuiter** |
| `AUTH_SECRET` | Secret de signature des sessions/JWT |
| `INFISICAL_SITE_URL` | URL d'accès à Infisical (adresse Tailscale), ex. `http://caesura.<tailnet>.ts.net:8090` |

Voir `.env.example` pour le gabarit à copier.

## Procédures courantes

- **Démarrage / mise à jour** : `docker compose up -d` (le tag `latest` sur l'image peut faire monter de version sans avertissement — à surveiller).
- **Logs** : `docker logs -f infisical`
- **Arrêt** : `docker compose down` (ne supprime pas les données, tout est dans le Postgres externe).

## Points de vigilance / TODO

1. **Sauvegarde** : `ENCRYPTION_KEY` et `AUTH_SECRET` n'ont, à ce jour, aucune copie de sauvegarde en dehors du `.env` local. Une perte de ce fichier rendrait les secrets stockés dans Postgres illisibles. À stocker dans un second endroit sûr.
2. **Tag `latest`** : épingler une version précise plutôt que `latest` pour éviter une mise à jour surprise en production.
3. **Reverse proxy** : décision à prendre selon l'avancement de la mutualisation des autres projets.
4. **Migration des secrets existants** : les autres projets utilisant encore des `.env` locaux ne sont pas encore migrés vers Infisical.

## Licence

[MIT](LICENSE)

# Éconoris — Proxy

Le reverse proxy Traefik du serveur : le seul point d'entrée sur les ports 80 et 443. `econoris-wrapper` le déploie avec le reste de l'écosystème.

Ce dépôt déclare tout ce qui est exposé : les routes, et ce qui leur est commun (HTTPS, certificats Let's Encrypt, options TLS, en-têtes de sécurité et limite de requêtes). Traefik n'a aucun accès à l'API Docker, et les autres dépôts n'ont aucun label Traefik.

# Table des matières

- [Éconoris — Proxy](#éconoris--proxy)
- [Table des matières](#table-des-matières)
- [Structure](#structure)
- [Mettre en service](#mettre-en-service)
- [Exposer un service](#exposer-un-service)
- [Limite de requêtes](#limite-de-requêtes)
- [Mettre à jour Traefik](#mettre-à-jour-traefik)
- [Sécurité](#sécurité)

# Structure

```
econoris-proxy/
├── docker-compose.yml          ← Traefik
├── .env.example                ← réglages de cette machine
└── dynamic/
    ├── routes.yml              ← tout ce qui est exposé : domaines et services
    ├── rate-limit.yml          ← limite de requêtes par IP
    ├── security-headers.yml    ← en-têtes appliqués à toutes les routes
    └── tls.yml                 ← options TLS de toutes les routes
```

# Mettre en service

Il n'y a rien à lancer depuis ce dépôt : tout passe par `econoris-wrapper`, dont le README décrit la mise en service.

Le réglage de cette machine se trouve dans `env/proxy/.env` du wrapper. `LETSENCRYPT_EMAIL` doit y être rempli.

Compose crée le réseau `traefik-net` avec IPv6. Sans IPv6 sur ce réseau, Docker relaierait les visiteurs IPv6 en IPv4 et leur adresse serait perdue : ils partageraient tous la même limite de requêtes. Les domaines ont des enregistrements `AAAA`, donc ce cas se produirait réellement.

# Exposer un service

1. Dans le `docker-compose.yml` du dépôt, le service rejoint son propre réseau interne :

    ```yaml
    services:
      econoris-vitrine:
        networks:
          - econoris-vitrine-net

    networks:
      econoris-vitrine-net:
        name: econoris-vitrine-net
        internal: true
    ```

2. Dans `docker-compose.override.yml` du wrapper, Traefik rejoint ce réseau :

    ```yaml
    services:
      traefik:
        networks:
          - econoris-vitrine-net
    ```

3. Dans `dynamic/routes.yml` de ce dépôt, déclarer son routeur et son service. Traefik joint le service par son nom :

    ```yaml
    http:
      routers:
        econoris-vitrine:
          rule: "Host(`econoris.fr`)"
          entryPoints:
            - websecure
          service: econoris-vitrine

      services:
        econoris-vitrine:
          loadBalancer:
            servers:
              - url: "http://econoris-vitrine:8080"
            healthCheck:
              path: /
              interval: 10s
              timeout: 5s
    ```

Il n'y a rien d'autre à déclarer : le certificat `le`, les en-têtes de sécurité et la limite de requêtes s'appliquent par défaut.

⚠️ **Ne pas écrire de clé `tls` dans un routeur.** Traefik n'applique le certificat par défaut qu'aux routes sans aucune configuration TLS : la route servirait un certificat invalide. Si la route doit préciser son TLS, utiliser `tls.certResolver: le`.

**Le service ne publie aucun port.** C'est la responsabilité de chaque dépôt : un port publié contourne Traefik, donc le HTTPS et la limite de requêtes, et Docker l'ouvre même si le pare-feu du serveur le bloque.

Si un service est arrêté, sa route reste déclarée : le visiteur reçoit une erreur `502` ou `503`.

# Limite de requêtes

Chaque adresse IP a droit à **30 requêtes par seconde en continu, avec des pics jusqu'à 150**. Au-delà, Traefik répond `429`. Le compteur est séparé pour chaque route. En IPv6, toutes les adresses d'un même `/64` comptent pour une seule.

Le réglage est dans `dynamic/rate-limit.yml`, où le calcul est expliqué. Une modification part au prochain `deploy.sh` du wrapper, une fois sur `main`.

Un routeur de `dynamic/routes.yml` peut ajouter une limite plus stricte, par exemple sur la connexion. Une limite plus large que celle-ci n'aurait aucun effet.

# Mettre à jour Traefik

L'image est figée sur une version et un digest, donc **les correctifs de sécurité ne s'installent pas seuls**. Traefik publie souvent : il faut suivre [ses releases](https://github.com/traefik/traefik/releases).

Pour mettre à jour, changer la version et le digest de `x-traefik-image` dans `docker-compose.yml`, publier sur `main`, puis lancer le `deploy.sh` du wrapper.

# Sécurité

- **Traefik n'a aucun accès à l'API Docker.** Le socket Docker n'est monté dans aucun conteneur. Si Traefik était compromis, il ne pourrait ni lire les variables, les journaux et les fichiers des autres conteneurs, ni agir sur eux.
- **Un conteneur ne peut pas se déclarer une route.** Seul `dynamic/routes.yml` expose un service : une image piégée ne peut ni s'exposer, ni détourner un domaine.
- **Traefik est le seul lien entre les services exposés.** Chacun a son réseau, sans accès à Internet si possible, et aucun ne peut en joindre un autre. Traefik, lui, les atteint tous : un service ne doit donc pas faire confiance à un appel sous prétexte qu'il vient de son réseau, et une base de données n'a jamais sa place sur un réseau de Traefik.
- **Le HSTS couvre les sous-domaines** : un navigateur qui a visité un domaine refusera le HTTP sur tous ses sous-domaines pendant un an. Tout nouveau sous-domaine doit donc être en HTTPS dès le départ.

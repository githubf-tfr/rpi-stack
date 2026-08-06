# rpi-stack

Bibliothèque générique de templates Docker Compose pour services auto-hébergés.
Ce repo ne contient **aucune valeur concrète, aucun secret** — uniquement des
templates (`${VAR}`) et des fichiers `.example`. Les valeurs réelles sont
apportées par le projet consommateur (ex. `rpi-nomade`).

> **Conventions figées.** Structure, nommage et plan d'adressage réseau
> (ci-dessous) sont des règles actées avec l'utilisateur — pas des choix
> libres. Tout assistant (LLM ou autre) qui s'en écarte, ou qui propose une
> alternative en créant un nouveau service, doit d'abord obtenir l'accord
> explicite de l'utilisateur (détail dans `CLAUDE.md`).

---

## ⚠️ RÈGLE IMPÉRATIVE — VALABLE DANS TOUT REPO CONSOMMATEUR EXTERNE ⚠️

**TOUTE STACK/CONFIG LAISSÉE INLINE** dans un repo consommateur (`rpi-nomade`, DMZ,
NAS, k3s, k3s pro...) **AU LIEU D'ÊTRE GÉNÉRALISÉE EN TEMPLATE ICI DOIT
OBLIGATOIREMENT S'ACCOMPAGNER DE LA CRÉATION D'UN TICKET GITHUB DANS `rpi-stack`**
(`githubf-tfr/rpi-stack`, `gh issue create --repo githubf-tfr/rpi-stack`).

- **Sans exception** : dette technique inline = ticket `rpi-stack`, dans le même lot
  de travail — jamais après coup, jamais « on verra plus tard ».
- **Avant de créer un ticket, chercher un doublon** (`gh issue list --repo
  githubf-tfr/rpi-stack --search "<mots-clés>" --state all`) — plusieurs repos
  consommateurs peuvent tomber sur le même besoin (ou un besoin voisin) :
  - Ticket **ouvert** correspondant → commenter dessus, pas de nouveau ticket (le
    second signal aide à prioriser).
  - Ticket **fermé**, besoin identique → rien à créer, juste consommer ce qui existe
    déjà (le template/la variable a été généralisé).
  - Ticket **fermé**, besoin voisin mais pas identique (cas non couvert) → nouveau
    ticket qui référence l'ancien (`Ref #N`), sans le rouvrir — l'ancien a été traité
    correctement pour son périmètre.
  - Rien trouvé → nouveau ticket, normalement.
- Le ticket référence le repo/fichier consommateur concerné et décrit le besoin resté
  inline et pourquoi (cf. les entrées `KANBAN.md` de ce repo pour le niveau de détail
  attendu).
- Les droits GitHub d'une session ouverte dans un repo consommateur sont les **mêmes**
  que ceux utilisés ici (identité du compte, pas du repo) : la création du ticket dans
  `rpi-stack` est possible depuis n'importe quel repo consommateur.

---

## Plan d'adressage réseau (Docker)

Chaque stack a son propre sous-réseau Docker dédié, toujours en `/24` (cf.
`CLAUDE.md`), alloué depuis un bloc `/16` fixe selon la catégorie du
service. Renumérotée une 2ème fois le 2026-07-23 (fusion des catégories
infrastructure/services communs en un seul dossier `socle/`, cf.
`CLAUDE.md` « Structure ») :

| Bloc `/16` | Catégorie |
|---|---|
| `10.0.0.0/16` | Réseau `proxy` partagé (routage, cf. ci-dessous — pas un bloc de `/24` par stack) |
| `10.1.0.0/16` | Socle technique (Portainer, ddclient, Traefik, monitoring...) — `10.1.255.0/24` réservé au secours physique (Ethernet du Pi), cf. tableau ci-dessous |
| `10.2.0.0/16` | Services métiers |
| `10.3.0.0/16` | LAN réels des sites, routés en site-à-site WireGuard sans NAT (cf. ci-dessous — pas un bloc de `/24` Docker par stack non plus) |

Allocations actuelles — **à tenir à jour à chaque nouveau service** :

| Stack | Catégorie | Subnet privé | Réseau/interface | IP sur `net_proxy` |
|---|---|---|---|---|
| `portainer` (serveur) | socle | `10.1.0.0/24` | `net_portainer` | — (routé via provider file, pas `net_proxy` — cf. ci-dessous) |
| `portainer` (agent) | socle | `10.1.0.0/24` (**partagé** avec le serveur, jamais colocalisés — cf. « Dérogation réseau » ci-dessous) | `net_portainer` | — (pas d'UI web, rien à router) |
| `ddclient` | socle | `10.1.1.0/24` | `net_ddclient` | — (pas d'interface web, rien à router) |
| `traefik` | socle | `10.1.2.0/24` | `net_traefik` | `10.0.1.2` (créateur du réseau) |
| `adguard` | socle | — (`network_mode: host`, pas de `/24` — cf. « Dérogation réseau » ci-dessous) | — | — (aucun réseau Docker) |
| `uptime-kuma` | socle | `10.1.3.0/24` | `net_uptime-kuma` | — (provider Docker de Traefik cassé/inutilisé, routé via provider file — cf. ci-dessous) — **pas encore dans le pipeline automatisé, cf. section dédiée** |
| `wireguard` | socle | `10.1.4.0/24` | `net_wireguard` | — (pas d'UI web, rien à router) |
| `wireguard` (tunnel `wg0`) | socle | `10.1.5.0/24` | — | — (**pas** un réseau Docker — adressage interne du tunnel WireGuard, géré dans `wg_confs/wg0.conf`, cf. section dédiée) |
| `— (réservé)` | socle | `10.1.255.0/24` | — | — (secours physique — port Ethernet du Pi, pas un stack Docker, **jamais à allouer**) |

Prochain `/24` libre : `10.1.6.0/24` (socle) ; `10.2.0.0/24` (services
métiers, bloc encore inutilisé). `adguard` ne consomme aucun `/24` (cf.
ci-dessous), la numérotation n'est donc pas affectée par son ajout. Idem
pour l'agent Portainer, qui réutilise le `/24` du serveur au lieu d'en
consommer un nouveau (cf. « Dérogation réseau » ci-dessous).
`10.1.255.0/24` (dernier `/24` du bloc socle) est réservé au secours
physique et **exclu du pool d'allocation** — ne jamais l'attribuer à une
nouvelle stack.

### Réseau `proxy` partagé (routage)

Amendement à la règle « un `/24` par stack » (cf. `CLAUDE.md`) pour les
stacks de routage type Traefik : en plus de son `/24` privé, ce genre de
stack **crée** un réseau Docker bridge **partagé** que les stacks backend
**locales** rejoignent **en plus** de leur propre `/24`, en le déclarant
`external: true` chez elles (auto-découverte par labels via le *provider*
Docker de Traefik). Un backend **délocalisé** (NAS, autre Pi, cluster...)
n'a besoin de rien de tout ça — juste une IP:port dans la config `dynamic/`.

- **Sous-réseau fixe** : `10.0.0.0/16` (plus laissé à l'attribution libre de
  Docker depuis le 2026-07-23).
- **Adressage** : `10.0.X.Y`, où `X` = code de catégorie (`1` socle, `2`
  métier — mêmes chiffres que les blocs `/16` privés) et `Y` = le 3ème
  octet du `/24` privé du service (sa position dans sa catégorie). Ex.
  `traefik` (`10.1.2.0/24`, 3ème octet `2`) → `10.0.1.2`. **Aucune autre
  stack ne le rejoint aujourd'hui** — `ddclient` n'a pas d'interface web
  (rien à router), `portainer` et `uptime-kuma` sont routés autrement (cf.
  « Pourquoi Portainer n'est pas sur `net_proxy` » ci-dessous — le provider
  Docker de Traefik est d'ailleurs cassé/inutilisé dans ce repo, pas
  seulement évité par choix) ; le mécanisme reste prêt pour un futur
  service qui en aurait vraiment besoin.
- **Point ouvert, pas encore tranché** : si une stack a besoin de plusieurs
  URLs Traefik distinctes (donc potentiellement plusieurs adresses sur ce
  réseau pour un seul service), le schéma à utiliser n'est pas décidé —
  piste envisagée mais non retenue non plus : un relais web local (reverse-
  proxy interne à la stack) pour ne présenter qu'une seule adresse à
  Traefik. À trancher au premier besoin réel.

| Réseau partagé | Créé par | Rejoint par |
|---|---|---|
| `net_proxy` | `traefik` | personne aujourd'hui — prêt pour une future stack locale qui voudrait être routée via Traefik avec découverte auto par labels (`external: true`) |

### Pourquoi Portainer n'est pas sur `net_proxy`

Décidé le 2026-07-23, après un premier essai (Portainer rejoignant
`net_proxy` + labels `traefik.*`) revenu en arrière — complexité pas
justifiée par le gain. Portainer veut du TLS Let's Encrypt, mais **pas**
via `net_proxy`/labels : il est déployé *avant* que `traefik` (et donc
`net_proxy`) n'existe, ce qui aurait imposé une dance en deux temps
(déployer Portainer, déployer Traefik, réappliquer le `compose.yaml` de
Portainer) rien que pour lui.

À la place, Traefik route vers Portainer via le **provider file** — exacte-
ment comme un backend délocalisé (cf. `socle/traefik/dynamic/portainer.yml.example`) :
une simple entrée `IP:port` pointant sur l'accès déjà publié de Portainer
(`PORTAINER_BIND_ADDR:9000`). Zéro changement dans son `compose.yaml`, zéro
label, zéro réseau partagé, zéro dépendance d'ordre de déploiement —
Portainer garde aussi son accès direct existant en fallback. Réutilisable
pour tout futur service qui veut du TLS Traefik sans les contraintes de
`net_proxy`.

### Ordre de déploiement

`net_proxy` n'existe qu'une fois `traefik` déployé. Ordre concret :
**Portainer** (bootstrap, toujours en premier) → **Traefik** (crée
`net_proxy`), déployé juste après, en 2ème. Compose refuse un réseau
`external` introuvable — pas de contournement possible sur l'ordre.

Aujourd'hui, aucune stack n'a besoin de rejoindre `net_proxy` (Portainer
utilise le provider file, cf. ci-dessus) — cette contrainte d'ordre ne
s'applique pour l'instant qu'à un futur service qui en aurait explicitement
besoin.

#### Rattacher une stack pré-existante à `net_proxy`

Pour exposer via Traefik une stack déjà déployée (locale), le contenu à
ajouter est **le même** quel que soit le mécanisme de déploiement — seule la
façon de le pousser change :

```yaml
services:
  monservice:
    networks:
      network: {}   # son /24 privé existant, inchangé
      proxy: {}     # ajout : rejoint le réseau partagé
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.monservice.rule=Host(`monservice.exemple.com`)"
      - "traefik.http.routers.monservice.entrypoints=websecure"
      - "traefik.http.routers.monservice.tls.certresolver=letsencrypt"

networks:
  network: {}       # inchangé
  proxy:
    external: true
    name: net_proxy # doit correspondre au nom cree par la stack traefik
```

- **Déploiement brut (`docker compose up -d`)** — éditer le `compose.yaml`
  de la stack ciblée sur la cible, puis relancer `docker compose up -d` dans
  son dossier. Compose connecte le conteneur en plus de son réseau existant,
  sans toucher aux volumes/données.
- **Déploiement via l'API Portainer** — même contenu YAML, mais mis à jour
  dans la Stack Portainer et redéployé via l'API (`PUT /api/stacks/{id}`,
  ce que fait le rôle `portainer-stack`) ; les variables passent par le
  champ *Environment variables* de la Stack plutôt que par un `.env` déposé
  à côté.
- **Dans les deux cas** : `net_proxy` doit déjà exister sur l'hôte (donc la
  stack `traefik` déjà déployée) avant l'update — sinon ça échoue en
  cherchant un réseau externe introuvable.

### Dérogation réseau : `network_mode: host` (AdGuard Home)

Décidé le 2026-07-24, à l'occasion de `socle/adguard/`. Deuxième dérogation
à la règle « chaque stack a son propre `/24` », différente de l'amendement
`net_proxy` ci-dessus : celui-là **ajoute** un réseau en plus du `/24`
privé, celui-ci **retire** le réseau Docker de la stack entièrement.

Le serveur DHCP d'AdGuard Home doit émettre en broadcast sur le segment L2
physique du réseau de l'AP (`rpi-nomade`) — un réseau bridge Docker isolé ne
le permet pas (cf. wiki officiel `AdguardTeam/AdGuardHome`, section Docker :
« If you want to use AdGuardHome's DHCP server, you should pass --network
host », Linux uniquement). Compose interdit par ailleurs de combiner
`network_mode: host` avec `networks:`/`ports:` sur le même service —
`socle/adguard/` n'a donc ni `/24` dédié, ni IP fixe, ni `net_xxx`,
contrairement à toutes les autres stacks de ce repo (pas de bloc
`networks:` ni de `ports:` dans son `compose.yaml`). Les ports réellement
écoutés par AdGuard Home (UI web, DNS 53, DHCP 67/udp si activé...) sont
ceux de l'hôte directement, configurés dans AdGuard Home lui-même
(assistant de première configuration sur l'UI web) — hors périmètre
Compose/`.env` de ce repo. Détail complet : `CLAUDE.md`, section
« Dérogation réseau : `network_mode: host` ».

Conséquence directe : `socle/adguard/` **ne rejoint pas non plus `net_proxy`**
(cf. « Réseau `proxy` partagé » plus haut). Pas un choix ni une simple
restriction de syntaxe Compose — une limitation réseau Linux réelle : en
`network_mode: host`, le conteneur est placé directement dans le namespace
réseau de l'hôte, il n'existe donc aucun namespace réseau conteneur sur
lequel attacher un second réseau Docker (bridge `net_proxy` ou autre). Si
AdGuard avait un jour besoin de TLS Traefik, ce serait via le même
mécanisme que Portainer/Uptime Kuma (provider file, IP:port de l'hôte —
cf. « Pourquoi Portainer n'est pas sur `net_proxy` » plus haut), jamais via
`net_proxy`.

### Dérogation réseau : sous-réseau partagé serveur/agent Portainer

Décidé le 2026-08-05, à l'occasion de `socle/portainer/agent/`. Troisième
dérogation à la règle « chaque stack a son propre `/24` », de nature
différente des deux précédentes : celles-ci changent la *forme* du réseau
d'une stack (réseau ajouté ou retiré) ; celle-ci fait que **deux stacks**
(`socle/portainer/serveur/` et `socle/portainer/agent/`) déclarent le
**même** `/24` (`10.1.0.0/24`) et le même nom de réseau (`net_portainer`)
dans leurs `.env` respectifs.

Un sous-réseau Docker n'est unique qu'à l'échelle d'un hôte (le démon
Docker de chaque Pi crée son propre bridge, sans lien avec ceux des autres
hôtes) — la règle « pas de collision » n'a donc de sens qu'*entre stacks
d'un même hôte*. Or serveur et agent Portainer ne tournent **jamais** sur
le même hôte par construction : l'agent sert justement à rattacher un
**autre** hôte (ex. `rpi-proxy`) à un Portainer serveur qui tourne
ailleurs — un serveur ne gérerait pas son propre hôte via son propre
agent. Rien n'empêche donc de réutiliser les mêmes valeurs par défaut dans
les deux `.example`, sans risque de collision réelle ni de gaspillage d'un
`/24` supplémentaire. Le consommateur reste libre de choisir des valeurs
différentes s'il préfère, comme pour toute variable de ce repo.

### Bloc `10.3.0.0/16` — LAN réels des sites (site-à-site WireGuard sans NAT)

Décidé le 2026-07-25, à l'occasion de la conception d'une future stack
WireGuard (pas encore construite dans ce repo — template à venir). Nature
différente des blocs `socle`/`métier` ci-dessus : **pas** des sous-réseaux
Docker `/24` par stack sur cet hôte, mais les **LAN physiques réels** de
chaque site distant relié en site-à-site **sans NAT** — renumérotés depuis
leur subnet d'origine (souvent choisi par la box/le routeur, ex.
`192.168.x.0/24`), pour garantir qu'ils ne collisionnent jamais entre eux.

- **Pourquoi une renumérotation** : un routage site-à-site sans NAT exige une
  adresse unique **globalement** (les deux sites doivent voir passer les
  vraies IP l'un de l'autre) — contrairement au tunnel WireGuard **NAT'é**
  des clients nomades/admin (road warrior), qui reste dans le pool `socle`
  (`10.1.X.0/24`, cf. futur `socle/wireguard/`) car ses IP ne sortent jamais
  du conteneur et n'ont donc besoin d'aucune unicité globale. Deux besoins
  différents, deux blocs différents.
- **Convention d'allocation** : un `/24` par site, tiré de ce bloc — le 3ème
  octet reprend si possible celui du subnet d'origine (mnémotechnique), ex.
  nomade `192.168.71.0/24` → `10.3.71.0/24`.
- **Migration physique** (renumérotation effective du LAN du site) **hors
  périmètre de ce repo** — pas du Docker/compose, à faire côté infra réseau
  du site concerné (ex. rôle dédié de `rpi-stage`, configuration DHCP/routeur
  de `rpi-nomade`). Ce repo se contente de réserver et documenter le bloc,
  pour que plusieurs sites gérés avec cet outillage ne choisissent jamais le
  même subnet indépendamment.

Sites (registre à tenir à jour à chaque site relié en site-à-site) :

| Site | Ancien subnet | Nouveau subnet (`10.3.0.0/16`) | Statut |
|---|---|---|---|
| `rpi-nomade` | `192.168.71.0/24` | `10.3.71.0/24` | migration prévue, pas encore faite |

---

## Stacks disponibles

### `socle/portainer/` — Portainer Business Edition (EE lts)

Deux stacks distinctes, jamais colocalisées sur le même hôte (cf.
« Dérogation réseau : sous-réseau partagé serveur/agent Portainer » plus
haut) :

- `socle/portainer/serveur/` — l'interface web de gestion Docker elle-même.
- `socle/portainer/agent/` — l'agent classique (pas Edge Agent) qui rattache
  un **autre** hôte (ex. `rpi-proxy`) à un Portainer serveur existant
  hébergé ailleurs. Différence agent classique / Edge Agent : direction de
  connexion (le serveur se connecte vers l'agent classique, en TLS mutuel
  sur le port 9001 ; l'Edge Agent fait l'inverse, pensé pour du NAT/pas
  d'IP joignable) — hors sujet ici, template Edge non fourni pour l'instant.

#### `socle/portainer/serveur/`

Licence gratuite jusqu'à 3 nœuds ; la version EE est utilisée car elle
partage le même codebase que CE et déverrouille les fonctionnalités sous
licence sans rien changer au reste.

Particularités du template :

- Mot de passe admin bcrypt pré-configuré au démarrage (`--admin-password-file`)
  — pas de fenêtre de 5 min, pas de setup token à recopier depuis les logs.
- Exposition sur une IP précise (`PORTAINER_BIND_ADDR`) plutôt que `0.0.0.0`
  — Docker bypasse `ufw`/`nftables` via DNAT, le binding est la seule parade.
- Réseau bridge dédié (`net_portainer`) avec IP fixe — adressage stable pour
  les règles pare-feu, interface prévisible dans `ip link`/tcpdump.
- Volume bindé sur une partition dédiée (`PORTAINER_DATA_DIR`) — les données
  restent hors de `/var/lib/docker`, réservé au moteur.
- **Pas de `healthcheck`** — l'image est buildée `FROM scratch` (aucun
  wget/curl/shell dedans) et Portainer n'a pas de sous-commande CLI dédiée ;
  un healthcheck HTTP classique est donc impossible sans changer l'image
  (`-alpine`), pas fait ici. Détail en commentaire dans `compose.yaml`.
- **Ne rejoint pas `net_proxy`** — pas de label `traefik.*`, pas de réseau
  partagé. Pour du TLS Let's Encrypt, Traefik route vers lui via le
  *provider file* (IP:port déjà publiée, cf. `socle/traefik/dynamic/portainer.yml.example`
  et « Pourquoi Portainer n'est pas sur `net_proxy` » plus haut) — l'accès
  direct `PORTAINER_BIND_ADDR:9000` reste utilisable en fallback.

Variables requises (voir `socle/portainer/serveur/portainer.env.example`) :

| Variable | Rôle |
|---|---|
| `PORTAINER_CONTAINER_NAME` | Nom du conteneur (`portainer`, un seul conteneur dans la stack) |
| `PORTAINER_BIND_ADDR` | IP d'écoute de l'UI (port 9000) |
| `PORTAINER_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `PORTAINER_NETWORK_IP` | IP fixe de Portainer dans ce sous-réseau (`.100`) |
| `PORTAINER_NETWORK_NAME` | Nom de réseau Docker (`net_portainer`) |
| `PORTAINER_NETWORK_IFACE` | Nom d'interface bridge (`net_portainer`, à garder identique par convention) |
| `PORTAINER_DATA_DIR` | Chemin hôte pour la persistance des données |

Secrets requis (voir `socle/portainer/serveur/secrets.example/`) :

| Fichier | Contenu |
|---|---|
| `secrets/portainer_admin_password` | Hash bcrypt du mot de passe admin |
| `secrets/license.env` | Clé de licence EE (`PORTAINER_LICENSE_KEY=…`) |

#### `socle/portainer/agent/`

Rattache l'hôte qui l'exécute à un Portainer serveur distant (autre hôte,
autre déploiement de `socle/portainer/serveur/`). Aucun secret : l'agent
n'a ni mot de passe ni licence propres, l'authentification se fait en TLS
mutuel négocié au moment du rattachement côté UI Portainer.

Particularités du template :

- Monte `/:/host` (racine du filesystem hôte) — active l'onglet « Host » de
  Portainer (specs matérielles, parcours du FS hôte). **Équivalent
  fonctionnel à un accès root sur l'hôte si ce conteneur est compromis** —
  décision explicite prise pour ce repo, à reconsidérer par le consommateur
  s'il préfère un agent moins privilégié (fonctionnalités de gestion des
  conteneurs/images/réseaux/volumes/stacks toujours disponibles sans ce
  mount).
- Monte aussi `/var/lib/docker/volumes` (parcours du contenu des volumes
  depuis l'UI) et le socket Docker, comme tout agent Portainer standard.
- **Pas de `healthcheck`** — l'image embarque un binaire dédié (`healthy`),
  mais il ne reflète que la connectivité d'un **Edge** Agent à son serveur
  (`edge/client/build.go` dans le code source `portainer/agent` — jamais
  appelé par l'agent classique utilisé ici, où c'est le serveur qui se
  connecte vers l'agent, pas l'inverse). L'utiliser produirait un faux
  négatif permanent. Détail en commentaire dans `compose.yaml`.
- Réutilise le même sous-réseau que `socle/portainer/serveur/`
  (`10.1.0.0/24`, `net_portainer`) — cf. « Dérogation réseau : sous-réseau
  partagé serveur/agent Portainer » plus haut.

Variables requises (voir `socle/portainer/agent/portainer-agent.env.example`) :

| Variable | Rôle |
|---|---|
| `PORTAINER_AGENT_CONTAINER_NAME` | Nom du conteneur (`portainer_agent`, un seul conteneur dans la stack) |
| `PORTAINER_AGENT_BIND_ADDR` | IP d'écoute de l'API agent (port 9001), sondée par le serveur distant |
| `PORTAINER_AGENT_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `PORTAINER_AGENT_NETWORK_IP` | IP fixe de l'agent dans ce sous-réseau (`.100`) |
| `PORTAINER_AGENT_NETWORK_NAME` | Nom de réseau Docker (`net_portainer`) |
| `PORTAINER_AGENT_NETWORK_IFACE` | Nom d'interface bridge (`net_portainer`, à garder identique par convention) |

---

### `socle/ddclient/` — DNS dynamique (linuxserver.io)

Met à jour un enregistrement DNS (ex. OVH DynHost) avec l'IP publique de
l'hôte. Image multi-arch, `ddclient` ≥ 3.10.0 (`protocol=ovh` natif).

Particularités du template :

- Le fichier de config applicatif complet (`ddclient.conf`,
  protocole/identifiants/domaine) n'est pas découpable en `${VAR}` Compose
  — voir `ddclient.conf.example` et la section « Cas particulier » du
  `CLAUDE.md`.
- **Pas de `healthcheck`** — ddclient n'écoute aucun port, ne sert aucune
  page web, rien à sonder depuis l'extérieur du process. Détail en
  commentaire dans `compose.yaml`.

Variables requises (voir `socle/ddclient/ddclient.env.example`) :

| Variable | Rôle |
|---|---|
| `DDCLIENT_CONTAINER_NAME` | Nom du conteneur (`ddclient`, un seul conteneur dans la stack) |
| `DDCLIENT_PUID` / `DDCLIENT_PGID` / `DDCLIENT_TZ` | Utilisateur hôte et fuseau horaire (image linuxserver.io) |
| `DDCLIENT_CONFIG_PATH` | Chemin hôte vers le `ddclient.conf` réel |
| `DDCLIENT_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `DDCLIENT_NETWORK_IP` | IP fixe de ddclient dans ce sous-réseau (`.100`) |
| `DDCLIENT_NETWORK_NAME` | Nom de réseau Docker (`net_ddclient`) |
| `DDCLIENT_NETWORK_IFACE` | Nom d'interface bridge (`net_ddclient`, à garder identique par convention) |

---

### `socle/traefik/` — reverse proxy + TLS (socle technique)

Expose les futurs services métiers derrière TLS (ACME/Let's Encrypt). Fait
partie du socle technique au même titre que Portainer/ddclient — sert
d'autres stacks plutôt que d'être un service métier lui-même.

Particularités du template :

- Config statique (`traefik.yml`) et dynamique (`dynamic/*.yml`) en fichiers
  complets, pas en `${VAR}` — même logique que `ddclient.conf` (voir « Cas
  particulier » du `CLAUDE.md`).
- Deux réseaux Docker : son `/24` privé classique, **et** `net_proxy`, réseau
  partagé qu'il crée pour router les stacks locales (voir « Réseau `proxy`
  partagé » ci-dessus) — pas encore rejoint par aucune stack aujourd'hui. Un
  backend délocalisé (NAS, autre Pi...) **ou même un service local déjà
  publié sur l'hôte** (ex. Portainer, cf. `dynamic/portainer.yml.example`)
  n'a besoin d'aucun des deux réseaux — juste une entrée `dynamic/*.yml`
  avec une IP:port.
- `exposedByDefault: false` — un conteneur n'est routé que s'il porte
  `traefik.enable=true` en label, jamais de découverte automatique.
- Dashboard jamais exposé nu (`api.insecure: false`), routé via un middleware
  `basicAuth` (`dynamic/dashboard.yml.example`).
- `docker.sock` monté en lecture seule pour l'auto-découverte — accès
  équivalent root sur l'hôte, même arbitrage déjà accepté pour Portainer.
- TLS : challenge HTTP-01 par défaut (port 80 joignable) ; alternative DNS-01
  via OVH documentée en commentaire dans `traefik.yml.example` (pas de port
  80 exposé, certs wildcard, mais identifiants API OVH différents de ceux de
  `ddclient`). Identifiants injectés en `environment:` (`TRAEFIK_OVH_*` →
  `OVH_*` côté conteneur, imposés par le provider "ovh" de lego), pas
  `env_file:` : un déploiement via l'API Portainer (`portainer-stack`)
  exécute le compose depuis le conteneur Portainer lui-même, sans visibilité
  sur l'arborescence hôte — `env_file:` y échoue même en chemin absolu,
  contrairement aux `volumes:` (bind-mounts, exécutés côté démon Docker de
  l'hôte, qui y a accès).
- **`healthcheck` via `traefik healthcheck --ping`** — sous-commande CLI
  native (pas un ping ICMP) : GET HTTP réel sur `/ping`, servi par
  l'entrypoint interne `traefik` (port 8080, non publié vers l'hôte).
  Marche sans wget/curl dans l'image ; nécessite `ping: {}` en config
  statique (déjà dans `traefik.yml.example`).

Variables requises (voir `socle/traefik/traefik.env.example`) :

| Variable | Rôle |
|---|---|
| `TRAEFIK_CONTAINER_NAME` | Nom du conteneur (`traefik`, un seul conteneur dans la stack) |
| `TRAEFIK_BIND_ADDR` | IP d'écoute des ports 80/443 |
| `TRAEFIK_HTTP_PORT` / `TRAEFIK_HTTPS_PORT` | Ports hôte publiés vers 80/443 |
| `TRAEFIK_STATIC_CONFIG_PATH` | Chemin hôte vers `traefik.yml` réel |
| `TRAEFIK_DYNAMIC_CONFIG_DIR` | Dossier hôte vers les `dynamic/*.yml` réels |
| `TRAEFIK_ACME_DIR` | Dossier hôte des certificats ACME (`acme.json` à pré-créer en `chmod 600`) |
| `TRAEFIK_OVH_ENDPOINT` / `TRAEFIK_OVH_APPLICATION_KEY` / `TRAEFIK_OVH_APPLICATION_SECRET` / `TRAEFIK_OVH_CONSUMER_KEY` | Identifiants API OVH pour le provider DNS-01 "ovh" de lego (`OVH_*` côté conteneur) — DIFFÉRENTS des identifiants DynHost de `ddclient` ; sans effet si `dnsChallenge` n'est pas activé dans `traefik.yml` |
| `TRAEFIK_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `TRAEFIK_NETWORK_IP` | IP fixe de Traefik dans ce sous-réseau (`.100`) |
| `TRAEFIK_NETWORK_NAME` | Nom de réseau Docker (`net_traefik`) |
| `TRAEFIK_NETWORK_IFACE` | Nom d'interface bridge (`net_traefik`, à garder identique par convention) |
| `TRAEFIK_PROXY_NETWORK_NAME` | Nom du réseau partagé de routage (`net_proxy`), créé par cette stack |
| `TRAEFIK_PROXY_NETWORK_SUBNET` | Sous-réseau fixe du réseau partagé (`10.0.0.0/16`) |
| `TRAEFIK_PROXY_IP` | IP fixe de Traefik lui-même sur `net_proxy` (`10.0.2.0`, formalisme `10.0.X.Y`) |

Pas de secrets — pas de `secrets.example/` dans ce dossier (les identifiants
API OVH sont des variables `${VAR}` classiques comme le reste de
`traefik.env.example`, cf. tableau ci-dessus ; `env_file:` a été écarté
justement pour rester compatible avec un déploiement via l'API Portainer).

---

### `socle/adguard/` — AdGuard Home (DNS + DHCP)

Résolveur DNS (filtrage pub/tracking) et, en option, serveur DHCP pour les
clients du réseau de l'AP `rpi-nomade`. Image officielle
`adguard/adguardhome`.

Particularités du template :

- **`network_mode: host`, pas de `/24` dédié** — dérogation au plan
  d'adressage réseau de ce repo (cf. « Dérogation réseau :
  `network_mode: host` » plus haut) : le serveur DHCP doit émettre en
  broadcast sur le segment L2 physique, ce qu'un bridge Docker isolé ne
  permet pas. Ni `networks:`, ni `ports:`, ni IP fixe dans `compose.yaml` —
  incompatibles avec `network_mode: host` en Compose.
- `cap_add: NET_ADMIN` — nécessaire au serveur DHCP (accès bas niveau à
  l'interface réseau) ; repris du compose historique côté `rpi-nomade`
  (`services/adguard/`, antérieur à la scission avec ce repo), qui
  fonctionnait en production avec ce cap — non listé comme requis par le
  wiki officiel AdGuard Home, gardé par prudence plutôt que retiré sans
  preuve du contraire.
- **Pas de `healthcheck`** — le `HEALTHCHECK` intégré à l'image officielle a
  été retiré en amont en v0.107.34 (trop de faux positifs, cf. discussion
  GitHub `AdguardTeam/AdGuardHome#5939`) et l'image finale (`alpine:latest`
  + seulement `ca-certificates`) n'embarque ni `wget` ni `curl` pour en
  refaire un côté `rpi-stack`. Détail en commentaire dans `compose.yaml`.
- Deux volumes nommés backés par bind (même pattern que `portainer_data`) :
  `work/` (état runtime — cache DNS, statistiques, logs de requêtes) et
  `conf/` (configuration, `AdGuardHome.yaml`). **Pas de `.example` pour
  `AdGuardHome.yaml`** — contrairement à `ddclient.conf`/`traefik.yml`, ce
  fichier n'est pas fourni par ce repo : AdGuard Home le génère lui-même via
  l'assistant de première configuration (UI web, port `3000` par défaut) ;
  le consommateur le gère ensuite directement dans `ADGUARD_CONF_DIR`.

Variables requises (voir `socle/adguard/adguard.env.example`) :

| Variable | Rôle |
|---|---|
| `ADGUARD_CONTAINER_NAME` | Nom du conteneur (`adguard`, un seul conteneur dans la stack) |
| `ADGUARD_WORK_DIR` | Chemin hôte pour l'état runtime (cache DNS, statistiques, logs) |
| `ADGUARD_CONF_DIR` | Chemin hôte pour la configuration (`AdGuardHome.yaml`, généré par AdGuard Home lui-même) |

Pas de secrets — pas de `secrets.example/` dans ce dossier (mot de passe
admin défini via l'assistant de première configuration, stocké haché dans
`AdGuardHome.yaml` sous `ADGUARD_CONF_DIR`).

---

### `socle/uptime-kuma/` — Uptime Kuma (monitoring de disponibilité)

> **⚠️ Pas (encore) intégré au pipeline de déploiement automatisé.** Décidé
> le 2026-07-25 : le template est conservé dans ce repo (structure/
> conventions déjà validées), mais son déploiement **automatisé/reproductible**
> côté `rpi-nomade` (rôle `rpi-stage`, sans passer par l'assistant web) est
> **en attente** tant que le premier démarrage sans assistant web (cf. plus
> bas) n'a pas une solution fiable — l'approche identifiée s'appuie sur une
> API interne d'Uptime Kuma non documentée et non garantie stable (cf.
> « Premier démarrage » ci-dessous), jugée pas assez mûre pour un
> déploiement automatisé en l'état. **Reste utilisable dès aujourd'hui pour
> une infra stable/durable** (pas un déploiement destiné à être détruit et
> rejoué régulièrement) : dans ce cas, passer une seule fois par l'assistant
> web de première configuration au déploiement initial est acceptable — ce
> n'est que la reproductibilité/l'automatisation intégrale qui est bloquée.
> À réactiver dans le pipeline automatisé une fois cette amélioration faite
> côté `rpi-stage` (ou si Uptime Kuma ajoute lui-même un mécanisme officiel —
> variable d'environnement, endpoint REST).

Monitoring de disponibilité (surveille Traefik/Portainer/AdGuard, et à
terme les services métiers). Image officielle `louislam/uptime-kuma`.

Particularités du template :

- **Déployé via l'API Portainer** (`portainer-stack`), comme
  `ddclient`/`adguard`/`traefik` — pas `docker-compose-stack`. Conséquence
  directe (même incident que Traefik, cf. commit dédié) : **zéro
  `env_file:`, zéro chemin de fichier référencé** dans `compose.yaml` — un
  déploiement par API exécute le compose depuis le conteneur Portainer
  lui-même, sans visibilité sur l'arborescence hôte.
- **Pas de `healthcheck` redéclaré** — l'image officielle en fournit déjà un
  nativement (binaire compilé `extra/healthcheck`, `HEALTHCHECK` intégré au
  `Dockerfile` upstream, `60s`/`30s`/`180s`/5 retries) : redondant, et moins
  bon, de le redéclarer ici.
- **Pas de `network_mode: host`** — contrairement à AdGuard, aucun besoin
  L2/broadcast ; `/24` dédié classique.
- **Pas de `net_proxy`** — le provider Docker de Traefik est cassé/inutilisé
  dans ce repo (incompatibilité de version), routage via provider file, même
  schéma que Portainer (cf. `socle/traefik/dynamic/uptime-kuma.yml.example`
  et « Pourquoi Portainer n'est pas sur `net_proxy` » plus haut).
- Un seul volume nommé backé par bind (même pattern que `portainer_data`) :
  base SQLite interne + captures d'écran de monitors, dans `/app/data`.

Variables requises (voir `socle/uptime-kuma/uptime-kuma.env.example`) :

| Variable | Rôle |
|---|---|
| `UPTIME_KUMA_CONTAINER_NAME` | Nom du conteneur (`uptime-kuma`, un seul conteneur dans la stack) |
| `UPTIME_KUMA_BIND_ADDR` | IP d'écoute du port 3001 (UI web) |
| `UPTIME_KUMA_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `UPTIME_KUMA_NETWORK_IP` | IP fixe d'Uptime Kuma dans ce sous-réseau (`.100`) |
| `UPTIME_KUMA_NETWORK_NAME` / `UPTIME_KUMA_NETWORK_IFACE` | Nom de réseau Docker et nom d'interface bridge (`net_uptime-kuma`, 15 caractères — pile la limite Linux `IFNAMSIZ`, ne pas allonger) |
| `UPTIME_KUMA_DATA_DIR` | Dossier hôte des données persistantes (base SQLite, captures d'écran) |

Pas de secrets — pas de `secrets.example/` dans ce dossier.

**Premier démarrage — pas d'automatisation possible depuis ce repo.**
Contrainte apprise le 2026-07-25 (cf. `KANBAN.md` de `rpi-nomade`) : aucune
exception à la règle « pas d'assistant web de première configuration » (même
exigence que Portainer/`--admin-password-file` et AdGuard/API de contrôle).
Vérifié contre le code source réel du dépôt `louislam/uptime-kuma` (pas une
doc résumée) : Uptime Kuma n'offre **ni variable d'environnement** (demande
de feature ouverte et non implémentée, cf.
[issue #4277](https://github.com/louislam/uptime-kuma/issues/4277)) **ni
endpoint REST** pour créer le premier compte admin — uniquement un
événement **Socket.IO** `setup(username, password, callback)`
(`server/server.js`). Deux pistes existent pour l'automatiser (client
Socket.IO scripté, ou pré-remplissage direct de la base SQLite avec un hash
bcrypt — même famille que le mot de passe admin Portainer), mais les deux
impliquent du **code**, hors périmètre zéro-code de ce repo : à résoudre
dans le rôle `rpi-stage` correspondant, pas ici. Ce template `compose.yaml`
n'a donc **aucun levier** pour ce problème (pas de `${VAR}` possible, Uptime
Kuma n'exposant aucune surface de configuration dessus) — il est inchangé
quelle que soit la solution retenue côté `rpi-stage`.

---

### `socle/wireguard/` — VPN (WireGuard)

VPN — couvre 3 profils de peers dans **une seule config** (WireGuard ne
distingue pas ces cas au niveau protocole, juste des `AllowedIPs`
différents par peer, cf. `wg_confs/wg0.conf.example`) :

1. **Client nomade classique** — accède au LAN du site (pas encore
   utilisé).
2. **Client admin** (1-2 clients) — accède en plus au réseau `socle`
   Docker (`10.1.0.0/16` en `AllowedIPs`).
3. **Site-à-site** — peer distant avec `Endpoint` fixe, tout son LAN réel
   (tiré du bloc `10.3.0.0/16`, cf. ci-dessus) en `AllowedIPs`.

Image `lscr.io/linuxserver/wireguard`.

Particularités du template :

- **Déployé via l'API Portainer** (`portainer-stack`), comme
  `ddclient`/`traefik`/`adguard`/`uptime-kuma` — pas `docker-compose-stack`.
  Zéro `env_file:`, zéro chemin de fichier référencé dans `compose.yaml`
  (cf. convention actée dans `CLAUDE.md`).
- **Réseau bridge classique, pas `network_mode: host`** — vérifié contre le
  README officiel `linuxserver/docker-wireguard` : contrairement à AdGuard
  (broadcast L2 pour le DHCP), WireGuard n'a besoin que de `cap_add:
  NET_ADMIN` + `sysctls: net.ipv4.conf.all.src_valid_mark=1` (documenté
  requis par l'image) + `net.ipv4.ip_forward=1` (pas documenté par l'image,
  mais indispensable pour relayer le trafic déchiffré vers le LAN).
- **Mode "custom" de l'image** — pas de variables `PEERS`/`SERVERURL`/...
  (génération automatique de peers), le consommateur dépose lui-même
  `wg_confs/wg0.conf` (cf. `.example`) : zéro génération de clé/peer par ce
  repo (zéro-code).
- **Pas de `healthcheck`** — l'image ne fournit aucun `HEALTHCHECK` natif ;
  le seul outil interne (`wg show wg0`) ne confirme que la présence de la
  config d'interface, pas que le tunnel/le routage fonctionnent réellement
  — pas assez fiable pour remplacer une vraie sonde (même prudence que pour
  AdGuard).
- **NAT (MASQUERADE) par défaut pour les 3 profils**, y compris le
  site-à-site — géré dans `PostUp`/`PreDown` de `wg0.conf`, pas au niveau
  Docker. Choix délibéré : marche sans aucune config côté routeur physique
  des sites, au prix de masquer les IP source réelles pour le site-à-site.
  Alternative "propre" (sans NAT, IP réelles visibles des deux côtés)
  documentée en commentaire dans `wg_confs/wg0.conf.example` mais **pas le
  défaut** — demande en plus une route statique sur le routeur physique de
  chaque site et de désactiver le NAT de bridge Docker
  (`com.docker.network.bridge.enable_ip_masquerade: false`), hors périmètre
  d'un template générique.
- Un seul volume nommé backé par bind (même pattern que `portainer_data`) :
  `/config` (contient `wg_confs/wg0.conf`, clés incluses).

Variables requises (voir `socle/wireguard/wireguard.env.example`) :

| Variable | Rôle |
|---|---|
| `WIREGUARD_CONTAINER_NAME` | Nom du conteneur (`wireguard`, un seul conteneur dans la stack) |
| `WIREGUARD_PUID` / `WIREGUARD_PGID` / `WIREGUARD_TZ` | Image linuxserver.io : utilisateur hôte propriétaire des fichiers montés, fuseau horaire des logs |
| `WIREGUARD_BIND_ADDR` | IP d'écoute du port UDP 51820 |
| `WIREGUARD_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` — **pas** l'adressage du tunnel WireGuard (cf. ci-dessous) |
| `WIREGUARD_NETWORK_IP` | IP fixe du conteneur dans ce sous-réseau (`.100`) |
| `WIREGUARD_NETWORK_NAME` / `WIREGUARD_NETWORK_IFACE` | Nom de réseau Docker et nom d'interface bridge (`net_wireguard`) |
| `WIREGUARD_CONFIG_DIR` | Dossier hôte des données persistantes — **doit contenir `wg_confs/wg0.conf`** avant le premier déploiement |

Pas de secrets `secrets.example/` séparés — les clés WireGuard vivent
directement dans `wg_confs/wg0.conf` (fichier de config applicative
complète, même cas que `ddclient.conf`/`traefik.yml`, cf. `CLAUDE.md` « Cas
particulier »), jamais dans un `.env`.

**Adressage du tunnel WireGuard — distinct du `/24` Docker ci-dessus.**
`10.1.5.0/24` (6ème `/24` du bloc socle, réservé mais **jamais** passé en
`${VAR}` Compose) : c'est l'espace `Address=`/`AllowedIPs` interne au
tunnel `wg0` lui-même, géré entièrement dans `wg_confs/wg0.conf` — NAT'é
donc jamais visible en dehors du conteneur, pas besoin d'unicité globale
(contrairement au bloc `10.3.0.0/16` utilisé pour le site-à-site, cf.
ci-dessus). Convention d'adressage WireGuard habituelle ici (`.1` =
serveur), pas celle de rpi-stack (`.100`).

---

## Déploiement

Ce repo ne contient que des templates `compose.yaml`, indépendants du
mécanisme utilisé pour les lancer — c'est au consommateur de choisir,
service par service : `docker compose up -d` brut, ou poussé via l'API
Portainer (rôle Ansible `portainer-stack`). Portainer lui-même est un cas
particulier fonctionnel : il ne peut pas se déployer via sa propre API
(il n'existe pas encore au moment de son propre déploiement), donc toujours
lancé en `docker compose up -d` brut côté consommateur.

---

## Comment consommer ce repo

Même mécanique que `rpi-format` / `rpi-cis` / `rpi-stage` : le projet
consommateur détecte d'abord un dossier/lien local, puis un repo frère
(`../../outillage/rpi-stack`), sinon clone au commit épinglé.

Le consommateur :

1. Lit le `compose.yaml` d'un service **tel quel** (ex. `lookup('file', …)` Ansible).
2. Fournit un fichier `.env` réel adapté du `.example` — déposé à côté du
   `compose.yaml` sur la cible, jamais committé ici.
3. Fournit ses propres secrets dans `secrets/` — jamais `secrets.example/` d'ici,
   toujours gitignorés côté consommateur.

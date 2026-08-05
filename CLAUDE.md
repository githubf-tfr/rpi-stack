# CLAUDE.md — rpi-stack

**Bibliothèque générique de templates Docker Compose** pour services auto-hébergés.
Joue pour les *stacks Docker* le rôle que `rpi-format`/`rpi-cis`/`rpi-stage` jouent
pour l'imaging/le durcissement/le provisioning : un outil **générique et
réutilisable**, consommé de l'extérieur par un projet concret (ex. `rpi-nomade`) qui
apporte, lui, les valeurs et les secrets de son déploiement.

## Règles d'or

- **Zéro code, zéro secret, zéro valeur concrète.** Uniquement des `compose.yaml`
  (syntaxe Compose native `${VAR}`/`${VAR:-défaut}`, jamais de valeur en dur) et des
  `*.example` (modèles vides ou factices). Aucune logique de déploiement (pas de
  script, pas de rôle Ansible) — c'est le rôle du consommateur.
- **Conventions figées.** La structure, le nommage (variables, conteneurs, volumes,
  réseaux) et le plan d'adressage sont des décisions actées avec l'utilisateur, pas
  des choix libres. **Obtiens son accord explicite avant de t'en écarter** — y
  compris pour proposer une alternative « meilleure » de ta propre initiative.

## Structure

`<catégorie>/<service>/` — 1 dossier = 1 service. **2 catégories seulement**, les
mêmes que les blocs `/16` du plan d'adressage : `socle/` (socle technique) et
`metier/` (services métiers, vide pour l'instant). La seule distinction qui compte :
fondation technique vs. service métier réel.

⚠️ Cet axe catégorise par **rôle du service**, pas par *mécanisme de déploiement* —
un découpage `compose/` brut vs `stacks/` API Portainer a été explicitement écarté,
ce choix reste entièrement au consommateur.

```
socle/ddclient/
├── compose.yaml              # template, ${VAR} uniquement
├── ddclient.env.example      # modèle des valeurs non secrètes
└── ddclient.conf.example     # config applicative complète, non découpable en ${VAR}
```

Le dossier `exemple/` (référence à 2 conteneurs serveur + bdd illustrant toutes les
conventions ci-dessous) vit **à la racine, hors catégorie** : n'étant pas un service
réellement déployé, il n'a sa place ni dans `socle/` ni dans `metier/`.

**Exception** : `socle/portainer/` contient deux sous-dossiers (`serveur/`, `agent/`),
chacun une stack Compose complète et indépendante (1 sous-dossier = 1 stack, la règle
« 1 dossier = 1 service » s'applique donc toujours, juste un niveau plus bas). Décidé
le 2026-08-05 : serveur et agent sont deux facettes du même service fonctionnel
(Portainer), jamais déployées sur le même hôte — cf. `README.md`, « Dérogation
réseau : sous-réseau partagé serveur/agent Portainer » pour le détail et le
raisonnement. Pas un précédent généralisable à d'autres services sans accord explicite
préalable.

Catalogue détaillé des stacks (variables et secrets requis par service) :
`README.md`, section « Stacks disponibles ».

## Conventions des `compose.yaml`

Applicables à **tout** nouveau template.

- **Nommage des variables** — préfixe = nom du service en MAJUSCULES
  (`PORTAINER_BIND_ADDR`, `DDCLIENT_CONFIG_PATH`…), pour éviter les collisions si
  plusieurs `.env` sont un jour concaténés côté consommateur.
- **Réseau en `/24`** — chaque stack a son propre sous-réseau Docker dédié, alloué
  depuis le bloc `/16` de sa catégorie (`10.0.0.0/16` = réseau `proxy` partagé,
  `10.1.0.0/16` = socle, `10.2.0.0/16` = métier). Documenter le subnet en commentaire
  à côté de sa variable. Registre des `/24` alloués dans `README.md` (« Plan
  d'adressage réseau ») — **à tenir à jour à chaque nouveau service**.
  **Réservé, jamais à allouer :** `10.1.255.0/24` (secours physique, port Ethernet du
  Pi — pas un stack Docker).
- **IP fixe par conteneur, en variable** — jamais de dépendance à la DNS interne
  Docker. Convention : `.100` = serveur/web, `.10` = bdd (à rappeler en commentaire à
  côté de chaque `ipv4_address`).
- **Nom de réseau et nom d'interface = deux variables distinctes** — formalisme
  `net_xxx` pour les deux (`name:` du réseau et
  `driver_opts.com.docker.network.bridge.name`). À garder identiques par convention,
  mais séparées : le consommateur doit pouvoir les découpler.
- **Nom des volumes** — formalisme `nomdelastack_xxx` (ex. `portainer_data`),
  documenté en commentaire ; en dur dans le compose (syntaxe Compose), pas en `${VAR}`.
- **Nom de conteneur, en variable** — `container_name` toujours en `${VAR}`,
  formalisme `nomdelastack_xxx`. Exception : stack à conteneur unique → pas de
  suffixe (`PORTAINER_CONTAINER_NAME=portainer`).
- **`healthcheck` seulement quand c'est réellement possible** — quand le conteneur
  offre un moyen fiable de se sonder *depuis l'intérieur de lui-même* (ex. `traefik
  healthcheck --ping`). **N'en ajoute pas un artificiel** juste pour en avoir un : si
  l'image ne fournit aucun outil exploitable (Portainer, image `scratch`) ou si le
  service n'a rien à sonder (ddclient, aucun port), documente l'absence en
  commentaire dans le `compose.yaml` avec la **raison technique précise** plutôt
  qu'une supposition. Pas de check maison en shell/log-parsing (ce serait du code).
- **Zéro `env_file:`, zéro chemin de fichier référencé** — dès qu'une stack est
  susceptible d'être déployée via l'**API Portainer**, `env_file:` échoue (même en
  chemin absolu) : le moteur Compose interne de Portainer s'exécute depuis son propre
  conteneur, sans visibilité sur l'arborescence hôte — contrairement aux `volumes:`,
  résolus par le démon Docker de l'hôte. Toute valeur, même sensible, passe donc en
  `${VAR}`/`environment:`. Concerne tout sauf Portainer lui-même.

### Config applicative non découpable en `${VAR}`

Certains services (ex. `ddclient`, `traefik`, `wireguard`) attendent un **fichier de
config applicatif complet** — Compose ne sait templater que son propre
`compose.yaml`, jamais le contenu d'un fichier monté dans le conteneur. Dans ce cas :
le fichier entier reste un `*.example` ici (structure + valeurs factices), le
`compose.yaml` le monte via un chemin `${VAR}` (ex. `${DDCLIENT_CONFIG_PATH}`), et
**le consommateur fournit le fichier réel**. Pas de découpage plus fin sans ajouter
un mécanisme de templating (entrypoint, `envsubst`…) — volontairement évité (zéro
code).

### Dérogations à la règle du `/24` par stack

Trois existent, **documentées en détail dans `README.md`** (ne pas les dupliquer
ici, il en est la source) ; les rappels utiles quand on écrit un template :

- **Réseau `net_proxy` partagé** (`10.0.0.0/16`, créé par `traefik`) — *ajoute* un
  réseau en plus du `/24` privé. Adressage `10.0.X.Y` (X = code catégorie, Y = 3ème
  octet du `/24` privé). Ne s'applique **qu'aux stacks qui en ont explicitement
  besoin** ; la règle par défaut reste un `/24` dédié, point. La stack qui *crée* le
  réseau le fait sans `external: true` (elle en est le bootstrap) ; celles qui le
  rejoindraient le déclarent `external: true` avec le nom en dur.
- **`network_mode: host`** (AdGuard Home, pour le broadcast DHCP L2) — *retire*
  entièrement le réseau Docker : ni `/24`, ni IP fixe, ni `net_xxx`, ni bloc
  `networks:`, ni `ports:` (Compose interdit de les combiner avec `network_mode`).
  Le dossier reste un dossier de service normal ; la dérogation se documente en
  commentaire en tête de son `compose.yaml`.
- **Sous-réseau partagé serveur/agent Portainer** (`socle/portainer/serveur/` et
  `socle/portainer/agent/`, toutes deux sur `10.1.0.0/24`/`net_portainer`) — *deux
  stacks* déclarent le même `/24` au lieu d'un `/24` chacune, car elles ne tournent
  jamais sur le même hôte (un sous-réseau Docker n'est unique qu'à l'échelle d'un
  hôte).

Le bloc `10.3.0.0/16` est réservé aux **LAN physiques réels** des sites reliés en
site-à-site WireGuard sans NAT — ce n'est pas un pool de `/24` Docker. Détail et
registre des sites : `README.md`.

## Comment un projet consomme ce repo

Même mécanisme que `rpi-format`/`rpi-cis`/`rpi-stage` : détection d'un repo local
(dossier/lien à la racine du consommateur) puis d'un repo frère
(`../../outillage/rpi-stack`), sinon clone au commit épinglé — à documenter dans le
`README.md` du consommateur, au même endroit que les 3 autres outils.

Le consommateur lit le `compose.yaml` **tel quel** (ex. Ansible `lookup('file', ...)`),
puis fournit **ses propres** valeurs (`.env` réel, copié depuis le `.example` mais
gardé dans son arborescence) et **ses propres** secrets (`secrets/`, gitignorés chez
lui — jamais `secrets.example/` d'ici).

Portainer (serveur **et** agent) est un cas particulier fonctionnel : ni l'un ni
l'autre ne peut se déployer via l'API Portainer puisque c'est justement ce qu'ils
bootstrapent — le serveur n'existe pas encore avant son propre premier démarrage, et
un hôte n'est manageable via l'API qu'une fois son agent up. C'est donc toujours en
`docker compose up -d` brut qu'on les lance, quelle que soit leur place dans ce repo.

Consommateurs connus : `rpi-nomade` (`services/portainer/`, `services/ddclient/`).

## Dépôt

https://github.com/githubf-tfr/rpi-stack — branche unique `main`.

## `CLAUDE.md` vs `README.md`

- **`CLAUDE.md`** (ce fichier) : conventions et contexte pour qui *modifie* ce repo.
  Ne catalogue pas les stacks.
- **`README.md`** : doc utilisateur — plan d'adressage, catalogue des stacks
  (variables/secrets par service), dérogations réseau détaillées, et comment
  consommer le repo.

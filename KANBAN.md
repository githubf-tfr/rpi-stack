# KANBAN — rpi-stack

Suivi des tâches du projet, **pour l'assistant** (pas un document consommateur —
pas de lien depuis `README.md`, contrairement à `KANBAN.md` de `rpi-stage`). 3
colonnes : À faire / En cours / Terminé. Mis à jour manuellement au fil des
sessions.

## À faire

- **#1 `adguard` — variante bridge/DNS-only** (`githubf-tfr/rpi-stack#1`) :
  dette technique remontée par `rpi-proxy` (nœud LAN parallèle, DHCP laissé à
  la box, uniquement le DNS des clients pointé vers le Pi). Le template actuel
  de `socle/adguard/` impose `network_mode: host` + `NET_ADMIN`, justifiés
  uniquement par le besoin de broadcast du serveur DHCP intégré — inutile pour
  ce cas d'usage. Besoin : variante (ou template paramétrable) tournant en
  bridge Docker classique + ports publiés, sans `network_mode: host` ni
  `NET_ADMIN`.
- Nouveaux templates au fil des besoins réels des autres consommateurs (DMZ,
  NAS, k3s, k3s pro) — même mécanique que `rpi-stage`/`roles` : dette laissée
  inline côté consommateur → ticket ici (cf. règle README).
- `socle/wireguard/` — profil 1 (« client nomade classique, accède au LAN du
  site ») documenté dans le template mais **pas encore utilisé par aucun
  consommateur** ; à vérifier au premier besoin réel (`AllowedIPs` exact côté
  routage LAN, pas encore éprouvé).
- Bloc `10.3.0.0/16` (site-à-site) : un seul site enregistré (`rpi-nomade`,
  migration prévue mais pas faite) — registre à tenir à jour au fil des
  prochains sites reliés.

## En cours

_(rien)_

## Terminé

- **Règle "ticket obligatoire" posée dans README/CLAUDE.md (2026-08-06)** —
  calquée sur celle de `rpi-stage` (à qui `rpi-stack` l'a empruntée) : toute
  stack/config laissée inline dans un repo consommateur au lieu d'être
  généralisée en template ici doit s'accompagner d'un ticket GitHub dans
  `rpi-stack`, avec recherche de doublon au préalable. Motivée en pratique par
  l'issue #1 (`rpi-proxy`/AdGuard), ouverte le jour même avant que la règle ne
  soit elle-même formalisée ici.
- **`socle/portainer/agent/` — template agent Portainer (2026-08-05/06)** —
  rattachement d'un hôte distant (ex. `rpi-proxy`) à un Portainer serveur
  existant. Réutilise volontairement le même `/24`/nom de réseau que
  `socle/portainer/serveur/` (dérogation documentée : serveur et agent ne
  tournent jamais sur le même hôte). Monte `/:/host` (onglet Host) — décision
  explicite, équivalent à un accès root si le conteneur est compromis. Pas de
  `healthcheck` : le binaire `healthy` de l'image ne couvre que l'Edge Agent,
  pas l'agent classique utilisé ici.
- **`socle/wireguard/` — template WireGuard (2026-08-06)** — un seul template
  couvre 3 profils de peers (nomade, admin, site-à-site) via `AllowedIPs`
  différents dans `wg_confs/wg0.conf`, pas de séparation en plusieurs
  templates (WireGuard ne distingue pas ces cas au niveau protocole). Mode
  "custom" de l'image linuxserver — zéro génération de clé/peer par ce repo.
  NAT par défaut sur les 3 profils, alternative sans NAT documentée en
  commentaire mais pas retenue par défaut.
- **Réservation `10.3.0.0/16` (sites site-à-site) et `10.1.255.0/24` (secours
  physique) (2026-08-06)** — anticipé avant même l'existence du template
  WireGuard, pour que plusieurs sites gérés avec cet outillage ne choisissent
  jamais le même subnet indépendamment.
- **`socle/uptime-kuma/` — template Uptime Kuma (2026-07-25)** — conservé dans
  ce repo mais **pas encore intégré au pipeline de déploiement automatisé**
  côté `rpi-stage` : le seul mécanisme trouvé pour créer le premier compte
  admin sans assistant web est un événement Socket.IO non documenté
  (`setup()`), qui impliquerait du code — hors périmètre zéro-code de ce repo.
  Reste utilisable dès aujourd'hui pour une infra stable (assistant web une
  seule fois au déploiement initial).
- **`socle/adguard/` — template AdGuard Home (2026-07-24)** — première
  dérogation au plan d'adressage réseau (`network_mode: host`, pas de `/24`
  dédié) : le serveur DHCP intégré doit émettre en broadcast sur le segment L2
  physique, impossible depuis un bridge Docker isolé.
- **`socle/traefik/` — template Traefik (reverse proxy + TLS) (2026-07-23)** —
  introduit le réseau partagé `net_proxy` (`10.0.0.0/16`) et le renumérotage
  du plan d'adressage par catégorie (`socle`/`métier`). Décision de router
  Portainer via le provider file plutôt que via `net_proxy`/labels (complexité
  non justifiée, Portainer déployé avant que Traefik n'existe).
- **Fusion `infra/`/`sc/` → `socle/`, structure à plat 1 dossier = 1 service
  (2026-07-23)** — la seule distinction qui compte est fondation technique vs.
  service métier réel.
- **`socle/ddclient/` — template ddclient (DNS dynamique) (2026-07-XX)** — cas
  de référence pour la convention « config applicative non découpable en
  `${VAR}` » (fichier `.example` complet monté via un chemin `${VAR}`).
- **`socle/portainer/serveur/` — template Portainer bootstrap (commit
  initial)** — mot de passe admin bcrypt pré-configuré, pas de fenêtre de 5
  min ni de setup token à recopier depuis les logs.

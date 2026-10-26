# Homelab SOC & Purple Team

> Laboratoire de cybersécurité auto-hébergé, conçu comme un mini-SOC et une
> plateforme purple team. Objectif : pratiquer la détection et la réponse
> côté défensif (blue), jouer des attaques réalistes côté offensif (red), et
> mesurer la couverture de détection en continu — le tout documenté pour le
> portfolio, en vue d'une alternance en cybersécurité.

---

## Vue d'ensemble

Le lab tourne sur un hyperviseur Proxmox (Ryzen 7), derrière un firewall
pfSense, sur un LAN isolé `192.168.10.0/24`. Il combine une chaîne de
collecte et de détection (SIEM Wazuh) et un environnement Active Directory
attaquable depuis Kali.

```
        Kali (red)      Win Server AD      Win Client
        192.168.10.102   192.168.10.10    192.168.10.20
            │                  │ agent Wazuh      │
            └──────────────────┼──────────────────┘
                               ▼  logs (1514/tcp)
                        ┌──────────────┐
                        │    Wazuh     │  192.168.10.60
                        │  SIEM (blue) │  https://192.168.10.60
                        └──────────────┘
              LAN isolé 192.168.10.0/24 — pfSense 192.168.10.1
```

Détail complet du matériel, du plan d'adressage et de la topologie :
[`infra.md`](./infra.md).

---

## Projets

| # | Projet | Statut | Doc |
|---|--------|--------|-----|
| 1 | **SOC maison — SIEM Wazuh** | 🟨 socle opérationnel | [projet1-siem-wazuh](./projet1-siem-wazuh/) |
| 2 | Purple team Active Directory | ⬜ à venir | — |
| 3 | IDS + analyse de trafic (Suricata) | ⬜ à venir | — |
| 4 | Boîte à outils d'automatisation | ⬜ à venir | — |
| 5 | Durcissement & conformité (ISO 27001 / HDS / ANSSI) | ⬜ à venir | — |

---

## État actuel

- **SIEM Wazuh 4.14.7** déployé en all-in-one (indexer + manager +
  dashboard) sur Ubuntu Server 24.04 LTS.
- **Premier agent enrôlé** : le contrôleur de domaine Windows Server AD,
  avec évaluation de configuration (SCA CIS), intégrité des fichiers (FIM)
  et rootcheck actifs.
- Infrastructure documentée, secrets exclus du versionnement.

---

## Compétences mises en avant

SIEM · monitoring · analyse de logs · réponse à incident · Active
Directory · purple team · MITRE ATT&CK · durcissement · conformité
(ISO 27001, HDS, ANSSI, RGPD) · virtualisation (Proxmox) · réseau &
firewall (pfSense) · automatisation.

---

## Stack

Proxmox VE · pfSense · Wazuh · Ubuntu Server · Windows Server 2025 (AD) ·
Kali Linux · WireGuard · Git.

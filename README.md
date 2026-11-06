# 📈 Homelab SOC & Purple Team

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-blue)
![Purple Team](https://img.shields.io/badge/Approach-Purple%20Team-purple)
![Project](https://img.shields.io/badge/Project-Homelab%20SOC-orange)
![Cybersecurity](https://img.shields.io/badge/Focus-Cybersecurity-red)

> Laboratoire de cybersécurité auto-hébergé, conçu comme un mini-SOC et une
> plateforme purple team. Objectif : pratiquer la détection et la réponse
> côté défensif (blue), jouer des attaques réalistes côté offensif (red), et
> mesurer la couverture de détection en continu — le tout documenté pour le
> portfolio, en vue d'une alternance en cybersécurité.

---

## 📋 Présentation

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

## 📂 Projets

| # | Projet | Statut | Doc |
|---|--------|--------|-----|
| 1 | **SOC maison — SIEM Wazuh** | ✅ socle complet | [projet1-siem-wazuh](./projet1-siem-wazuh/) |
| 2 | **Purple team Active Directory** | 🟨 en cours | [projet2-purpleteam-ad](./projet2-purpleteam-ad/) |
| 3 | IDS + analyse de trafic (Suricata) | ⬜ à venir | — |
| 4 | Boîte à outils d'automatisation | ⬜ à venir | — |
| 5 | Durcissement & conformité (ISO 27001 / HDS / ANSSI) | ⬜ à venir | — |

---

## 📍 État actuel

- **SIEM Wazuh 4.14.7** déployé en all-in-one (indexer + manager +
  dashboard) sur Ubuntu Server 24.04 LTS.
- **3 agents actifs** (Windows Server AD, Windows Client, Kali), avec
  **Sysmon** sur les deux hôtes Windows pour une télémétrie de niveau EDR.
- **Première boucle purple team bouclée** : énumération AD depuis Kali →
  détection dans Wazuh → **règle de détection custom** (`100100`, T1078)
  écrite et validée.
- Infrastructure documentée, secrets exclus du versionnement.

---

## 🧠 Compétences démontrées

SIEM · monitoring · analyse de logs · réponse à incident · Active
Directory · purple team · MITRE ATT&CK · écriture de règles de détection ·
durcissement · conformité (ISO 27001, HDS, ANSSI, RGPD) · virtualisation
(Proxmox) · réseau & firewall (pfSense) · automatisation.

---

## 🛠️ Stack

Proxmox VE · pfSense · Wazuh · Ubuntu Server · Windows Server 2025 (AD) ·
Kali Linux · NetExec · Sysmon · WireGuard · Git.

---

## 👤 Auteur

**Alexis Noiret** — Bachelor IT Cybersécurité, La Plateforme

- Portfolio : [alexis-noiret.students-laplateforme.io](https://alexis-noiret.students-laplateforme.io)
- GitHub : [github.com/alexis-noiret](https://github.com/alexis-noiret)

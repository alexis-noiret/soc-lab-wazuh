# Infrastructure — Homelab SOC & Purple Team

> Fichier de connaissance du projet Claude. Décrit le matériel, les hôtes,
> le plan d'adressage et la topologie du lab. À tenir à jour.

---

## 1. Matériel physique

| Rôle | Machine | Détails | OS / Hyperviseur |
|------|---------|---------|------------------|
| Firewall / routeur | Lenovo ThinkPad T430 | i5-3320M, upgradé RAM + SSD | pfSense 2.7.2-RELEASE |
| Hyperviseur | Mini PC Ryzen 7 | Bare-metal — `192.168.10.50` | Proxmox VE 9.1 (node `proxmox-lab`) |
| Station de gestion | PC Ryzen 5 | Poste d'admin | Ubuntu |
| Réseau | Switch | `À COMPLÉTER` (modèle) | — |

---

## 2. Machines virtuelles (Proxmox)

| VM | Rôle dans le lab | Services | Adresse IP |
|----|------------------|----------|------------|
| Kali Linux | Attaquant (red) | Nmap, BloodHound, outils offensifs — hostname `kali-attacker`, Docker installé (`172.17.0.0/16`) | `192.168.10.102` (vmid 100) |
| Windows Server 2025 | Contrôleur de domaine | AD DS, DNS, Kerberos — domaine `lab.local` | `192.168.10.10` (vmid 101) |
| Windows Client | Poste utilisateur joint au domaine | Client AD, cible télémétrie | `192.168.10.20` (vmid 102) |
| Wazuh | SIEM / SOC (blue) | Manager + Indexer + Dashboard (all-in-one) — Wazuh 4.14.7 sur Ubuntu Server 24.04.4 LTS | `192.168.10.60` (vmid 103) |

> Sysmon à installer sur les hôtes Windows pour enrichir la télémétrie
> envoyée au SIEM.
>
> **Agents Wazuh enrôlés :**
> - `win-server-ad` (ID 001) — Windows Server AD `192.168.10.10` — statut *active*.

---

## 3. Plan d'adressage

- **LAN lab isolé :** `192.168.10.0/24` (segmenté du réseau domestique)
- **Passerelle (pfSense LAN) :** `192.168.10.1`
- **WAN pfSense :** `192.168.1.169` (derrière la box `192.168.1.254`)
- **VPN WireGuard :** `10.0.0.0/24` (pfSense = `10.0.0.1`)
- **DNS du lab :** chaque hôte du LAN isolé résout via le resolver pfSense
  (`192.168.10.1`), qui porte les forwarders vers Internet. Centralise et
  journalise les requêtes DNS (utile pour la détection au projet 3). Le
  DNS du domaine `lab.local` reste porté par le Windows Server AD pour les
  noms internes.

| Hôte | IP | Notes |
|------|----|-------|
| pfSense (LAN) | `192.168.10.1` | Firewall + passerelle |
| Proxmox (hyperviseur) | `192.168.10.50` | Node `proxmox-lab`, console sur `:8006` |
| Windows Server AD | `192.168.10.10` | DNS du domaine lab.local (vmid 101) |
| Windows Client | `192.168.10.20` | Joint au domaine (vmid 102) |
| Kali | `192.168.10.102` | Attaquant `kali-attacker` (vmid 100) |
| Wazuh | `192.168.10.60` | SIEM déployé (vmid 103) — dashboard `https://192.168.10.60` |
| Station Ubuntu | `À COMPLÉTER` | Admin — session admin pfSense vue depuis 192.168.10.101 |

---

## 4. Accès distant

- **VPN WireGuard** hébergé sur pfSense, multi-clients :
  - iPhone (5G)
  - Laptop HP sous Linux Mint
  - Depuis l'école (fonctionnel derrière le portail captif ALCASAR filtrant)

---

## 5. Topologie (schéma ASCII)

```
                          Internet
                             │
                     ┌───────┴────────┐
                     │  pfSense (T430) │  ── WireGuard VPN ──► iPhone / laptop / école
                     │  Firewall + IDS │
                     └───────┬────────┘
                             │  LAN isolé 192.168.10.0/24
              ┌──────────────┼──────────────┬───────────────┐
              │              │              │               │
        ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐   ┌─────┴─────┐
        │   Kali    │  │ Win Server│  │Win Client │   │  Wazuh    │
        │  (red)    │  │ AD lab.   │  │  (joint)  │   │  SIEM     │
        │  .102     │  │ local .10 │  │   .20     │   │  .60      │
        │           │  │ [agent ✓] │  │           │   │  (blue)   │
        └───────────┘  └───────────┘  └───────────┘   └───────────┘
              │              │              │               ▲
              └──────────────┴──────────────┴───────────────┘
                agents Wazuh (+ Sysmon à venir) → logs vers le SIEM
                            port 1514/tcp · enrôlement 1515

     Hyperviseur : Proxmox VE 9.1 (Ryzen 7)   │   Admin : Ubuntu (Ryzen 5)
```

---

## 6. Liens

- **Portfolio :** https://alexis-noiret.students-laplateforme.io/
- **GitHub :** https://github.com/alexis-noiret
- **Dépôt cyber-lab :** https://github.com/alexis-noiret/cyber-lab-perso

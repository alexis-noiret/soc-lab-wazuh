# Projet 1 — SOC maison : SIEM Wazuh

> Premier maillon du homelab SOC & Purple Team : déploiement d'un SIEM
> Wazuh en all-in-one et enrôlement du premier agent (Windows Server AD).
> Ce projet pose le socle de collecte et de détection sur lequel
> s'appuieront les projets suivants (purple team AD, IDS Suricata,
> durcissement & conformité).

**Statut :** socle opérationnel — SIEM déployé, 1er agent actif.

---

## 1. Objectif

Mettre en place une plateforme de journalisation centralisée et de
détection pour le lab, capable de :

- collecter les logs des hôtes (Windows, Linux) via des agents,
- analyser ces logs et générer des alertes mappées MITRE ATT&CK,
- exposer des dashboards de conformité (HIPAA, GDPR, PCI DSS) —
  pertinents pour une cible santé,
- servir de brique « bleue » aux exercices purple team à venir.

---

## 2. Architecture retenue

| Élément | Choix | Justification |
|---------|-------|---------------|
| SIEM | Wazuh **4.14.7** (branche stable 4.14) | Version stable au moment du déploiement. Wazuh 5.0 encore en bêta et montée d'index irréversible → écarté pour un lab qu'on veut stable. |
| Déploiement | **All-in-one** (indexer + manager + dashboard sur une VM) | Un nœud unique gère confortablement bien plus d'agents que le lab n'en aura. Pas besoin de cluster. |
| OS | Ubuntu Server **24.04.4 LTS** | LTS, support long, base saine et minimale. |
| Hyperviseur | Proxmox VE 9.1 (Ryzen 7) | Hôte existant du lab. |
| VM specs | 4 vCPU / 8 Go RAM / 50 Go disque | L'indexer (OpenSearch, JVM) est gourmand en RAM. Ballooning désactivé pour ne pas perturber le heap Java. |
| IP | `192.168.10.60` (statique) | LAN lab isolé `192.168.10.0/24`, derrière pfSense. |

Topologie :

```
        Kali (red)      Win Server AD      Win Client
            │            192.168.10.10     192.168.10.20
            │                  │ agent Wazuh      │
            └──────────────────┼──────────────────┘
                               ▼  logs (1514/tcp)
                        ┌──────────────┐
                        │    Wazuh     │  192.168.10.60
                        │ indexer +    │  (all-in-one)
                        │ manager +    │
                        │ dashboard    │  https://192.168.10.60
                        └──────────────┘
              LAN isolé 192.168.10.0/24 — pfSense 192.168.10.1
```

---

## 3. Déploiement du SIEM

### 3.1 Provisionnement de la VM (Proxmox)

- VM ID 103, nom `wazuh`, SeaBIOS, machine q35, QEMU agent activé.
- Disque SCSI 50 Go sur `local-lvm` (discard + iothread), contrôleur
  VirtIO SCSI single.
- CPU type `host` (perf max, nœud unique).
- 8192 Mo RAM, ballooning désactivé.
- Carte réseau VirtIO sur le bridge du LAN lab (`vmbr0`).

### 3.2 Installation d'Ubuntu

Points d'attention notés pendant l'install :

- **IP statique manuelle** : `192.168.10.60/24`, gateway `192.168.10.1`,
  DNS `192.168.10.1` (le resolver pfSense).
- **Partitionnement LVM** : l'installeur n'alloue par défaut que ~la
  moitié du disque au volume root. Le `ubuntu-lv` a été **étendu au
  maximum** pour donner tout l'espace au SIEM (qui écrit beaucoup d'index).
- OpenSSH server installé, aucun snap ajouté (surface minimale).

### 3.3 Installation de Wazuh (all-in-one)

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Le flag `-a` installe indexer + manager + dashboard, génère les
certificats et un mot de passe admin (affiché en fin d'install et stocké
dans `wazuh-install-files.tar`).

Vérification :

```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard
```

Les trois services en `active (running)`. Dashboard accessible sur
`https://192.168.10.60` (certificat auto-signé au premier accès).

---

## 4. Enrôlement du premier agent — Windows Server AD

Cible prioritaire : le contrôleur de domaine `192.168.10.10`
(`lab.local`), là où se joueront les exercices purple team AD.

### 4.1 Installation de l'agent (PowerShell admin)

```powershell
# Téléchargement du MSI (version alignée sur le manager)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi" -OutFile "$env:tmp\wazuh-agent.msi"

# Installation silencieuse en pointant le manager
msiexec.exe /i "$env:tmp\wazuh-agent.msi" /q WAZUH_MANAGER="192.168.10.60" WAZUH_AGENT_NAME="win-server-ad"
```

### 4.2 Enregistrement auprès du manager (port 1515)

```powershell
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m 192.168.10.60 -A "win-server-ad"
NET START WazuhSvc
```

Résultat attendu dans les logs de l'agent
(`C:\Program Files (x86)\ossec-agent\ossec.log`) :

```
agent-auth: INFO: Valid key received
wazuh-agent: INFO: Connected to the server ([192.168.10.60]:1514/tcp)
wazuh-agent: INFO: Agent is now online.
```

Confirmation côté dashboard (Agents management → Endpoints) :

| ID | Name | IP | OS | Version | Status |
|----|------|----|----|---------|--------|
| 001 | win-server-ad | 192.168.10.10 | Windows Server 2025 | v4.14.7 | active |

---

## 5. Lessons learned (pièges rencontrés)

- **DNS du Proxmox** : le nœud résolvait vers la box domestique
  (`192.168.1.1`, injoignable depuis le LAN isolé). Corrigé en le
  pointant vers le resolver pfSense (`192.168.10.1`). Décision d'archi :
  dans un LAN isolé, chaque hôte résout via le resolver du segment, qui
  centralise et journalise le DNS (utile pour la détection d'exfiltration
  DNS au projet 3).
- **Téléchargement de l'ISO** : une URL de dossier (au lieu du fichier
  `.iso`) renvoie une page HTML → toujours vérifier la **taille annoncée**
  (~3 Go pour l'ISO, pas 15 ko) avant de lancer.
- **Enrôlement de l'agent** : la ligne d'install collée depuis
  l'assistant, retapée à la main en console, avait des mots collés
  (`...60'WAZUH_AGENT_NAME`), ce qui a écrit l'adresse du manager avec le
  paramètre agrégé dedans. Diagnostic via `ossec.conf` + `ossec.log`,
  correction de la balise `<address>`, puis ré-enregistrement avec
  `agent-auth.exe`. Leçon : découper les commandes longues et vérifier la
  config effective (`<address>` propre) avant de conclure.

---

## 6. Sécurité & hygiène

- Les secrets générés par l'installeur ne sont **jamais** versionnés (voir
  `.gitignore`) : `wazuh-install-files.tar` (certificats + mots de passe),
  `wazuh-passwords.txt`, clés et certificats.
- `wazuh-install-files.tar` verrouillé : `chown root:root`, `chmod 600`.
- Snapshots Proxmox pris à chaque étape stable (`ubuntu-clean`,
  `wazuh-installed-ok`) comme points de retour.
- À durcir ensuite (projet 5) : SSH par clé uniquement, remplacement du
  certificat auto-signé du dashboard, changement du mot de passe admin
  généré.

---

## 7. Prochaines étapes

- [ ] Installer et configurer **Sysmon** sur les hôtes Windows (télémétrie enrichie).
- [ ] Enrôler les agents restants (Windows Client, Kali).
- [ ] Ingestion des logs **pfSense** dans Wazuh.
- [ ] Première règle de détection custom + premier dashboard.
- [ ] Brancher les exercices purple team (projet 2) sur ce SIEM.

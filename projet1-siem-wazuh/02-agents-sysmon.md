# Projet 1 — Étape 2 : enrôlement des agents & télémétrie Sysmon

> Suite du déploiement du SIEM. Objectif : couvrir l'ensemble des hôtes du
> lab avec des agents Wazuh, et enrichir la télémétrie Windows avec Sysmon
> pour disposer d'une visibilité de niveau EDR (création de processus,
> réseau, DNS, DLL) — indispensable aux détections Active Directory du
> projet 2.

**Statut :** 3 agents actifs, Sysmon opérationnel sur les 2 hôtes Windows.

---

## 1. Agents enrôlés

| ID | Nom | Hôte | IP | OS | Méthode |
|----|-----|------|----|----|---------|
| 001 | win-server-ad | DC lab.local | 192.168.10.10 | Windows Server 2025 | MSI |
| 002 | win-client | client01 | 192.168.10.20 | Windows 11 Pro | MSI |
| 003 | kali-attacker | kali | 192.168.10.102 | Kali Linux | dépôt APT |

> Note : le nom d'agent Wazuh (ex. `win-client`) est indépendant du
> hostname réel de la machine (ex. `client01`). Dans les événements on
> retrouve les deux : `agent.name` et le champ `computer`.

### Enrôlement Windows (MSI)

Commandes découpées pour éviter les erreurs de saisie en console (voir
lessons learned) :

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi" -OutFile "$env:tmp\wazuh-agent.msi"
msiexec.exe /i "$env:tmp\wazuh-agent.msi" /q WAZUH_MANAGER="192.168.10.60" WAZUH_AGENT_NAME="<nom>"
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m 192.168.10.60 -A "<nom>"
NET START WazuhSvc
```

Contrôle systématique de l'adresse manager après install :

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.conf" | Select-String "address"
# attendu : <address>192.168.10.60</address>
```

### Enrôlement Linux / Kali (dépôt APT)

Méthode plus fiable : le manager est passé en variable d'environnement et
écrit directement dans la configuration, sans risque de saisie erronée.

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER="192.168.10.60" WAZUH_AGENT_NAME="kali-attacker" apt install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

---

## 2. Sysmon sur les hôtes Windows

Sysmon (System Monitor, Sysinternals) ajoute au journal Windows une
télémétrie détaillée absente par défaut : création de processus avec ligne
de commande complète (Event ID 1), connexions réseau (ID 3), requêtes DNS
(ID 22), création de fichiers (ID 11), etc.

### Installation (par hôte Windows)

```powershell
iwr "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:tmp\Sysmon.zip"
Expand-Archive "$env:tmp\Sysmon.zip" -DestinationPath "$env:tmp\Sysmon" -Force
iwr "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:tmp\Sysmon\sysmonconfig.xml"
cd "$env:tmp\Sysmon"
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Config utilisée : **SwiftOnSecurity/sysmon-config** — baseline reconnue,
point de départ à affiner ensuite (au projet 2, quand une attaque ne
remonte pas, on ajuste la config Sysmon). Alternative plus modulaire et
mappée ATT&CK : la config d'Olaf Hartong.

### Collecte du canal Sysmon par Wazuh

Ajout du canal à la configuration **partagée du groupe `default`** (tous
les agents Windows du groupe en héritent), côté manager, dans
`/var/ossec/etc/shared/default/agent.conf` (voir [`agent.conf`](./agent.conf)) :

```xml
<agent_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

Puis :

```bash
sudo chown wazuh:wazuh /var/ossec/etc/shared/default/agent.conf
sudo chmod 660 /var/ossec/etc/shared/default/agent.conf
sudo /var/ossec/bin/verify-agent-conf   # doit répondre OK
sudo systemctl restart wazuh-manager
```

---

## 3. Vérification

Dans le dashboard (Threat Hunting), requête :

```
data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"
```

Résultat observé : événements Sysmon des deux hôtes Windows, déjà
catégorisés par les règles Wazuh (`sysmon`, `sysmon_eid1_detection`,
`sysmon_eid11_detection`) et mappés MITRE ATT&CK (PowerShell, Account
Discovery, Windows Command Shell, Ingress Tool Transfer, DLL Search Order
Hijacking).

---

## 4. Lessons learned

- **Ordre d'installation Sysmon / agent** : si l'agent Wazuh reçoit la conf
  « collecter le canal Sysmon » **avant** que Sysmon ne soit installé, il
  journalise une erreur `EvtSubscribe() ... (15007)` (canal introuvable) et
  n'abonne pas le canal. Correctif : installer Sysmon d'abord, puis
  `Restart-Service WazuhSvc` pour que l'agent re-tente l'abonnement. Après
  ça, la ligne `Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'`
  apparaît sans erreur.
- **MSI vs saisie manuelle** : la ligne d'install MSI collée puis retapée en
  console noVNC est source d'erreurs (mots collés → adresse manager
  corrompue). Découper les commandes et vérifier `<address>` après install.
- **Ne pas confondre les shells** : les commandes `Get-Content ...ossec.log`
  se lancent côté Windows (PowerShell), pas dans le SSH de la VM Wazuh
  (Linux). De même, les requêtes `data.win.system.channel:...` se saisissent
  dans la barre de recherche du dashboard web, pas dans un terminal.
- **Cache d'affichage du dashboard** : la page d'accueil "Overview" peut
  afficher « no agents » alors que les agents sont actifs. La vue
  **Endpoints** lit directement le manager et donne l'état réel.

---

## 5. Couverture actuelle

- 3 agents actifs (2 Windows + Kali).
- Télémétrie Sysmon opérationnelle sur les 2 hôtes Windows.
- Règles Sysmon et mapping MITRE ATT&CK actifs dans Wazuh.

Prêt pour le projet 2 (purple team AD) : les attaques lancées depuis Kali
génèrent des événements Sysmon sur les cibles Windows, corrélés et mappés
ATT&CK dans le dashboard.

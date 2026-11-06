# Journal de détection — Purple Team

> Cœur du travail purple team. Chaque ligne = une technique jouée depuis
> Kali (rouge) puis vérifiée dans Wazuh (bleu). But : mesurer et améliorer
> la couverture de détection en continu, mappée sur MITRE ATT&CK.
>
> Boucle : attaquer → observer → (si non détecté) créer/améliorer la règle
> → rejouer → confirmer.

---

## Tableau

| Date | Technique | MITRE | Commande / outil | Événement généré | Détecté | Règle | Notes |
|------|-----------|-------|------------------|------------------|---------|-------|-------|
| 2026-10-11 | Énumération AD via compte lambda | T1087.002 | `nxc ldap srv-plv-dc01.lab.local -u alexis -p *** --bloodhound --collection All` (compte `alexis`, zéro privilège) | Binds LDAP + logons sur le DC → Wazuh : MITRE **Domain Accounts** + **Valid Accounts**, 167 auth success | ✅ | Règles Wazuh par défaut (groupe `windows` + mapping ATT&CK) | Voir note d'exercice ci-dessous |
| 2026-11-06 | Logon NTLM distant depuis l'hôte attaquant | T1078 | `nxc ldap srv-plv-dc01.lab.local -u alexis -p ***` | Event 4624 (logon réseau NTLM) → règle Wazuh **92652** (level 6, "possible pass-the-hash") | ✅ | **Règle custom `100100` (level 12)** — escalade 92652 quand `ipAddress` = Kali | Validée en live, faux-positif ANONYMOUS LOGON exclu par `negate` |
| `AAAA-MM-JJ` | Kerberoasting | T1558.003 | `GetUserSPNs.py` | Event 4769 (ticket TGS) | ❌ | `À CRÉER` | Prochaine technique |
| `AAAA-MM-JJ` | AS-REP roasting | T1558.004 | `GetNPUsers.py` | Event 4768 (pré-auth off) | — | — | À jouer |
| `AAAA-MM-JJ` | Pass-the-hash | T1550.002 | `nxc smb ...` | Event 4624 type 3 | — | — | À jouer |
| `AAAA-MM-JJ` | Scan réseau | T1046 | `nmap -sV ...` | Alerte Suricata | — | — | Vérifier remontée IDS→Wazuh |

---

## Exercice 2026-10-11 — Énumération AD (premier exercice purple team)

**Scénario.** Un attaquant a compromis un compte utilisateur standard (`alexis`,
aucun privilège) et énumère l'annuaire pour cartographier le domaine
`lab.local` (DC `SRV-PLV-DC01`, 192.168.10.10).

**Déroulé.**
1. `bloodhound-python` (collecteur legacy) **échoue** : le DC impose le
   **LDAP signing (Enforced)** et n'expose **pas de LDAPS** (pas de certif).
   Le bind NTLM non signé est refusé.
2. Contournement avec **NetExec** : `nxc ldap` négocie le *sign & seal* SASL
   sur le port 389 → canal signé accepté → `[+] lab.local\alexis` authentifié,
   énumération réussie.

**Côté détection (Wazuh, agent `win-server-ad`, canal Security).**
- ~372 events sur le DC en 1 h, pic net à l'heure de l'attaque.
- Mapping MITRE automatique : **T1087.002 Domain Accounts**, **T1078 Valid
  Accounts** (+ Pass the Hash / Account Manipulation en marge).
- 167 authentifications réussies = la rafale de requêtes annuaire.

**Enseignement clé.** Le DC résiste à l'énumération LDAP « facile » grâce au
**LDAP signing** (issu du script de durcissement `04_Securite_AD.ps1`). Le
signing n'est pas qu'un réglage : c'est une **mitigation** qui force
l'attaquant vers un canal signé/scellé et modifie la trace laissée.

---

## Exercice 2026-11-06 — Première règle custom Wazuh (détection ciblée)

**Objectif.** Passer de la détection générique à une **règle maison** qui
escalade en priorité haute un logon venant de l'hôte attaquant connu (Kali,
`192.168.10.102`).

**Méthode (boucle complète attaque → détection → règle → rejeu).**
1. Rejoué l'auth depuis Kali (`nxc ldap ... -u alexis`) → génère un
   **Event 4624** (logon réseau NTLM) sur le DC.
2. Identifié la règle Wazuh de base qui matche : **92652** — *"Successful
   Remote Logon Detected... NTLM authentication, possible pass-the-hash
   attack"*, level 6. (Au premier essai, règle parente supposée `60106`
   fausse → corrigée après inspection du `rule.id` réel dans l'event.)
3. Écrit la règle custom `100100` (voir `local_rules.xml`) : escalade de
   92652 au niveau 12 si `ipAddress` = Kali, en excluant les ANONYMOUS LOGON.
4. Restart du manager → rejeu `nxc` → **règle 100100 déclenchée en level 12**,
   description dynamique correcte (*"...compte alexis depuis 192.168.10.102"*).

**Enseignements.**
- **Les règles Wazuh ne sont pas rétroactives** : regénérer l'event après
  création/modification de la règle.
- **Toujours lire le `rule.id` réel** de l'event avant de choisir `if_sid`.
- **Affiner pour réduire le bruit** : exclusion des ANONYMOUS LOGON via
  `<field name="win.eventdata.targetUserName" negate="yes">ANONYMOUS</field>`.
- Principe réutilisable : prendre une règle générique et **l'escalader selon
  le contexte** (ici une IP attaquante connue = threat intel maison).

---

## Couverture MITRE ATT&CK (synthèse)

| Tactique | Techniques testées | Détectées | Couverture |
|----------|--------------------|-----------|-----------|
| Reconnaissance / Discovery | 1 (T1087.002) | 1 | 🟩 |
| Lateral Movement | 1 (T1078) | 1 | 🟩 (règle custom 100100) |
| Credential Access | 0 | 0 | ⬜ |
| Persistence | 0 | 0 | ⬜ |

---

## Idées de techniques à couvrir ensuite

- Kerberoasting (T1558.003) — prochaine technique logique
- Golden / Silver ticket
- DCSync
- Énumération des partages SMB
- Bruteforce Kerberos
- Exfiltration DNS

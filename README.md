# SIEM-Log-Monitoring-Threat-Detection
> Détection d'attaques réelles via Splunk Free, Universal Forwarders, Sysmon et 4 machines virtuelles.

[![Splunk](https://img.shields.io/badge/SIEM-Splunk%20Free-FF6B35?style=flat-square&logo=splunk)](https://www.splunk.com/en_us/download/splunk-enterprise.html)
[![Sysmon](https://img.shields.io/badge/Windows-Sysmon-0078D4?style=flat-square&logo=windows)](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
[![Platform](https://img.shields.io/badge/Platform-VirtualBox-183A61?style=flat-square&logo=virtualbox)](https://www.virtualbox.org/)
[![MITRE](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-E63946?style=flat-square)](https://attack.mitre.org/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

---

## Présentation

Ce projet simule un environnement SOC (Security Operations Center) complet en local, avec :

- Un **SIEM Splunk** centralisant les logs de 2 machines cibles
- Des **agents Universal Forwarder** envoyant les logs en temps réel
- **Sysmon** configuré sur Windows pour une télémétrie riche
- Des **attaques simulées** (brute force, exploitation, reverse shell, PowerShell obfusqué)
- Des **règles de détection SPL** avec mapping MITRE ATT&CK

**Objectif pédagogique** : reproduire le workflow complet d'un analyste Tier 1–2 SOC, de l'ingestion des logs jusqu'au triage d'alertes et à la rédaction d'un rapport d'incident.

---

## Architecture du lab

```
┌─────────────────────────────────────────────────────────────┐
│                  Réseau Host-Only : 192.168.57.0/24          │
│                                                              │
│  ┌──────────────┐             ┌─────────────────────────┐   │
│  │  Kali Linux  │             │    Ubuntu SIEM          │   │
│  │ 192.168.57.10│<-(attaquant)│    Splunk Free          │   │
│  └──────────────┘             │    192.168.57.20:8000   │   │
│                               └─────────────────────────┘   │
│  ┌──────────────┐    UF       ▲                             │
│  │Metasploitable│ ────────────┤                             │
│  │192.168.57.30 │  auth.log   │                             │
│  └──────────────┘  syslog     │                             │
│                               │                             │
│  ┌──────────────┐    UF       │                             │
│  │  Windows 10  │ ────────────┘                             │
│  │192.168.57.40 │  Sysmon + WinEventLog                     │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

| Machine | OS | Rôle | RAM | IP |
|---|---|---|---|---|
| Kali Linux | Kali Rolling | Attaquant | 2 GB | 192.168.57.10 |
| Ubuntu SIEM | Ubuntu 22.04 LTS | Splunk Free | 4 GB | 192.168.57.20 |
| Metasploitable 2 | Ubuntu 8.04 | Cible Linux vulnérable | 1 GB | 192.168.57.30 |
| Windows 10 | Windows 10 Pro | Cible Windows + Sysmon | 2 GB | 192.168.57.40 |

---

## Stack technique

| Composant | Version | Rôle |
|---|---|---|
| Splunk Enterprise (Free) | 9.x | SIEM — indexation, recherche, alertes |
| Splunk Universal Forwarder | 9.x | Agent de collecte sur chaque cible |
| Sysmon (Sysinternals) | 15.x | Télémétrie Windows avancée |
| Sysmon config (SwiftOnSecurity) | Latest | Couverture optimale des Event IDs |
| Splunk Add-on for Sysmon | Latest | Parsing automatique des champs XML |
| VirtualBox | 7.x | Hyperviseur de type 2 |

---

## Structure du repo

```
SIEM-Splunk-Lab/
├── README.md
├── architecture/
│   └── network-diagram.png          # Schéma réseau du lab
├── splunk-config/
│   ├── inputs.conf                  # Configuration Universal Forwarder Linux
│   ├── inputs-windows.conf          # Configuration Universal Forwarder Windows
│   └── transforms.conf              # Normalisation des champs
├── sysmon/
│   └── sysmonconfig.xml             # Config SwiftOnSecurity modifiée
├── detection-rules/
│   ├── ssh-bruteforce.md            # SPL + MITRE T1110.001
│   ├── powershell-obfuscation.md    # SPL + MITRE T1059.001
│   ├── reverse-shell.md             # SPL + MITRE T1059
│   ├── lateral-movement.md          # SPL + MITRE T1021
│   └── suspicious-account.md       # SPL + MITRE T1136.001
├── attack-simulations/
│   ├── 01-hydra-ssh-bruteforce.md
│   ├── 02-nmap-recon.md
│   ├── 03-metasploit-vsftpd.md
│   ├── 04-powershell-encoded.md
│   └── 05-reverse-shell-netcat.md
└── writeups/
    └── incident-report-ssh-bruteforce.md
```

---

##  Installation & configuration

### Prérequis

- VirtualBox 7.x installé
- ~12 GB de RAM disponible
- ~80 GB d'espace disque
- Images ISO : Kali Linux, Ubuntu 22.04, Windows 10
- OVA Metasploitable 2 (téléchargeable sur SourceForge)

### 1. Réseau VirtualBox

```bash
# File → Host Network Manager → Créer
Subnet : 192.168.57.0/24
DHCP   : désactivé (IPs statiques)

# Chaque VM : 2 interfaces
Adaptateur 1 : Host-Only (192.168.57.0/24)
Adaptateur 2 : NAT (accès internet)
```

### 2. Splunk Free sur Ubuntu SIEM

```bash
# Télécharger depuis splunk.com (compte gratuit requis)
sudo dpkg -i splunk-9.x.x-linux-amd64.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start

# Autoriser les ports
sudo ufw allow 8000/tcp   # Web UI
sudo ufw allow 9997/tcp   # Universal Forwarders

# Accès : http://192.168.57.20:8000
```

### 3. Universal Forwarder — Linux (Metasploitable)

```bash
sudo dpkg -i splunkforwarder-9.x-linux-amd64.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.57.20:9997

# Monitorer les logs d'authentification
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log \
  -index linux_logs -sourcetype linux_secure
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog \
  -index linux_logs
```

### 4. Sysmon + Universal Forwarder — Windows

```powershell
# 1. Installer Sysmon avec la config SwiftOnSecurity
.\Sysmon64.exe -accepteula -i sysmon\sysmonconfig.xml

# 2. Installer le Universal Forwarder (MSI)
msiexec.exe /i splunkforwarder.msi `
  RECEIVING_INDEXER="192.168.57.20:9997" `
  WINEVENTLOG_SEC_ENABLE=1 `
  WINEVENTLOG_SYS_ENABLE=1 `
  AGREETOLICENSE=Yes /quiet
```

`inputs.conf` Windows (`C:\Program Files\SplunkUniversalForwarder\etc\system\local\`) :

```ini
[WinEventLog://Security]
index = windows_logs
disabled = 0

[WinEventLog://System]
index = windows_logs
disabled = 0

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = 0
renderXml = true
```

---

## Attaques simulées

Toutes les attaques sont confinées au réseau host-only `192.168.57.0/24`.

| # | Attaque | Source | Cible | MITRE |
|---|---|---|---|---|
| 1 | Brute Force SSH (Hydra) | Kali | Metasploitable | T1110.001 |
| 2 | Scan réseau Nmap agressif | Kali | Toutes | T1046 |
| 3 | Exploitation vsftpd 2.3.4 (Metasploit) | Kali | Metasploitable | T1190 |
| 4 | PowerShell encodé base64 | Local | Windows | T1059.001 |
| 5 | Reverse Shell Netcat | Metasploitable | Kali | T1059 |
| 6 | Création compte admin caché | Local | Windows | T1136.001 |

Chaque simulation est documentée dans `attack-simulations/` avec : commandes exactes, logs générés, Event IDs attendus, et capture Splunk.

---

## Règles de détection SPL

### Brute Force SSH

```spl
index=linux_logs sourcetype=linux_secure "Failed password"
| stats count by src_ip, user
| where count > 5
| eval severity="HIGH", mitre="T1110.001"
| table _time, src_ip, user, count, severity, mitre
| sort -count
```

### PowerShell obfusqué (base64)

```spl
index=sysmon EventCode=1
(CommandLine="*-enc*" OR CommandLine="*EncodedCommand*"
 OR CommandLine="*-e *" OR CommandLine="*-en *")
| eval severity="CRITICAL", mitre="T1059.001"
| table _time, ComputerName, User, ParentImage, CommandLine, severity
```

### Connexion réseau suspecte (Sysmon EID 3)

```spl
index=sysmon EventCode=3
| stats count by DestinationIp, DestinationPort, Image, User
| where count > 10
| eval mitre="T1071"
| sort -count
```

### Création de compte Windows

```spl
index=windows_logs EventCode=4720
| eval severity="HIGH", mitre="T1136.001"
| table _time, ComputerName, SubjectUserName, TargetUserName, severity
```

> Toutes les règles sont sauvegardées comme **Alertes temps réel** dans Splunk avec seuil et déclencheur configurés.

---

## Dashboard Splunk — SOC Overview

Le dashboard principal (`SOC_Overview`) inclut :

- **Top 10 Source IPs** par volume de logs (bar chart)
- **Failed Logins over time** — Linux + Windows (line chart)
- **Sysmon Event IDs distribution** (pie chart)
- **Alertes actives par sévérité** (metric cards)
- **Timeline des événements suspects** (table filtrable)

---

## Rapport d'incident — exemple

Un write-up complet est disponible dans `writeups/incident-report-ssh-bruteforce.md`. Il suit le format SOC standard :

1. **Résumé exécutif** — alerte déclenchée, heure, sévérité
2. **Timeline** — chronologie des événements dans Splunk
3. **IOCs** — adresses IP, user agents, commandes
4. **Analyse SPL** — requêtes utilisées pour l'investigation
5. **Remédiation** — actions recommandées (blocage IP, reset password, etc.)
6. **Mapping MITRE ATT&CK** — tactiques et techniques identifiées

---

## Compétences développées

| Domaine | Compétences |
|---|---|
| SIEM | Ingestion de logs, indexation, création d'index, SPL |
| Detection Engineering | Écriture de règles, tuning, corrélation, alertes temps réel |
| Log Analysis | Parsing auth.log, Sysmon XML, WinEventLog |
| Incident Response | Triage d'alertes, investigation, rédaction de rapport |
| Windows Security | Sysmon, Event IDs, audit policy, PowerShell logging |
| Linux Security | auth.log, syslog, SSH hardening |
| Threat Intelligence | MITRE ATT&CK mapping, TTP identification |

---

## Auteur

**Alberick Mahoussi**    
Spécialisation Blue Team · SOC Analysis & Incident Response  
En recherche d'alternance SOC — septembre 2026

[![LinkedIn](https://img.shields.io/badge/LinkedIn-alberick--mahoussi-0A66C2?style=flat-square&logo=linkedin)](https://linkedin.com/in/alberick-mahoussi)
[![Email](https://img.shields.io/badge/Email-alberick.mahoussi%40ecole2600.com-EA4335?style=flat-square&logo=gmail)](mailto:alberick.mahoussi@ecole2600.com)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-%40huen992-212C42?style=flat-square&logo=tryhackme)](https://tryhackme.com/p/huen992)
[![Root--Me](https://img.shields.io/badge/Root--Me-%40huen992-343434?style=flat-square)](https://www.root-me.org/huen992)

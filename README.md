# TP Supervision — Zabbix 7.0 LTS

TP de supervision réalisé en B3 à l'EFREI : installer un serveur **Zabbix 7.0 LTS** dans une VM VMware Workstation, puis superviser une **VM Linux** (serveur web Nginx) et un **PC Windows**.

## 🎯 But du TP

Superviser, c'est mesurer en continu l'état des machines et des services pour détecter une panne **avant** que les utilisateurs ne la signalent. Le but du TP est de mettre en place une chaîne de supervision complète :

1. **Collecter** des métriques (CPU, mémoire, disques, services) grâce à l'agent Zabbix 2, à des *simple checks*, à des scénarios web et à SNMP ;
2. **Détecter** les problèmes avec des *triggers* (seuils, sans fausses alertes) ;
3. **Prévenir** les bonnes personnes par e-mail ;
4. **Visualiser** l'état de l'infrastructure sur un tableau de bord ;
5. **Réduire le bruit** (dépendances, maintenances) et **dépanner** de façon méthodique.

À la fin, Zabbix détecte l'arrêt de Nginx sur la VM Linux et l'arrêt du service d'impression (Spooler) sur le PC Windows, les affiche sur un tableau de bord et envoie une alerte.

## 🏗️ Architecture

```
                 Réseau NAT VMware (VMnet8) — 192.168.100.0/24
                 Passerelle / DNS : 192.168.100.2

 ┌──────────────────────┐   TCP 10050   ┌──────────────────────┐
 │ ZBX-SRV (VM Ubuntu)  │ ────────────▶ │ LNX-CLIENT (VM)      │
 │ 192.168.100.10       │               │ 192.168.100.20       │
 │ Zabbix server        │               │ Agent 2 + Nginx      │
 │ MySQL + Apache/PHP   │               │ + snmpd (partie 11)  │
 └──────────────────────┘               └──────────────────────┘
            │  TCP 10050
            ▼
 ┌──────────────────────┐
 │ PC Windows (hôte)    │   ← le navigateur du PC ouvre
 │ 192.168.100.1        │     http://192.168.100.10/zabbix
 │ Agent 2 (PC-WINDOWS) │
 └──────────────────────┘
```

| Machine | Rôle | IP | Ressources |
|---|---|---|---|
| ZBX-SRV (VM) | Serveur Zabbix, MySQL, interface web | 192.168.100.10 | 2 cœurs, 4 Go RAM, 30 Go |
| LNX-CLIENT (VM) | Linux supervisé (Nginx) | 192.168.100.20 | 1 cœur, 2 Go RAM, 20 Go |
| PC Windows | Windows supervisé | 192.168.100.1 (carte VMnet8) | — |

**Ports à retenir :** `10050` serveur → agent (passif) · `10051` agent → serveur (actif) · `80` interface web · `3306` MySQL (local) · `161/udp` SNMP.

## ✅ Prérequis

- PC Windows 10/11, 8 Go de RAM minimum (16 conseillés), 60 Go libres, virtualisation activée dans le BIOS, droits administrateur ;
- [VMware Workstation Pro](https://support.broadcom.com) (gratuit) ;
- ISO [Ubuntu Server 24.04 LTS](https://ubuntu.com/download/server) ;
- [Agent Zabbix 2 pour Windows](https://www.zabbix.com/download_agents) (7.0 LTS, amd64, OpenSSL, MSI).

> Les mots de passe ci-dessous (`ZbxLab2026!`, `Admin/zabbix`) sont ceux d'un **lab** : ne jamais les réutiliser en production.

---

## 🛠️ Réalisation

### Tronc commun (parties 1 à 7)

<details>
<summary><b>Partie 1 — Réseau VMware</b></summary>

`Edit › Virtual Network Editor › Change Settings › VMnet8` :
- Subnet `192.168.100.0`, masque `255.255.255.0` ;
- NAT Settings : passerelle `192.168.100.2` ;
- DHCP Settings : plage `192.168.100.128` → `192.168.100.254` (les adresses basses restent libres pour les IP fixes).

Vérification : `ipconfig` sur le PC montre la carte **VMnet8** en `192.168.100.1`.
</details>

<details>
<summary><b>Partie 2 — VM ZBX-SRV et Ubuntu Server</b></summary>

- *New Virtual Machine › Custom* : « installer l'OS plus tard », Ubuntu 64-bit, 2 cœurs, 4 Go, NAT, disque 30 Go ;
- Monter l'ISO, installer Ubuntu Server (utilisateur `stagiaire`, nom `zbx-srv`, **cocher OpenSSH**) ;
- Mettre à jour et relever l'IP :

```bash
sudo apt update && sudo apt upgrade -y
ip -br addr
```

Se connecter ensuite en SSH depuis le PC (`ssh stagiaire@192.168.100.1xx`) pour pouvoir copier-coller.
</details>

<details>
<summary><b>Partie 3 — Cloner la VM et fixer les IP</b></summary>

1. Éteindre ZBX-SRV, puis *Manage › Clone › Full clone* → `LNX-CLIENT` (2 Go, 1 cœur, **nouvelle adresse MAC**).
2. Sur ZBX-SRV, désactiver la gestion réseau de cloud-init et créer `/etc/netplan/01-tp.yaml` :

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo nano /etc/netplan/01-tp.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.100.10/24]
      routes:
        - to: default
          via: 192.168.100.2
      nameservers:
        addresses: [192.168.100.2]
```

```bash
sudo chmod 600 /etc/netplan/01-tp.yaml
sudo netplan apply
```

3. Sur LNX-CLIENT, rendre le clone unique puis mettre l'IP `.20` dans le même fichier :

```bash
sudo hostnamectl set-hostname lnx-client
sudo sed -i 's/zbx-srv/lnx-client/g' /etc/hosts
sudo rm /etc/machine-id && sudo systemd-machine-id-setup
sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server
sudo nano /etc/netplan/01-tp.yaml   # 192.168.100.20
sudo netplan apply && sudo reboot
```

Vérification : le PC pingue les deux VM, ZBX-SRV pingue LNX-CLIENT, les deux VM ont Internet.
</details>

<details>
<summary><b>Partie 4 — Installer le serveur Zabbix (sur ZBX-SRV)</b></summary>

```bash
# Dépôt officiel Zabbix 7.0
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update

# Paquets
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
  zabbix-sql-scripts zabbix-agent2 zabbix-get mysql-server fping
```

Base de données (`sudo mysql`) :

```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'ZbxLab2026!';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

Import du schéma (1 à 3 min, sans affichage) puis vérification :

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
sudo mysql -e "set global log_bin_trust_function_creators = 0;"
sudo mysql -e "select count(*) from zabbix.users;"
```

Dans `/etc/zabbix/zabbix_server.conf` : `DBPassword=ZbxLab2026!`, puis :

```bash
sudo systemctl restart zabbix-server zabbix-agent2 apache2
sudo systemctl enable zabbix-server zabbix-agent2 apache2
sudo tail -n 20 /var/log/zabbix/zabbix_server.log
```

Terminer l'assistant sur `http://192.168.100.10/zabbix` (MySQL, `localhost`, base/user `zabbix`, fuseau `Europe/Paris`), se connecter en `Admin / zabbix` et **changer le mot de passe**.

Vérification : *Zabbix server is running : Yes* et l'hôte *Zabbix server* est vert.
</details>

<details>
<summary><b>Partie 5 — Superviser LNX-CLIENT</b></summary>

Sur LNX-CLIENT : ajouter le même dépôt Zabbix, puis :

```bash
sudo apt install -y zabbix-agent2 nginx
sudo nano /etc/zabbix/zabbix_agent2.conf
```

```ini
Server=192.168.100.10
ServerActive=192.168.100.10
Hostname=LNX-CLIENT
```

```bash
sudo systemctl restart zabbix-agent2 && sudo systemctl enable zabbix-agent2
sudo ss -tlnp | grep 10050
```

Test depuis ZBX-SRV :

```bash
zabbix_get -s 192.168.100.20 -k agent.ping                  # 1
zabbix_get -s 192.168.100.20 -k 'net.tcp.service[http,,80]' # 1
```

Dans l'interface :
- **Hôte** `LNX-CLIENT` — templates *Linux by Zabbix agent* + *ICMP Ping*, groupe *Linux servers*, interface Agent `192.168.100.20:10050`, tag `role=web` ;
- **Item** *Nginx : port 80* — Simple check, clé `net.tcp.service[http,,80]`, toutes les 30 s ;
- **Trigger** *Nginx ne répond plus sur {HOST.NAME}* (Average) : `last(/LNX-CLIENT/net.tcp.service[http,,80])=0`.

Test : `sudo systemctl stop nginx` → problème dans *Monitoring › Problems* ; `start` → RESOLVED.

*Bonus* : trigger CPU avec hystérésis `avg(...,5m)>80` / recovery `avg(...,5m)<50`, testé avec `stress-ng`.
</details>

<details>
<summary><b>Partie 6 — Superviser le PC Windows</b></summary>

Installer le MSI de l'agent 2 : Host name `PC-WINDOWS`, serveur `192.168.100.10`, port `10050`, active checks `192.168.100.10`, PSK décoché.

PowerShell **administrateur** :

```powershell
Get-Service "Zabbix Agent 2"
New-NetFirewallRule -DisplayName "Zabbix agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -RemoteAddress 192.168.100.10 -Action Allow
New-NetFirewallRule -DisplayName "Ping depuis Zabbix" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -RemoteAddress 192.168.100.10 -Action Allow
```

Test depuis ZBX-SRV : `zabbix_get -s 192.168.100.1 -k 'service.info[Spooler,state]'` → `0` (démarré).

Dans l'interface :
- Groupe *Postes Windows* ; hôte `PC-WINDOWS` avec *Windows by Zabbix agent* + *ICMP Ping*, interface `192.168.100.1:10050`, tag `role=poste` ;
- Item *Service Spooler : état* — clé `service.info[Spooler,state]`, 30 s ;
- Trigger (High) : `last(/PC-WINDOWS/service.info[Spooler,state])<>0` ;
- Macro d'hôte `{$CPU.UTIL.CRIT}` = `70` (prioritaire sur le 90 du template).

Test : `Stop-Service Spooler` / `Start-Service Spooler`.
</details>

<details>
<summary><b>Partie 7 — Tableau de bord et test final</b></summary>

Dashboard **TP VMware** avec 4 widgets : *Problems*, *Host availability*, *Graph* (CPU des deux hôtes), *Top hosts* (mémoire %).
Test final : arrêter Nginx et le Spooler en même temps, acquitter un problème (« Je m'en occupe »), puis tout rétablir.
</details>

### Approfondissement (parties 8 à 13)

| Partie | Sujet | Ce qu'il faut faire |
|---|---|---|
| 8 | **Alertes e-mail** | Média *Email* via le sandbox SMTP Mailtrap (STARTTLS, port 2525), adresse sur l'utilisateur Admin, action *Alertes Linux – TP B3* : groupe *Linux servers*, sévérité ≥ Average, e-mail à l'ouverture **et** à la résolution, objet personnalisé avec macros (`{EVENT.SEVERITY}`, `{EVENT.NAME}`, `{HOST.NAME}`). |
| 9 | **Mesure personnalisée** | `UserParameter=tp.fichiers[*],find $1 -maxdepth 1 -type f \| wc -l` dans `/etc/zabbix/zabbix_agent2.d/`, item `tp.fichiers[/srv/depot]` et trigger High « plus de 5 fichiers pendant 2 min » (indice : `min(...,2m)>5`). |
| 10 | **Scénario web** | Scénario *Intranet* qui vérifie le code 200 et la chaîne « Intranet TP B3 », triggers *Intranet en erreur* (`web.test.fail`) et *Intranet lent* (temps moyen > 2 s sur 5 min). Démontrer une panne de contenu invisible pour le Simple check. |
| 11 | **SNMP sans agent** | `snmpd` sur LNX-CLIENT (communauté `tpb3` limitée à ZBX-SRV, `public` désactivée), hôte `LNX-CLIENT-SNMP` avec *Linux by SNMP*, communauté stockée dans la macro `{$SNMP_COMMUNITY}`. |
| 12 | **Réduire le bruit** | Dépendances des triggers Nginx/Intranet vers *Unavailable by ICMP ping* (en les rendant plus lents : 5 min), puis période de maintenance *With data collection*. |
| 13 | **Défi de dépannage** | En binôme, l'un sabote, l'autre diagnostique en 20 min à partir de ce que montre Zabbix, et remplit une fiche : symptôme, hypothèses, diagnostic, cause, correction, prévention. |

---

## 🧰 Aide-mémoire

```bash
# Services et journaux (Linux)
sudo systemctl status zabbix-server
sudo systemctl restart zabbix-agent2
sudo tail -f /var/log/zabbix/zabbix_server.log
sudo tail -f /var/log/zabbix/zabbix_agent2.log

# Tester un agent depuis le serveur
zabbix_get -s 192.168.100.20 -k agent.ping
zabbix_get -s 192.168.100.1 -k system.cpu.util
```

```powershell
# Windows (admin)
Get-Service "Zabbix Agent 2"
Restart-Service "Zabbix Agent 2"
Get-Content "C:\Program Files\Zabbix Agent 2\zabbix_agent2.log" -Tail 20
```

| Symptôme | Cause probable | Solution |
|---|---|---|
| *Zabbix server is running : No* | Service arrêté / mauvais mot de passe BDD | `systemctl status zabbix-server`, lire le journal, vérifier `DBPassword` |
| `Table 'zabbix.users' doesn't exist` | Schéma non importé | Refaire l'import (partie 4) |
| Icône ZBX rouge | Agent arrêté, pare-feu, mauvaise IP | `zabbix_get -k agent.ping` depuis ZBX-SRV |
| `Check access restrictions` | `Server=` faux dans l'agent | Corriger et redémarrer l'agent |
| `host not found` dans le journal agent | `Hostname=` ≠ nom de l'hôte (casse comprise) | Aligner les deux noms |
| Timeout vers le PC | Pare-feu Windows | Recréer les règles (partie 6) |
| Erreur YAML netplan | Indentation | 2 espaces, jamais de tabulation |

**Réflexe de dépannage** : suivre le chemin de la donnée — la machine répond-elle au ping ? le service tourne-t-il ? le port est-il ouvert ? le serveur a-t-il le droit de demander ? Zabbix interroge-t-il la bonne adresse ?

## 🧹 Après le TP

- Éteindre les VM : `sudo poweroff` ;
- Désinstaller l'agent Windows si besoin, puis supprimer les règles de pare-feu :

```powershell
Remove-NetFirewallRule -DisplayName "Zabbix agent"
Remove-NetFirewallRule -DisplayName "Ping depuis Zabbix"
```

## 📚 Ressources

- [Zabbix — téléchargement et commandes d'installation](https://www.zabbix.com/download)
- [Zabbix — agents Windows](https://www.zabbix.com/download_agents)
- [Documentation Zabbix 7.0](https://www.zabbix.com/documentation/7.0/)
- [Ubuntu Server](https://ubuntu.com/download/server)

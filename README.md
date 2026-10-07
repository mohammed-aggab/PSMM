# PSMM - Système de Surveillance et Gestion d'Infrastructure
## Documentation Complète (Jobs 1-14)

**Plateforme:** Moon Exercice - La Plateforme  
**Date:** 2026-10-07  
**Responsable:** Mohammed Aggab  
**Email:** mohammedaggab7@gmail.com

---

## 📋 Vue d'ensemble du projet

**PSMM** est un système de surveillance automatisé qui:
- Une VM d'administration surveille 3 serveurs (FTP, Web, MariaDB)
- Détecte les tentatives de connexion frauduleuses
- Détecte les problèmes de ressources (CPU, RAM, Disque)
- Prévient l'administrateur par email et Google Chat
- Effectue les mises à jour automatiques
- Centralise les erreurs et logs

---

## 🖥️ Infrastructure

| Serveur | IP | Rôle |
|---------|----|----|
| Management | 10.60.0.203 | Serveur d'administration & monitoring |
| FTP | 10.60.0.201 | Serveur FTP |
| Web | 10.60.0.200 | Serveur Web (Nginx) |
| MariaDB | 10.60.0.202 | Base de données |

---

## 📚 Description complète des 14 Jobs

### **Job 1: Monitoring de base du système**

**Fichier:** `system_monitoring.py`

**Description:**
Récupère les métriques de CPU, RAM et Disque de chaque serveur via SSH et les insère dans la base de données.

**Fonctionnalités:**
- Connexion SSH à chaque VM avec authentification par clé
- Récupération CPU via `/proc/loadavg`
- Récupération RAM via `/proc/meminfo`
- Récupération Disque via `df /`
- Insertion des données dans la table `system_status`
- Nettoyage automatique des données > 72h
- Gestion robuste des erreurs et timeouts

**Configuration:**
```python
MONITOR_USER = "monitor"
SSH_KEY = "/root/.ssh/id_rsa"
SSH_TIMEOUT = 5
MARIADB_IP = "10.60.0.202"
MARIADB_USER = "monitor"
MARIADB_PASS = "monitor"

vms = {
    "Management": "10.60.0.203",
    "FTP": "10.60.0.201",
    "Web": "10.60.0.200",
    "MariaDB": "10.60.0.202"
}
```

**Fréquence:** Toutes les 5 minutes

**Cron:**
```bash
*/5 * * * * /usr/bin/python3 /root/system_monitoring.py >> /var/log/system_monitoring.log 2>&1
```

---

### **Job 2: Vérification de la connexion SSH**

**Fichier:** `ssh_login.py`

**Description:**
Vérifie que l'utilisateur `monitor` peut se connecter à chaque serveur via SSH et teste l'authentification par clé.

**Fonctionnalités:**
- Test de connectivité SSH vers chaque serveur
- Vérifie que l'authentification par clé fonctionne
- Enregistre les tentatives de connexion réussies/échouées
- Détecte les problèmes de configuration SSH
- Valide les permissions de fichiers (~/.ssh et id_rsa)

**Commandes exécutées:**
```bash
ssh -i /root/.ssh/id_rsa monitor@<IP> "whoami"
```

**Fréquence:** Toutes les 5 minutes

**Cron:**
```bash
*/5 * * * * /usr/bin/python3 /root/ssh_login.py >> /var/log/ssh_login.log 2>&1
```

---

### **Job 3: Détection des erreurs FTP**

**Fichier:** `ssh_ftp_error.py`

**Description:**
Analyse les logs du serveur FTP pour détecter les erreurs, rejets de connexion et anomalies.

**Fonctionnalités:**
- Récupère les logs vsftpd du serveur FTP
- Cherche les patterns d'erreur (failed login, timeout, etc.)
- Enregistre les erreurs détectées
- Alerte si trop d'erreurs en peu de temps
- Aide à identifier les tentatives d'accès frauduleuses

**Logs surveillés:**
```bash
/var/log/vsftpd.log
```

**Patterns détectés:**
- Connexions échouées
- Authentifications refusées
- Timeouts
- Erreurs de permissions

**Fréquence:** Toutes les heures

**Cron:**
```bash
0 * * * * /usr/bin/python3 /root/ssh_ftp_error.py >> /var/log/ssh_ftp_error.log 2>&1
```

---

### **Job 4: Détection des erreurs de login système**

**Fichier:** `ssh_login_sudo.py`

**Description:**
Analyse les tentatives de sudo échouées et les erreurs d'authentification sur chaque serveur.

**Fonctionnalités:**
- Récupère les logs d'authentification système
- Détecte les tentatives sudo échouées
- Identifie les erreurs de permission
- Enregistre les utilisateurs avec accès refusé
- Alerte en cas de tentatives suspectes répétées

**Logs surveillés:**
```bash
/var/log/auth.log
/var/log/secure
```

**Patterns détectés:**
- "sudo: ... COMMAND NOT ALLOWED"
- "Authentication failure"
- "Invalid user"
- "sudo: ... not in sudoers file"

**Fréquence:** Toutes les heures

**Cron:**
```bash
0 * * * * /usr/bin/python3 /root/ssh_login_sudo.py >> /var/log/ssh_login_sudo.log 2>&1
```

---

### **Job 5: Détection des erreurs MySQL/MariaDB**

**Fichier:** `ssh_mysql_error.py`

**Description:**
Analyse les erreurs dans les logs MySQL/MariaDB pour détecter les problèmes de base de données.

**Fonctionnalités:**
- Récupère les logs MariaDB
- Cherche les erreurs de connexion
- Détecte les problèmes de performance (slow queries)
- Identifie les tables corrompues
- Alerte sur les erreurs critiques

**Logs surveillés:**
```bash
/var/log/mysql/error.log
/var/log/mariadb/mariadb.log
```

**Patterns détectés:**
- Erreurs de connexion
- Queries slow
- Fichiers manquants
- Tables corrompues
- Erreurs de réplication

**Fréquence:** Toutes les 5 minutes

**Cron:**
```bash
*/5 * * * * /usr/bin/python3 /root/ssh_mysql_error.py >> /var/log/ssh_mysql_error.log 2>&1
```

---

### **Job 6: Requêtes SQL paramétrées (Sécurité)**

**Fichier:** `ssh_mysql.py`

**Description:**
Fournit une interface sécurisée pour exécuter des requêtes MySQL en utilisant des requêtes paramétrées contre l'injection SQL.

**Fonctionnalités:**
- Utilise les requêtes paramétrées pour éviter l'injection SQL
- Échappe les caractères spéciaux
- Valide les entrées
- Centralise les erreurs
- Enregistre les queries pour audit

**Pattern d'utilisation:**
```python
# Mauvais (vulnérable)
query = f"SELECT * FROM users WHERE id = {user_id};"

# Bon (sécurisé - Job 6)
query = "SELECT * FROM users WHERE id = %s"
execute_query(query, [user_id])
```

**Seuils de sécurité:**
- Validation de tous les paramètres
- Échappement des caractères spéciaux
- Préparation des statements
- Enregistrement des erreurs

**Fréquence:** À la demande (utilisé par d'autres jobs)

---

### **Job 7: Exécution sécurisée des commandes MySQL**

**Fichier:** `ssh_mysql.py` (partie II)

**Description:**
Wrapper sécurisé pour l'exécution de commandes MySQL avec gestion des erreurs et logging.

**Fonctionnalités:**
- Exécute les requêtes via SSH sur le serveur MariaDB
- Gère les erreurs de connexion
- Enregistre les résultats
- Timeout configurable
- Retry automatique en cas d'échec temporaire

**Code exemple:**
```python
def execute_mysql_query(query, host, user, password):
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password
        )
        cursor = connection.cursor()
        cursor.execute(query)
        results = cursor.fetchall()
        return results
    except mysql.connector.Error as err:
        logger.error(f"MySQL Error: {err}")
```

**Fréquence:** À la demande (utilisé par d'autres jobs)

---

### **Job 8: Envoi d'alertes mail à l'administrateur**

**Fichier:** `ssh_server_mail.py`

**Description:**
Envoie des alertes mail détaillées à l'administrateur quand des problèmes sont détectés.

**Fonctionnalités:**
- Envoie des alertes formatées par email
- Inclut les détails du problème
- Ajoute les logs pertinents
- Envoie les récapitulatifs quotidiens
- Limitation du spam (max 1 email par heure par type d'erreur)

**Contenu des alertes:**
- Date et heure de l'erreur
- Serveur affecté
- Type d'erreur
- Details techniques
- Actions recommandées

**Configuration mSMTP:**
```bash
apt install msmtp msmtp-mta mailutils

# Configuration /etc/msmtprc
account gmail
auth plain
host smtp.gmail.com
port 587
from mohammedaggab7@gmail.com
user mohammedaggab7@gmail.com
password [APP_PASSWORD]
```

**Format email:**
```
Subject: [ALERTE PSMM] Erreur détectée - [Serveur]

Date: YYYY-MM-DD HH:MM:SS
Serveur: [Nom du serveur]
Type: [Type d'erreur]

Description:
[Détails complets]

Actions recommandées:
- Vérifier la configuration
- Consulter les logs
- Redémarrer le service si nécessaire
```

**Fréquence:** À la demande (déclenché par d'autres jobs)

---

### **Job 9: Détection des erreurs Web (Nginx)**

**Fichier:** `ssh_web_error.py`

**Description:**
Analyse les logs du serveur Web (Nginx) pour détecter les erreurs HTTP, timeouts et anomalies.

**Fonctionnalités:**
- Récupère les logs Nginx
- Détecte les erreurs HTTP (4xx, 5xx)
- Identifie les patterns d'attaque (brute force, SQL injection)
- Enregistre les timeouts et erreurs de connexion
- Alerte sur les anomalies de trafic

**Logs surveillés:**
```bash
/var/log/nginx/error.log
/var/log/nginx/access.log
```

**Erreurs détectées:**
- 400 Bad Request
- 401 Unauthorized
- 403 Forbidden
- 404 Not Found
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout
- Connexions refused
- Connexions timeout

**Fréquence:** Toutes les heures

**Cron:**
```bash
0 * * * * /usr/bin/python3 /root/ssh_web_error.py >> /var/log/ssh_web_error.log 2>&1
```

---

### **Job 10: Vérification de l'état des services**

**Fichier:** `ssh_service_check.py`

**Description:**
Vérifie que les services critiques sont actifs sur chaque serveur.

**Fonctionnalités:**
- Se connecte à chaque VM via SSH
- Vérifie l'état de chaque service avec `systemctl is-active`
- Enregistre l'état (running/stopped) dans MariaDB
- Alerte si un service est arrêté
- Affiche un résumé avec ✓ ou ✗

**Services monitorés:**
- Management: `ssh`
- FTP: `vsftpd`
- Web: `nginx`
- MariaDB: `mysql`

**Commande SSH:**
```bash
systemctl is-active [service_name]
```

**Table MariaDB:**
```sql
INSERT INTO service_status (vm_name, service_name, status, check_time)
VALUES ('FTP', 'vsftpd', 'running', NOW());
```

**Fréquence:** Chaque heure

**Cron:**
```bash
0 * * * * /usr/bin/python3 /root/ssh_service_check.py >> /var/log/ssh_service_check.log 2>&1
```

---

### **Job 11: Monitoring système avec stockage en base**

**Fichier:** `ssh_system_status.py`

**Description:**
Version avancée du monitoring système (Job 1). Récupère les métriques de CPU, RAM et Disque et les insère dans MariaDB.

**Fonctionnalités:**
- Connexion SSH robuste à chaque VM
- Récupération CPU, RAM, Disque
- Insertion dans `system_status`
- Nettoyage des données > 72h
- Gestion des erreurs avec retry
- Logging détaillé

**Métriques collectées:**
```
vm_name: Nom du serveur
cpu_percent: Charge CPU en %
ram_percent: Utilisation RAM en %
disk_percent: Utilisation Disque en %
check_time: Timestamp de la mesure
```

**Fréquence:** Toutes les 5 minutes

**Cron:**
```bash
*/5 * * * * /usr/bin/python3 /root/ssh_system_status.py >> /var/log/ssh_system_status.log 2>&1
```

---

### **Job 12: Sauvegarde cron des configurations**

**Fichier:** `ssh_cron_backup.py`

**Description:**
Effectue une sauvegarde programmée des configurations cron et des scripts de monitoring.

**Fonctionnalités:**
- Sauvegarde du fichier crontab de chaque serveur
- Archivage compressé (tar.gz)
- Horodatage automatique
- Sauvegarde locale + distant
- Vérification d'intégrité des backups
- Suppression des vieux backups (> 30 jours)

**Fichiers sauvegardés:**
```bash
/etc/crontab
/var/spool/cron/crontabs/*
/root/*.py (scripts monitoring)
/etc/msmtprc (config email)
```

**Structure de sauvegarde:**
```
/backup/cron/
├── cron_backup_2026-10-07_14-30-00.tar.gz
├── cron_backup_2026-10-06_14-30-00.tar.gz
└── cron_backup_2026-10-05_14-30-00.tar.gz
```

**Fréquence:** Une fois par jour à 23:00

**Cron:**
```bash
0 23 * * * /usr/bin/python3 /root/ssh_cron_backup.py >> /var/log/ssh_cron_backup.log 2>&1
```

---

### **Job 13: Monitoring avec alertes email (Seuils)**

**Fichier:** `ssh_system_mail.py`

**Description:**
Extension du Job 11 avec système d'alertes par email quand les seuils sont dépassés.

**Seuils d'alerte (configurables):**
```python
SEUIL_CPU = 70%
SEUIL_RAM = 80%
SEUIL_DISK = 90%
```

**Fonctionnalités:**
- Récupère les métriques (comme Job 11)
- Vérifie les seuils configurés
- Envoie une alerte email si dépassement
- Limitation: max 1 email par heure par serveur (anti-spam)
- Stockage des timestamps pour throttling
- Format email détaillé avec valeurs actuelles

**Format de l'alerte email:**
```
Subject: [ALERTE] Serveur FTP - Ressources

Alerte système détectée

Date : 2026-10-07 14:25:30
Serveur : FTP

Seuils dépassés :
  - RAM : 85% (seuil : 80%)
  - DISK : 95% (seuil : 90%)

Valeurs actuelles :
  - CPU : 45%
  - RAM : 85%
  - DISK : 95%

Actions recommandées:
- Libérer de l'espace disque
- Arrêter les services inutiles
- Augmenter la RAM
```

**Configuration email:**
```python
EMAIL_FROM = "mohammedaggab7@gmail.com"
EMAIL_ADMIN = "mohammed.aggab@laplateforme.io"
MSMTP_ACCOUNT = "gmail"
TIMESTAMP_FILE = "/var/log/last_mail_alert.json"
```

**Throttling:**
```python
def peut_envoyer_alerte(nom_vm):
    # Retourne True si > 1 heure depuis le dernier email
    derniere_alerte = timestamps.get(nom_vm)
    if not derniere_alerte:
        return True
    
    difference = (datetime.now() - derniere_alerte).total_seconds()
    return difference >= 3600  # 1 heure
```

**Fréquence:** Toutes les 5 minutes

**Cron:**
```bash
*/5 * * * * /usr/bin/python3 /root/ssh_system_mail.py >> /var/log/ssh_system_mail.log 2>&1
```

---

### **Job 14: Mise à jour automatique des serveurs**

**Fichier:** `ssh_update.py`

**Description:**
Vérifie et installe les mises à jour de sécurité sur tous les serveurs. Envoie une alerte si un redémarrage est nécessaire.

**Fonctionnalités:**
- Vérifie les mises à jour disponibles (`apt list --upgradable`)
- Installe les mises à jour automatiquement (`apt upgrade -y`)
- Détecte si un redémarrage est requis (`needrestart -b`)
- Envoie une alerte email si redémarrage nécessaire
- Enregistre les mises à jour installées
- Gestion des erreurs et rollback

**Commandes exécutées (individuellement):**
```bash
apt update                  # Met à jour la liste des packages
apt list --upgradable       # Liste les packages à mettre à jour
apt upgrade -y              # Installe les mises à jour
needrestart -b              # Détecte les redémarrages nécessaires
```

**Détection du redémarrage:**
```bash
needrestart -b
# Retourne 0 si pas de redémarrage nécessaire
# Retourne autre code si redémarrage requis
```

**Email d'alerte en cas de redémarrage:**
```
Subject: [REBOOT REQUIS] Serveur Management

Date: 2026-10-07 02:15:30
Serveur: Management

Redémarrage requis après les mises à jour suivantes:
- kernel 5.10.0-8-generic
- systemd 247.3-5
- openssl 1.1.1k-1

Actions:
1. Prévoir une maintenance
2. Redémarrer le serveur
3. Vérifier que les services redémarrent correctement

Commande:
sudo reboot
```

**Fréquence:** Une fois par jour à 02:00 (heures creuses)

**Cron:**
```bash
0 2 * * * /usr/bin/python3 /root/ssh_update.py >> /var/log/ssh_update.log 2>&1
```

---

## 🌐 Google Chat: Notifications périodiques

**Fichier:** `google_chat.py`

**Description:**
Envoie un rapport périodique de l'état des serveurs sur un Google Chat Space via webhook.

**Fonctionnalités:**
- Récupère l'état actuel des serveurs
- Formate un message visuel avec les métriques
- Envoie le message sur Google Chat
- Notifie tous les membres du Space
- Inclut les alertes critiques

**Configuration Google Chat:**
1. Créer un Space Google Chat
2. Ajouter les membres du groupe + accompagnateur
3. Créer un Webhook entrant (Manage → Apps and integrations → Webhooks → Incoming webhooks)
4. Copier l'URL du webhook

**Configuration Python:**
```python
WEBHOOK_URL = "https://chat.googleapis.com/v1/spaces/SPACE_ID/messages?key=KEY&token=TOKEN"
```

**Format du message Google Chat:**
```
État du système - 14:30:25

🔴 **Management** (10.60.0.203)
  CPU: 35% | RAM: 60% | Disque: 45%
  Services: SSH ✓

🟡 **FTP** (10.60.0.201)
  CPU: 75% | RAM: 82% | Disque: 88%
  Services: vsftpd ✓

🟢 **Web** (10.60.0.200)
  CPU: 25% | RAM: 55% | Disque: 52%
  Services: nginx ✓

🔵 **MariaDB** (10.60.0.202)
  CPU: 45% | RAM: 70% | Disque: 60%
  Services: mysql ✓

Alertes: FTP RAM > 80%, FTP Disque > 90%
```

**Installation:**
```bash
pip install requests
```

**Fréquence:** Chaque heure

**Cron:**
```bash
0 * * * * /usr/bin/python3 /root/google_chat.py >> /var/log/google_chat.log 2>&1
```

---

## 🔐 Configuration complète

### SSH - Utilisateur dédié

```bash
# Créer l'utilisateur monitor
useradd -m -s /bin/bash monitor

# Générer les clés SSH
ssh-keygen -t ed25519 -f /home/monitor/.ssh/id_rsa -N ""

# Configurerles permissions
mkdir -p /home/monitor/.ssh
chmod 700 /home/monitor/.ssh
chmod 600 /home/monitor/.ssh/id_rsa

# Copier pour root
cp /home/monitor/.ssh/id_rsa /root/.ssh/id_rsa
chmod 600 /root/.ssh/id_rsa
```

### Droits sudo

```bash
# Créer le fichier sudoers
echo "monitor ALL=(ALL) NOPASSWD: /usr/bin/mysql, /usr/bin/apt, /usr/sbin/needrestart" > /etc/sudoers.d/monitor
chmod 0440 /etc/sudoers.d/monitor

# Vérifier
visudo -c
```

### MariaDB

```sql
-- Créer la base de données
CREATE DATABASE monitoring;

-- Table system_status
CREATE TABLE system_status (
    id INT AUTO_INCREMENT PRIMARY KEY,
    vm_name VARCHAR(50),
    cpu_percent INT,
    ram_percent INT,
    disk_percent INT,
    check_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_vm_time (vm_name, check_time)
);

-- Table service_status
CREATE TABLE service_status (
    id INT AUTO_INCREMENT PRIMARY KEY,
    vm_name VARCHAR(50),
    service_name VARCHAR(100),
    status VARCHAR(20),
    check_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_vm_time (vm_name, check_time)
);

-- Créer l'utilisateur
CREATE USER 'monitor'@'10.60.0.%' IDENTIFIED BY 'monitor';
GRANT INSERT, SELECT, DELETE ON monitoring.* TO 'monitor'@'10.60.0.%';
FLUSH PRIVILEGES;
```

### Email (mSMTP)

```bash
# Installation
apt install msmtp msmtp-mta mailutils

# Configuration /etc/msmtprc
cat > /etc/msmtprc << 'EOF'
defaults
auth on
tls on
tls_starttls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /var/log/msmtp

account gmail
auth plain
host smtp.gmail.com
port 587
from mohammedaggab7@gmail.com
user mohammedaggab7@gmail.com
password [MOT_DE_PASSE_APPLICATION]

account default : gmail
EOF

chmod 600 /etc/msmtprc

# Test
echo -e "Subject: Test\n\nMessage de test" | msmtp -a gmail mohammed.aggab@laplateforme.io
```

---

## 📊 Calendrier des exécutions

| Job | Tâche | Fréquence | Heure |
|-----|-------|-----------|-------|
| 1 | Monitoring de base | Toutes les 5 min | * |
| 2 | Vérification SSH | Toutes les 5 min | * |
| 3 | Erreurs FTP | Chaque heure | 0 * * * * |
| 4 | Erreurs Login/Sudo | Chaque heure | 0 * * * * |
| 5 | Erreurs MySQL | Toutes les 5 min | */5 * * * * |
| 6 | Requêtes sécurisées | À la demande | - |
| 7 | Exécution MySQL | À la demande | - |
| 8 | Alertes mail | À la demande | - |
| 9 | Erreurs Web | Chaque heure | 0 * * * * |
| 10 | Vérification services | Chaque heure | 0 * * * * |
| 11 | Monitoring système | Toutes les 5 min | */5 * * * * |
| 12 | Backup cron | Une fois par jour | 23:00 |
| 13 | Alertes seuils | Toutes les 5 min | */5 * * * * |
| 14 | Mise à jour serveurs | Une fois par jour | 02:00 |
| - | Google Chat | Chaque heure | 0 * * * * |

---

## 📁 Structure des fichiers

```
/root/
├── system_monitoring.py      # Job 1
├── ssh_login.py              # Job 2
├── ssh_ftp_error.py          # Job 3
├── ssh_login_sudo.py         # Job 4
├── ssh_mysql_error.py        # Job 5
├── ssh_mysql.py              # Job 6 & 7
├── ssh_server_mail.py        # Job 8
├── ssh_web_error.py          # Job 9
├── ssh_service_check.py      # Job 10
├── ssh_system_status.py      # Job 11
├── ssh_cron_backup.py        # Job 12
├── ssh_system_mail.py        # Job 13
├── ssh_update.py             # Job 14
├── google_chat.py            # Google Chat
└── .ssh/
    └── id_rsa                # Clé SSH

/var/log/
├── system_monitoring.log
├── ssh_login.log
├── ssh_ftp_error.log
├── ssh_login_sudo.log
├── ssh_mysql_error.log
├── ssh_server_mail.log
├── ssh_web_error.log
├── ssh_service_check.log
├── ssh_system_status.log
├── ssh_cron_backup.log
├── ssh_system_mail.log
├── ssh_update.log
├── google_chat.log
└── msmtp                     # Logs email

/etc/
├── msmtprc                   # Config email
└── sudoers.d/
    └── monitor               # Droits sudo

/backup/
└── cron/
    └── cron_backup_*.tar.gz  # Backups cron
```

---

## 🔍 Monitoring et diagnostic

### Logs

```bash
# Job en temps réel
tail -f /var/log/ssh_system_mail.log

# Tous les logs
ls -la /var/log/ssh_*.log /var/log/system_*.log

# Erreurs mails
tail -f /var/log/msmtp

# Cron
grep CRON /var/log/syslog | tail -20
```

### Base de données

```bash
# Nombre d'enregistrements
mysql -h 10.60.0.202 -u monitor -p'monitor' monitoring \
  -e "SELECT COUNT(*) FROM system_status;"

# Dernières métriques
mysql -h 10.60.0.202 -u monitor -p'monitor' monitoring \
  -e "SELECT * FROM system_status ORDER BY check_time DESC LIMIT 10;"

# Métriques par VM
mysql -h 10.60.0.202 -u monitor -p'monitor' monitoring \
  -e "SELECT vm_name, COUNT(*) as count, 
       MAX(cpu_percent) as cpu_max, 
       AVG(cpu_percent) as cpu_avg,
       MAX(ram_percent) as ram_max
    FROM system_status 
    GROUP BY vm_name;"
```

### Test des jobs

```bash
# Test Job 1
/usr/bin/python3 /root/system_monitoring.py

# Test Job 13 (avec email)
/usr/bin/python3 /root/ssh_system_mail.py

# Test email directement
echo -e "Subject: Test\n\nMessage de test" | \
  msmtp -a gmail mohammed.aggab@laplateforme.io
```

---

## 🚨 Troubleshooting

### SSH: "Permission denied"
```bash
# Vérifier les droits
chmod 600 /root/.ssh/id_rsa
chmod 700 /root/.ssh

# Tester la connexion
ssh -i /root/.ssh/id_rsa monitor@10.60.0.200 "whoami"

# Vérifier authorized_keys sur la VM cible
ssh monitor@10.60.0.200 "cat ~/.ssh/authorized_keys"
```

### Email non reçu
```bash
# Vérifier mSMTP
msmtp --serverinfo -a gmail

# Tester l'envoi
echo "Test" | msmtp -a gmail mohammed.aggab@laplateforme.io

# Vérifier les logs
tail -f /var/log/msmtp
```

### Base de données "Access denied"
```bash
# Vérifier les identifiants
mysql -h 10.60.0.202 -u monitor -p'monitor' -e "SELECT 1;"

# Vérifier les droits MySQL
mysql -u root -p
SHOW GRANTS FOR 'monitor'@'10.60.0.%';
```

### Cron ne s'exécute pas
```bash
# Vérifier le service
systemctl status cron

# Vérifier les tâches
crontab -l

# Vérifier les logs cron
journalctl -u cron | tail -20

# Accès à /var/log/
ls -la /var/log/ssh_*.log
```

### Job 14: needrestart non trouvé
```bash
apt install needrestart
```

---

## 📈 Performance et capacité

**Volume de données:**
- Monitoring toutes les 5 min: ~288 enregistrements/jour par VM
- 4 VMs = ~1,152 enregistrements/jour
- 72h de conservation = ~3,456 enregistrements en base
- Espace disque estimé: ~500 KB pour 72h

**CPU/RAM estimée:**
- Scripts: < 5% CPU chacun
- MariaDB: ~100 MB RAM
- Total: Négligeable sur une VM moderne

---

## 📝 Changelog

**2026-10-07:**
- Documentation complète des 14 jobs
- Ajout de Job 12 (Backup cron)
- Affinage des seuils Job 13
- Configuration finale du système

---

## 👤 Auteur & Contact

**Développeur:** Mohammed Aggab  
**Email:** mohammedaggab7@gmail.com  
**Organisation:** La Plateforme - Moon Exercice  

---

**Dernière mise à jour:** 2026-10-07  
**Version:** 1.0 - Complète  
**Statut:** Production

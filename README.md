# 📞 Standard Téléphonique Multilingue avec Asterisk
### École Polytechnique de Sousse

---

> **Description :** Mise en place d'un standard téléphonique IVR (Interactive Voice Response) multilingue permettant d'orienter automatiquement les appelants vers les bons services (Scolarité, Comptabilité, Administration) avec messagerie vocale et notification par email.

---

## 📋 Table des matières

1. [Architecture du projet](#-architecture-du-projet)
2. [Prérequis](#-prérequis)
3. [Installation d'Asterisk](#1️⃣-installation-dasterisk)
4. [Configuration des fichiers audio](#2️⃣-configuration-des-fichiers-audio)
5. [Configuration des extensions SIP](#3️⃣-configuration-des-extensions-sip-pjsipconf)
6. [Configuration du standard IVR](#4️⃣-configuration-du-standard-ivr-extensionsconf)
7. [Configuration de la messagerie vocale](#5️⃣-configuration-de-la-messagerie-vocale-voicemailconf)
8. [Configuration des notifications email](#6️⃣-configuration-des-notifications-email-postfix)
9. [Rechargement de la configuration](#7️⃣-rechargement-de-la-configuration)
10. [Démarrage et gestion du serveur](#8️⃣-démarrage-et-gestion-du-serveur)
11. [Test avec Zoiper](#9️⃣-test-avec-zoiper)
12. [Liste des comptes SIP](#-liste-des-comptes-sip)

---

## 🏗 Architecture du projet
<img width="1030" height="864" alt="Capture d&#39;écran 2026-04-22 233739" src="https://github.com/user-attachments/assets/74cfe254-a0aa-47e0-a730-92e0fd0e03cc" />


## 📌 Prérequis

| Élément | Détail |
|---|---|
| 🖥️ OS | Ubuntu Server 22.04 |
| 🔐 Accès | Root ou sudo |
| 🌐 Réseau | Connexion Internet pour les téléchargements |
| 📡 IP | Adresse IP fixe : `192.168.1.20` |

---

## 1️⃣ Installation d'Asterisk

```bash
sudo apt update && sudo apt upgrade -y
```

**Installation des dépendances :**
```bash
sudo apt install -y build-essential git curl wget libssl-dev libncurses5-dev \
libnewt-dev libxml2-dev libsqlite3-dev uuid-dev libjansson-dev libedit-dev \
pkg-config libxslt1-dev python3-dev liburiparser-dev libcurl4-openssl-dev
```

**Téléchargement et compilation d'Asterisk 20.15.1 :**
```bash
cd /usr/src
sudo wget https://github.com/asterisk/asterisk/archive/refs/tags/20.15.1.tar.gz \
  -O asterisk-20.15.1.tar.gz
sudo tar xzf asterisk-20.15.1.tar.gz
cd asterisk-20.15.1
```

**Configuration et compilation :**
```bash
sudo ./configure --with-jansson-bundled
sudo make -j$(nproc)
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

**Création de l'utilisateur Asterisk :**
```bash
sudo adduser --system --group --home /var/lib/asterisk --no-create-home asterisk
sudo chown -R asterisk:asterisk /etc/asterisk /var/{lib,log,spool}/asterisk /usr/lib/asterisk
```

**Démarrage du service :**
```bash
sudo systemctl start asterisk
sudo systemctl enable asterisk
```

**Vérification de l'installation :**
```bash
sudo asterisk -rvvv
core show version
exit
```

---

## 2️⃣ Configuration des fichiers audio

**Création des répertoires :**
```bash
sudo mkdir -p /var/lib/asterisk/sounds/fr
sudo mkdir -p /var/lib/asterisk/sounds/en
sudo mkdir -p /var/lib/asterisk/sounds/ar
```

**Transfert des fichiers audio depuis votre PC (PowerShell) :**
```powershell
scp .\bienvenue.mp3 sami@192.168.1.20:/tmp/
scp .\menu_francais.mp3 sami@192.168.1.20:/tmp/
scp .\menu_anglais.mp3 sami@192.168.1.20:/tmp/
scp .\menu_arabe.mp3 sami@192.168.1.20:/tmp/
```

**Conversion au format GSM :**
```bash
sudo apt install -y ffmpeg

# Annonce de bienvenue (choix de langue)
sudo ffmpeg -i /tmp/bienvenue.mp3 -ar 8000 -ac 1 -c:a gsm -f gsm \
  /var/lib/asterisk/sounds/fr/choix-langue.gsm

# Menu principal - Français
sudo ffmpeg -i /tmp/menu_francais.mp3 -ar 8000 -ac 1 -c:a gsm -f gsm \
  /var/lib/asterisk/sounds/fr/menuprincipal.gsm

# Menu principal - Anglais
sudo ffmpeg -i /tmp/menu_anglais.mp3 -ar 8000 -ac 1 -c:a gsm -f gsm \
  /var/lib/asterisk/sounds/en/menuprincipal.gsm

# Menu principal - Arabe
sudo ffmpeg -i /tmp/menu_arabe.mp3 -ar 8000 -ac 1 -c:a gsm -f gsm \
  /var/lib/asterisk/sounds/ar/menuprincipal.gsm
```

**Attribution des droits et nettoyage :**
```bash
sudo chown -R asterisk:asterisk /var/lib/asterisk/sounds/
sudo chmod -R 755 /var/lib/asterisk/sounds/
sudo rm /tmp/bienvenue.mp3 /tmp/menu_*.mp3
```

---

## 3️⃣ Configuration des extensions SIP (`pjsip.conf`)

```bash
sudo nano /etc/asterisk/pjsip.conf
```

```ini
[transport-udp]
type = transport
protocol = udp
bind = 0.0.0.0:5060

; ============================================
; STANDARD TÉLÉPHONIQUE
; ============================================
[6000]
type = endpoint
context = internes
disallow = all
allow = ulaw
auth = auth6000
aors = 6000

[auth6000]
type = auth
auth_type = userpass
password = test123
username = 6000

[6000]
type = aor
max_contacts = 1

; ============================================
; SERVICE SCOLARITÉ
; ============================================
[6001]
type = endpoint
context = internes
disallow = all
allow = ulaw
auth = auth6001
aors = 6001

[auth6001]
type = auth
auth_type = userpass
password = scolarite123
username = 6001

[6001]
type = aor
max_contacts = 1

; ============================================
; SERVICE COMPTABILITÉ
; ============================================
[6002]
type = endpoint
context = internes
disallow = all
allow = ulaw
auth = auth6002
aors = 6002

[auth6002]
type = auth
auth_type = userpass
password = finance123
username = 6002

[6002]
type = aor
max_contacts = 1

; ============================================
; SERVICE ADMINISTRATION
; ============================================
[6003]
type = endpoint
context = internes
disallow = all
allow = ulaw
auth = auth6003
aors = 6003

[auth6003]
type = auth
auth_type = userpass
password = admin123
username = 6003

[6003]
type = aor
max_contacts = 1
```

---

## 4️⃣ Configuration du standard IVR (`extensions.conf`)

```bash
sudo nano /etc/asterisk/extensions.conf
```

```ini
[general]
static=yes
writeprotect=no

[globals]

; ============================================
; CONTEXTE PRINCIPAL - STANDARD TÉLÉPHONIQUE
; ÉCOLE POLYTECHNIQUE DE SOUSSE
; ============================================
[ivr-menu]
exten => 6000,1,NoOp(Appel entrant vers le standard)
same => n,Answer()
same => n,Wait(1)
same => n(choix_langue),Background(fr/choix-langue)
same => n,WaitExten(5)

; Choix de la langue
exten => 1,1,NoOp(Langue: English)
same => n,Set(CHANNEL(language)=en)
same => n,Goto(main-menu,s,1)

exten => 2,1,NoOp(Langue: Arabe)
same => n,Set(CHANNEL(language)=ar)
same => n,Goto(main-menu,s,1)

exten => 3,1,NoOp(Langue: Francais)
same => n,Set(CHANNEL(language)=fr)
same => n,Goto(main-menu,s,1)

; Option invalide
exten => i,1,NoOp(Option invalide)
same => n,Playback(pbx-invalid)
same => n,Goto(choix_langue)

; Temps dépassé
exten => t,1,NoOp(Temps depasse)
same => n,Playback(vm-goodbye)
same => n,Hangup()

; ============================================
; MENU PRINCIPAL (3 SERVICES)
; ============================================
[main-menu]
exten => s,1,NoOp(Menu principal - Choix du service)
same => n,Background(${CHANNEL(language)}/menuprincipal)
same => n,WaitExten(5)

; Choix 1: Scolarité
exten => 1,1,NoOp(Redirection vers Scolarite)
same => n,Playback(transfer)
same => n,Dial(PJSIP/6001,30)
same => n,VoiceMail(6001@default,u)
same => n,Goto(s,1)

; Choix 2: Comptabilité
exten => 2,1,NoOp(Redirection vers Comptabilite)
same => n,Playback(transfer)
same => n,Dial(PJSIP/6002,30)
same => n,VoiceMail(6002@default,u)
same => n,Goto(s,1)

; Choix 3: Administration
exten => 3,1,NoOp(Redirection vers Administration)
same => n,Playback(transfer)
same => n,Dial(PJSIP/6003,30)
same => n,VoiceMail(6003@default,u)
same => n,Goto(s,1)

; Option invalide
exten => i,1,NoOp(Option invalide)
same => n,Playback(pbx-invalid)
same => n,Goto(s,1)

; Temps dépassé
exten => t,1,NoOp(Temps depasse)
same => n,Playback(vm-goodbye)
same => n,Hangup()

; ============================================
; CONTEXTE INTERNE (appels entre services)
; ============================================
[internes]
exten => _600[1-3],1,NoOp(Appel vers service ${EXTEN})
same => n,Dial(PJSIP/${EXTEN},30)
same => n,VoiceMail(${EXTEN}@default,u)
same => n,Hangup()

exten => 6000,1,NoOp(Appel vers le standard)
same => n,Goto(ivr-menu,6000,1)

exten => _6XXX,1,NoOp(Appel interne)
same => n,GotoIf($["${CALLERID(num)}" = "${EXTEN}"]?self,1)
same => n,Dial(PJSIP/${EXTEN},30)
same => n,Hangup()

exten => self,1,NoOp(Appel vers soi-meme)
same => n,Playback(vm-goodbye)
same => n,Hangup()

; ============================================
; CONTEXTE DEFAULT
; ============================================
[default]
include => ivr-menu
```

---

## 5️⃣ Configuration de la messagerie vocale (`voicemail.conf`)

```bash
sudo nano /etc/asterisk/voicemail.conf
```

```ini
[general]
format=wav49|gsm|wav
attach=yes
serveremail=asterisk@polytechnicien.tn
emailsubject=New voicemail from ${VM_CALLERID} in mailbox ${VM_MAILBOX}
emailbody=Dear ${VM_NAME},\n\nYou have a new voicemail (${VM_DUR}) from \
${VM_CALLERID} in mailbox ${VM_MAILBOX}.\n\n--Asterisk
mailcmd=/usr/sbin/sendmail -t
skipms=3000
maxsilence=10
silencethreshold=128
maxlogins=3
sendvoicemail=yes

[zonemessages]
eastern=America/New_York|'vm-received' Q 'digits/at' IMp
central=America/Chicago|'vm-received' Q 'digits/at' IMp

[default]
6001 => 1234,Scolarite,sami.hamouda@polytechnicien.tn,,attach=yes|saycid=yes|delete=no
6002 => 1234,Comptabilite,sami.hamouda@polytechnicien.tn,,attach=yes|saycid=yes|delete=no
6003 => 1234,Administration,sami.hamouda@polytechnicien.tn,,attach=yes|saycid=yes|delete=no
```

---

## 6️⃣ Configuration des notifications email (Postfix)

**Installation de Postfix :**
```bash
sudo apt install -y postfix mailutils
# Pendant l'installation : choisir "Internet Site"
# Nom du système : polytechnicien.tn
```

**Configuration de Postfix :**
```bash
sudo nano /etc/postfix/main.cf
```

```ini
# Configuration SMTP Gmail
relayhost = [smtp.gmail.com]:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

**Authentification Gmail :**
```bash
sudo nano /etc/postfix/sasl_passwd
# Ajouter la ligne :
[smtp.gmail.com]:587 votre-email@gmail.com:mot-de-passe-application

sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

---

## 7️⃣ Rechargement de la configuration

```bash
# Recharger la configuration SIP
sudo asterisk -rx "module reload res_pjsip.so"

# Recharger le dialplan
sudo asterisk -rx "dialplan reload"

# Recharger la messagerie vocale
sudo asterisk -rx "module reload app_voicemail.so"

# Redémarrage complet
sudo systemctl restart asterisk
```

---

## 8️⃣ Démarrage et gestion du serveur

```bash
# Démarrer Asterisk
sudo systemctl start asterisk

# Arrêter Asterisk
sudo systemctl stop asterisk

# Redémarrer Asterisk
sudo systemctl restart asterisk

# Vérifier le statut
sudo systemctl status asterisk

# Se connecter à la console Asterisk
sudo asterisk -rvvv

# Vérifier les extensions connectées
sudo asterisk -rx "pjsip show endpoints"

# Vérifier les contacts enregistrés
sudo asterisk -rx "pjsip show contacts"

# Vérifier le transport
sudo asterisk -rx "pjsip show transports"

# Vérifier les utilisateurs de la messagerie
sudo asterisk -rx "voicemail show users"
```

---

## 9️⃣ Test avec Zoiper

**Configuration de Zoiper :**

| Champ | Valeur |
|---|---|
| Username | `6000` |
| Password | `test123` |
| Domain | `192.168.1.20` |
| Port | `5060` |
| Transport | `UDP` |

---

## 📋 Liste des comptes SIP

| Service | Extension | Username | Password |
|---|---|---|---|
| 📞 Standard (IVR) | 6000 | 6000 | `test123` |
| 🎓 Scolarité | 6001 | 6001 | `scolarite123` |
| 💰 Comptabilité | 6002 | 6002 | `finance123` |
| 🏛️ Administration | 6003 | 6003 | `admin123` |

---

<div align="center">

**École Polytechnique de Sousse** — Projet Standard Téléphonique Multilingue  
Asterisk 20.15.1 · Ubuntu Server 22.04 · SIP/PJSIP · IVR FR/EN/AR

</div>

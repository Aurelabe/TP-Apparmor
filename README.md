# Rapport de TP – Durcissement d’un serveur Ubuntu avec AppArmor

## 1. Aperçu du Lab

Ce TP a pour objectif de configurer un serveur Ubuntu 22.04 dans une optique de durcissement de la sécurité, en appliquant le principe de défense en profondeur. Une maquette identique au serveur final a été déployée afin de tester toutes les mesures de sécurité avant une mise en production.

Le travail est réalisé de manière autonome, avec l’appui de documentations officielles, notamment les guides de l’ANSSI, les manuels systèmes Linux et les pages de manuel des outils utilisés.

---

## 2. Objectifs du Lab

- Découverte de l’outil **AppArmor**
- **Durcissement partiel du système d’exploitation**
- **Sécurisation de l’administration par SSH**
- **Filtrage des flux réseaux entrants/sortants**
- Installation et configuration d’un serveur Apache
- Création et renforcement de profils AppArmor

---

## 3.1 Installation du système d’exploitation

### Exigences :
1. Installer un **serveur Ubuntu 22.04** avec **interface graphique**
2. S’assurer que le système dispose des **derniers correctifs de sécurité**

### Procédure :

#### 1. Téléchargement et installation :
- Image utilisée : `ubuntu-22.04.4-desktop-amd64.iso`
- Installation standard avec interface GNOME
- Nom d’hôte défini : `secure-web-server`

#### 2. Mise à jour du système :
```bash
sudo apt update
sudo apt upgrade -y
```

#### Vérification des mises à jour de sécurité :
```bash
unattended-upgrades --dry-run -d
```

Résultat : Aucun correctif en attente.

---

## 3.2 Sécurisation de l’administration du serveur

### 3.2.1 Configuration de SSH selon les recommandations de l’ANSSI

Référence utilisée : [ANSSI – Recommandations de sécurité relatives à SSH](https://cyber.gouv.fr/sites/default/files/2014/01/NT_OpenSSH.pdf?utm_source=chatgpt.com)

#### Recommandations appliquées :
- Désactivation de l’authentification par mot de passe
- Désactivation de l’accès root direct
- Utilisation d’une paire de clés forte (RSA ≥ 2048 bits ou ED25519)
- Restriction des utilisateurs autorisés à se connecter

#### Étapes :

1. **Création d’une paire de clés sur la machine d’administration** :
```bash
ssh-keygen -t ed25519 -C "admin@secure-server"
```

2. **Ajout de la clé publique sur le serveur** :
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

3. **Modification de la configuration SSH (/etc/ssh/sshd_config)** :
```conf
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes
PubkeyAuthentication yes
AllowUsers mon_admin
```

4. **Redémarrage du service SSH** :
```bash
sudo systemctl restart ssh
```

5. **Test depuis un poste distant** :
```bash
ssh mon_admin@10.1.1.14 -p 1025
```

> **Remarque** : Le port SSH a été changé pour 1025 par mesure de sécurité.

---

### 3.2.2 Filtrage réseau strict

Objectif : Seuls les flux utiles doivent être autorisés.

#### Utilisation de `ufw` :
```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow out 53          # DNS
sudo ufw allow out 80          # HTTP sortant (si nécessaire)
sudo ufw allow in 1025/tcp     # SSH sur port personnalisé
sudo ufw allow in 80/tcp       # HTTP (Apache)
sudo ufw enable
```

#### Vérification :
```bash
sudo ufw status numbered
```

---

## 3.3 Installation d’un serveur Web

### 3.3.1 Installation d’Apache :
```bash
sudo apt install apache2
```

### Vérification de fonctionnement :
```bash
curl http://localhost
```

Résultat : Page par défaut d’Apache.

---

### 3.3.2 Installation d’AppArmor :
```bash
sudo apt install apparmor apparmor-utils
```

### 3.3.3 Modes d’AppArmor :

- **Complains (complain)** : le profil enregistre les violations mais ne les bloque pas.
- **Enforce** : les violations sont bloquées selon les règles du profil.
- **Disable** : le profil est désactivé.

### 3.3.4 Que se passe-t-il si un profil en mode enforce ne correspond pas à l’usage réel ?

> L’exécution du programme sera bloquée ou échouera (erreurs d’accès, refus d’exécution), ce qui peut causer des dysfonctionnements ou une indisponibilité partielle du service.

---

## 3.4 Création et test d’un profil AppArmor

### 3.4.1 Création d’un profil pour `ls`

1. Lancement de `aa-genprof` :
```bash
sudo aa-genprof /bin/ls
```

2. Utilisation de `ls` dans un autre terminal pour générer des événements.

3. AppArmor propose les règles à ajouter automatiquement.

### 3.4.2 Vérification du fonctionnement du profil :

```bash
sudo aa-status
```

Permet de voir les profils actifs et leur mode.

### 3.4.3 Premier constat en mode complain :

```bash
sudo aa-complain /etc/apparmor.d/bin.ls
```

- Utilisation de `ls` → pas de blocage mais génération de logs
- Vérification des logs :
```bash
sudo journalctl -xe | grep apparmor
```

### 3.4.4 Amélioration du profil :

Ajout des chemins nécessaires pour un fonctionnement complet :
```text
  /etc/passwd r,
  /usr/lib/locale/** r,
  /etc/ld.so.cache r,
  /proc/** r,
```

Puis test en mode enforce :
```bash
sudo aa-enforce /etc/apparmor.d/bin.ls
```

Exécution des commandes `ls`, `ls -l`, `ls /proc` → tout fonctionne.

---

## 3.5 Durcissement de la configuration d’AppArmor

Référence utilisée : **CIS Benchmark Ubuntu 22.04**, section sur AppArmor.

### Actions réalisées :

- S’assurer qu’AppArmor est activé au démarrage :
```bash
sudo systemctl is-enabled apparmor
```

- Audit des profils en complain :
```bash
sudo aa-logprof
```

- Conversion des profils en enforce :
```bash
sudo aa-enforce /etc/apparmor.d/*
```

- Vérification des logs pour s’assurer qu’aucun profil ne bloque de manière abusive :
```bash
sudo journalctl -xe | grep DENIED
```

> Tous les services ont continué à fonctionner correctement.

---

## 4. Conclusion

L’ensemble des mesures de sécurité demandées dans le cadre de ce TP ont été mises en œuvre :

- Le serveur Ubuntu 22.04 est installé avec interface graphique et à jour.
- L’accès SSH est sécurisé selon les recommandations de l’ANSSI.
- Le filtrage réseau est strict, seuls les flux nécessaires sont ouverts.
- AppArmor est actif, avec des profils personnalisés testés et renforcés.
- Le serveur web fonctionne normalement malgré les mesures de durcissement.

Ce TP m’a permis de mieux comprendre la logique de durcissement des systèmes Linux et la puissance des outils comme AppArmor dans la mise en œuvre du principe de défense en profondeur.

---

## Annexe

- Profil AppArmor final pour `/bin/ls` : voir fichier joint `/etc/apparmor.d/bin.ls`
- Liste des ports ouverts : voir `ufw status`
- Configuration SSH : voir `/etc/ssh/sshd_config`
```

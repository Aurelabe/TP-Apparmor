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

![image](https://github.com/user-attachments/assets/9aaae35b-8079-490f-9fad-b9f97dfc026f)

---

## 3.2 Configuration de l’adresse IP statique via `nmcli`

Afin d'attribuer une adresse IP statique à notre serveur, l'outil `nmcli` a été utilisé.

#### Étapes :

1. **Vérification de la connexion réseau actuelle** :
```bash
nmcli con show
```

![image](https://github.com/user-attachments/assets/47eb1364-1a8e-4456-aaf2-9f50f1c00251)


2. **Modification de la connexion réseau** pour définir une IP statique :
```bash
nmcli conn mod "Connexion filaire 1" con.id enp0s3
nmcli conn mod "Connexion filaire 2" con.id enp0s8
nmcli con mod enp0s8 ipv4.addresses 10.1.1.14/24
nmcli con mod enp0s8 ipv4.gateway 10.1.1.1
nmcli con modify enp0s8 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con mod enp0s8 ipv4.method manual
```

3. **Redémarrage de la connexion réseau pour appliquer les modifications** :
```bash
nmcli connection down enp0s8 && nmcli connection up enp0s8
```

4. **Vérification de la nouvelle configuration IP** :
```bash
ip a
```
![image](https://github.com/user-attachments/assets/52e18049-8163-400d-90af-f6f5ce996a7f)

L'adresse IP statique du serveur est désormais configurée sur `10.1.1.14`.

---

## 3.3 Sécurisation de l’administration du serveur

### 3.3.1 Configuration de SSH selon les recommandations de l’ANSSI

Référence utilisée : [ANSSI – Recommandations de sécurité relatives à SSH](https://cyber.gouv.fr/sites/default/files/2014/01/NT_OpenSSH.pdf?utm_source=chatgpt.com)

#### Recommandations appliquées :
- Désactivation de l’authentification par mot de passe
- Désactivation de l’accès root direct
- Utilisation d’une paire de clés forte (RSA ≥ 2048 bits ou ED25519)
- Restriction des utilisateurs autorisés à se connecter

#### Étapes :

1. **Création d’une paire de clés sur la machine d’administration** :
```bash
ssh-keygen -t ed25519 -C "aurel@secure-server"
```

2. **Ajout de la clé publique sur le serveur** :
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
**Copie de la clé à partir de la machine génératrice** :
```bash
scp C:\Users\Aurel/.ssh/id_ed25519.pub aurel@secure-web-server:~/.ssh/authorized_keys
```

3. **Modification de la configuration SSH (/etc/ssh/sshd_config)** :
```conf
Port 1025
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
ssh -i C:\Users\Aurel/.ssh/id_ed25519 -p 1025 aurel@secure-web-server
```

> **Remarque** : Le port SSH a été changé pour 1025 par mesure de sécurité.

---

### 3.3.2 Filtrage réseau strict

Objectif : Seuls les flux utiles doivent être autorisés.

#### Utilisation de `ufw` :
```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow out 53          # DNS
sudo ufw allow out 80          # HTTP sortant
sudo ufw allow in 1025/tcp     # SSH sur port personnalisé
sudo ufw allow in 80/tcp       # HTTP (Apache)
sudo ufw enable
```

#### Vérification :
```bash
sudo ufw status numbered
```

![image](https://github.com/user-attachments/assets/ccfa11d0-2235-473c-8e61-f0c6c7e81125)


---

## 3.4 Installation d’un serveur Web

### 3.4.1 Installation d’Apache :
```bash
sudo apt install apache2
```

### Vérification de fonctionnement :
```bash
curl http://localhost
```

**Résultat** : Page par défaut d’Apache.

---

### 3.4.2 Installation d’AppArmor :
```bash
sudo apt install apparmor apparmor-utils
```

### 3.4.3 Modes d’AppArmor :

- **Complains (complain)** : le profil enregistre les violations mais ne les bloque pas.
- **Enforce** : les violations sont bloquées selon les règles du profil.
- **Disable** : le profil est désactivé.

### 3.4.4 Que se passe-t-il si un profil en mode enforce ne correspond pas à l’usage réel ?

> L’exécution du programme sera bloquée ou échouera (erreurs d’accès, refus d’exécution), ce qui peut causer des dysfonctionnements ou une indisponibilité partielle du service.

---

## 3.5 Création et test d’un profil AppArmor

### 3.5.1 Création d’un profil squelette AppArmor pour le binaire « ls »

Lors de la création d'un profil AppArmor pour le programme `/bin/ls`, l'objectif était de définir les permissions et restrictions d'accès nécessaires à son bon fonctionnement tout en renforçant la sécurité du système. Cependant, une difficulté est survenue en raison du marquage du programme `ls` comme devant être exempté de la création de profils dans la configuration par défaut d'AppArmor.

#### Étape 1 : Blocage dû au marquage par défaut du programme

Lorsque la commande suivante a été exécutée pour générer le profil pour `ls` :

```bash
sudo aa-genprof /bin/ls
```

Une erreur est survenue, indiquant que le programme `/usr/bin/ls` était actuellement marqué comme un programme ne devant pas avoir son propre profil. Ce marquage fait partie de la configuration par défaut d'AppArmor, qui exclut certains programmes systèmes jugés essentiels pour le bon fonctionnement du système. Cette restriction est mise en place pour éviter des interruptions potentielles dans le comportement des programmes critiques.

Le message d'erreur reçu était le suivant :

```
/usr/bin/ls is currently marked as a program that should not have its own profile.
```

Cette restriction empêche la création automatique d'un profil via l'outil `aa-genprof`.

#### Étape 2 : Modification de la configuration d'AppArmor

Pour contourner cette limitation, il était nécessaire de modifier la configuration d'AppArmor afin de permettre la création d'un profil pour `ls`. Cela a été réalisé en éditant le fichier de configuration `/etc/apparmor/logprof.conf`. Ce fichier contient une section `[qualifiers]` qui spécifie les programmes exclus de la génération automatique de profils. En supprimant ou en commentant la ligne correspondant à `/usr/bin/ls`, il est possible d'autoriser la création du profil pour ce programme.

Voici les étapes suivies pour effectuer cette modification :

1. Ouverture du fichier de configuration avec un éditeur de texte :

   ```bash
   sudo nano /etc/apparmor/logprof.conf
   ```

2. Recherche et suppression de la ligne correspondant à `ls` dans la section `[qualifiers]`, comme suit :

   ```text
   # /usr/bin/ls
   ```

3. Sauvegarde et fermeture du fichier (`Ctrl + S` pour sauvegarder et `Ctrl + X` pour quitter).

#### Étape 3 : Création du profil

Après avoir modifié la configuration d'AppArmor, il était possible de réexécuter la commande `aa-genprof /bin/ls` pour générer un profil pour `ls`. L'outil a alors guidé l'utilisateur à travers un processus interactif permettant de définir les permissions nécessaires au bon fonctionnement de `ls`.

#### Étape 4 : Application du profil

Une fois le profil généré, il a été appliqué à l'aide de la commande suivante :

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.ls
```

Cette commande a permis de recharger et appliquer immédiatement le profil pour le programme `ls`.

#### Étape 5 : Problème d'exécution de `ls`

Après avoir appliqué le profil, une tentative d'exécution de la commande `ls` a échoué, renvoyant l'erreur suivante :

```
ls: impossible d'ouvrir le répertoire '.': Permission non accordée
```

Ce problème est survenu car les règles définies dans le profil pour `/usr/bin/ls` étaient trop restrictives, notamment concernant les permissions d'accès aux répertoires. Le programme n'a pas pu accéder au répertoire courant, ce qui a causé l'échec de la commande.

#### Étape 6 : Résolution du problème d'accès

Pour résoudre ce problème, il a été nécessaire de modifier le fichier du profil pour accorder à `ls` les permissions nécessaires à l'accès en lecture des répertoires.

Voici les étapes suivies pour effectuer cette modification :

1. **Édition du profil AppArmor** :
   Pour permettre au binaire `ls` de fonctionner correctement, le fichier du profil pour `/usr/bin/ls` a été ouvert avec un éditeur de texte :

   ```bash
   sudo nano /etc/apparmor.d/usr.bin.ls
   ```

2. **Ajout d'une règle pour autoriser l'accès aux répertoires** :
   La ligne `/** r,` a été ajoutée pour autoriser l'accès en lecture à tous les répertoires. Le fichier modifié ressemble maintenant à ceci :

   ```bash
   # Last Modified: Tue Apr  8 20:18:52 2025
   abi <abi/3.0>,

   include <tunables/global>

   /usr/bin/ls {
     include <abstractions/base>

     /usr/bin/ls mr,

     /** r,
   }
   ```

3. **Sauvegarde et sortie** :
   Après avoir effectué cette modification, le fichier a été sauvegardé en appuyant sur `Ctrl + S`, puis l'éditeur a été quitté avec `Ctrl + X`.

4. **Rechargement du profil AppArmor** :
   Le profil a été rechargé à l'aide de la commande suivante pour appliquer immédiatement les changements :

   ```bash
   sudo apparmor_parser -r /etc/apparmor.d/usr.bin.ls
   ```

5. **Test du binaire** :
   Enfin, la commande `ls` a été testée pour vérifier que l'erreur d'accès aux répertoires avait disparu :

   ```bash
   ls
   ```

   Après cette modification, la commande `ls` a fonctionné correctement, et les erreurs d'accès ont été résolues.


### 3.5.2 Vérification du fonctionnement du profil :

```bash
sudo aa-status
```

Permet de voir les profils actifs et leur mode.

![image](https://github.com/user-attachments/assets/c057c31d-1bb6-4abf-81b5-9a8972d423f7)

`ls` fait partir des profiles en mode `enforce`

### 3.5.3 Premier constat en mode complain :

```bash
sudo aa-complain /etc/apparmor.d/bin.ls
```

- Utilisation de `ls` → pas de blocage mais génération de logs
- Vérification des logs :
```bash
sudo journalctl -xe | grep apparmor
```

### 3.5.4 Amélioration du profil :

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

## 3.6 Durcissement de la configuration d’AppArmor

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

---

N'hésite pas à me dire si tu as d'autres ajustements à faire.

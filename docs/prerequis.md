# Configuration des postes avant déploiement

Cette phase est essentielle pour garantir le bon fonctionnement de chaque poste. Avant de commencer, il est important de respecter le plan d’adressage défini dans la section [Infrastructure](./infra.md), afin d’éviter tout conflit d’adresses IP. 

## Configuration d'une adresse IP statique sur chaque machine
Chaque machine de la DMZ doit être configurée avec une adresse IP statique pour assurer une connectivité stable.
### 1. Les étapes pour accèder à la configuration réseau :
1. **Se connecter sur la machine au switch dédié** dans GNS3.
2. **Configurer la machine manuellement :**
    - **Clic droit sur la machine** dans GNS3 → **Configure**.
    - Aller dans l’onglet **General settings**.
    - Descendre jusqu’à **Network configuration** et cliquer sur **Edit**.
### 2. Modification du fichier de configuration réseau
**Ouvrir le fichier de configuration réseau** et activer l’IP statique, comme dans l'exemple ci-dessous :
```conf
iface eth0 inet static
	address 10.10.10.2         # Adresse IP de la machine
	netmask 255.255.255.240    # Masque de sous-réseau
	gateway 10.10.10.1         # Passerelle dédié
    # Le dns qui lui correspond (DMZ-DNS ici)
	up echo "nameserver 10.10.10.4" > /etc/resolv.conf 
	up echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

**Remarque :** Si un serveur DNS interne est utilisé, remplacer le dns préféré par `172.16.50.2` (IP du serveur DNS local) et `8.8.8.8`.

### 3. Appliquer les modifications et tester la connexion

1. **Redémarrer l’interface réseau** avec :
    ```bash
    ip link set eth0 down
    ip link set eth0 up
    ```
    
2. Vérifier l’assignation IP :
    ```bash
    ip a
    ```
    
3. **Tester la connectivité** :
    ```bash
    ping 10.10.10.1    # Tester la communication avec la passerelle
    ping 8.8.8.8         # Tester la connectivité Internet
    ```

### 5. Répéter l’opération pour les autres machines
- Modifier l’**adresse IP** pour chaque machine en respectant la plage définie.
- Assigner une **IP unique** pour éviter les conflits.
- Vérifier que chaque machine peut **communiquer avec la passerelle et les autres équipements**.

---

## Rendre la configuration persistante
Dans GNS3, les conteneurs Docker sont par défaut éphémères. Cela signifie que toute modification effectuée à l’intérieur (installation de paquets, fichiers créés, configurations réseau…) est perdue au redémarrage si elle n’est pas explicitement sauvegardée.

**Activer la persistance**, c’est dire à GNS3 quels dossiers doivent être conservés même après arrêt ou redémarrage du conteneur.

1. Faire un clic droit sur la machine, ensuite cliquer sur "**Configure**"
2. Dans l'onglet "**Advanced**" nous avons le second champ qui nous renseigne : "_Additional directories to make persistent that are not included in the image VOLUMES config. One directory per line._"
3. Ajouter les répertoires suivants :
```
/bin
/boot
/etc
/gns3
/home
/lib
/lib64
/media
/mnt
/opt
/root
/run
/sbin
/srv
/tmp
/usr
/var
```

---

## Installation OpenSSH Server
Configurer l'accès SSH sur chaque machine pour permettre l’automatisation via Ansible, de manière sécurisée et persistante.

### 1. Ouvrir la console du conteneur dans GNS3
1. Lancer le conteneur dans GNS3.
2. Faire un clic droit > **Console**.
3. Le shell s'ouvre avec utilisateur par défaut `root` directement.
### 2. Installation du serveur SSH
Depuis la console du conteneur entrer les commandes suivantes pour installer le service SSH (fichier binaire `sshd`) et les dépendances :
```sh
apt update 
apt install openssh-server -y
```
### 3. Vérifier (et démarrer) le service SSH

```sh
service ssh start
```

---
### 4. Attribuer un mot de passe `root`
Souvent dans Docker, le compte `root` n’a pas de mot de passe, pour éviter des erreurs et des problèmes avec ssh et ansible alors on modifie cela. Pour en mettre un :
```sh
passwd root
```

Il nous sera demandé de saisir puis confirmer un mot de passe.

### 5. Créer un un autre utilisateur pour Ansible
#### Pourquoi ?
Le fait d’autoriser une connexion SSH directe en tant que **root** est généralement considéré comme moins sécurisé pour plusieurs raisons :

- **Surface d’attaque plus large** : S’il est compromis l’attaquant dispose immédiatement de tous les droits.
- **Cible évidente** : brute force sur utilisateur "**root**".
- **Moins de traçabilité** : Dans le journal des logs seul l'utilisateur `root` est utilisé, ce qui ne permet pas d'identifier qui a effectuer quelles actions.

#### Créer un utilisateur standard avec droits `sudo`
```sh
adduser ansible
usermod -aG sudo ansible
```

### Désactiver l’authentification `root` par mot de passe
Dans le fichier de configuration SSH (`/etc/ssh/sshd_config`) dé-commenter la ligne suivante et la modifier pour **interdire complètement** la connexion SSH directe de root (même via une clé) :
```
PermitRootLogin no
```

Cela force l’utilisation d’un autre utilisateur pour la connexion SSH, ou impose l’usage d’une clé SSH s’il faut malgré tout se connecter en `root`.

Redémarrer le service :
```sh
service ssh restart
```

---

## Installation `sudo`
Par défaut, dans beaucoup de conteneurs Docker (comme ceux utilisés dans GNS3), sudo n’est pas installé car on y est souvent connecté directement en tant que root. Mais pour des raisons de sécurité et de bonnes pratiques, nous avons choisi de :

- Créer un utilisateur non-root (ansible)
- Désactiver l'accès SSH direct au compte `root`

Mais l'utilisateur aura besoin de droits administratifs. C’est là qu’intervient `sudo`, il permet à un utilisateur standard (comme `ansible`) d'exécuter des commandes avec les privilèges `root`, sans devoir se connecter en `root`.

#### On installe sudo sur la machine distante :

```sh
apt install sudo 
```

#### On édite la configuration sudo via `visudo` sur la machine distante :
```bash
visudo
```

#### Ajouter la ligne suivante à la fin du fichier :
```bash
ansible ALL=(ALL) NOPASSWD: ALL
```
Cela permet à Ansible d’exécuter des commandes sudo sans mot de passe.

---

## Configuration SSH 
### À quoi ça sert exactement ce fichier `~/.ssh/config` ?
Ce fichier permet de **préconfigurer des connexions SSH**, en donnant un **nom court (alias)** à chaque machine, avec :
- son adresse IP ou nom de domaine (`HostName`)
- l’utilisateur par défaut (`User`)
- la clé à utiliser (`IdentityFile`)
- et plein d’autres options utiles (port, compression, etc.)

### 1. Créé et remplir le fichier `config` 
J'écris dans `~/.ssh/config` (sur la machine **IT-ANSIBLE**) :

```config
Host dmz-smtp
  HostName 10.10.10.2
  User ansible
  IdentityFile ~/.ssh/id_ed25519

Host dmz-dns
  HostName 10.10.10.3
  User ansible
  IdentityFile ~/.ssh/id_ed25519

Host dmz-web
  HostName 10.10.10.4
  User ansible
  IdentityFile ~/.ssh/id_ed25519

...
```

### 2. Vérifier les permissions
SSH exige que ce fichier ait les bonnes permissions :
```sh
chmod 600 ~/.ssh/config
```

### Sans ce fichier
On doit taper la commande **longue** à chaque fois :
```bash
ssh -i ~/.ssh/id_ed25519 ansible@10.10.10.2
```
### Avec ce fichier
On peut taper :
```bash
ssh dmz-smtp
```

SSH saura automatiquement :
- Quelle IP utiliser
- Quel utilisateur
- Quelle clé utiliser

---

## Script de démarrage automatique pour les conteneurs
Le script suivant est utilisé pour initialiser automatiquement certains services (comme SSH, Grafana, Prometheus) et s'assurer que les droits sur sudo sont correctement configurés à chaque démarrage du conteneur.
```sh
#!/bin/sh
# run docker startup, first arg is new PATH, remainder is command

PATH="$1"         # Le premier argument est utilisé comme nouvelle valeur de la variable d’environnement PATH
shift             # On décale les arguments pour exécuter la vraie commande après

# S'assurer que sudo appartient à root et a les bons droits
chown root:root /usr/bin/sudo 
chmod 4755 /usr/bin/sudo

# Démarrer les services nécessaires

# Lancer la commande par défaut du conteneur
exec "$@"
```

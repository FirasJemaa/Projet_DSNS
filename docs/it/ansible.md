# IT-ANSIBLE (`it-ansible.itway.local`)

[Playbook Ansible complet](https://github.com/FirasJemaa/Projet_DSNS_playbook)

## Objectif

Configurer et administrer la machine **IT-ANSIBLE** (`it-ansible.itway.local`), en tant que **serveur d’automatisation Ansible** pour déployer, configurer et maintenir les services de l'infrastructure IT-WAY de manière centralisée, sécurisée et reproductible.

---

## 1. Préparation initiale

### 1.1. Configuration réseau en DHCP

[Configurer le conteneur en adresse IP comme on le montre ici](../prerequis.md).

### 1.2. Installation des paquets nécessaires

```bash
apt update
apt install ansible sshpass python3-apt -y
```

* `ansible` : Le cœur du système. Il s'agit de l’outil principal d'automatisation
* `sshpass` : Permet à Ansible (ou ssh) d’utiliser un mot de passe en ligne de commande.
* `python3-apt` : Librairie Python permettant d’utiliser les modules APT d’Ansible pour Debian/Ubuntu.

---

## 2. Sécurisation SSH avec clés

### 2.1. Génération d’une clé SSH
On génère une clé SSH pour permettre à Ansible de se connecter de manière sécurisée et automatisée aux machines distantes sans avoir à entrer de mot de passe à chaque fois.

La commande suivante permet de créé la clé : 

```bash
ssh-keygen -t ed25519
```

* Chemin par défaut (`~/.ssh/id_ed25519`)
* Sans passphrase

### 2.2. Diffusion de la clé publique
On diffuse la clé SSH publique pour permettre à la machine distante de reconnaître la machine IT-ANSIBLE comme un client autorisé à se connecter sans mot de passe.

Voici la commande qui permet de copié la clé sur les autres machines : 
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@<IP>
```

⚠️ Si problème de "No route to host" :

* Ajouter une route statique dans **CORE-RT** (ex. vers DMZ)
* Vérifier les ACL et les objets réseaux sur l’interface de configuration

### 2.3. Configuration simplifiée avec `~/.ssh/config`
C’est le fichier de configuration client SSH, qui permet de préconfigurer les options de connexion pour chaque hôte distant.

```text
Host dmz-smtp
  HostName 10.10.10.3
  User ansible
  IdentityFile ~/.ssh/id_ed25519
```

---

## 3. Arborescence Ansible recommandée
Ansible recommande de structurer un projet en rôles, avec une arborescence comme celle-ci :
```text
project/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── roles/
│   ├── webserver/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   ├── files/
│   │   └── vars/
├── playbooks/
│   ├── site.yml
│   └── install-web.yml
```

[Consulter le site Ansible ici.](https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html)

### Détail des composants :

* `ansible.cfg` : configuration globale (inventaire, user par défaut, etc.)
* `inventory/hosts.ini` : liste des hôtes par groupe
* `group_vars/` : variables propres à chaque groupe ou machine
* `roles/` : chaque dossier représente un service avec :
    * `tasks/` :  Contient les tâches principales à exécuter (souvent dans main.yml)
    * `handlers/` :  Actions déclenchées par notification (après modification)
    * `templates/` :  Fichiers de configuration dynamiques avec variables Jinja2 (*.j2)
    * `files/` : Fichiers statiques copiés tels quels sur la machine cible
    * `vars`: Variables internes au rôle (priorité haute)
* `playbooks/` : chaque fichier YAML lance un ou plusieurs rôles sur une cible spécifique

---

## 4. Ansible Vault – Gestion des secrets

### 4.1 À quoi sert Vault ?

Ansible Vault permet de **chiffrer** des informations sensibles dans nos fichiers :

* Mots de passe
* Clés API
* Identifiants SMTP, LDAP, etc.

Cela évite de stocker en clair des secrets dans `group_vars/*.yml` ou dans les playbooks.

### 4.2 Créer un fichier chiffré

```bash
ansible-vault create group_vars/srv-mail.yml
```

### 4.3 Chiffrer un fichier

```bash
ansible-vault encrypt group_vars/srv-mail.yml
```

* Le fichier devient chiffré avec AES256
* Ansible nous demandera un mot de passe pour le déchiffrer à l’exécution

### 4.4. Modifier un fichier chiffré

```bash
ansible-vault edit group_vars/srv-mail.yml
```

### 4.5. Déchiffrer un fichier (temporairement)

```bash
ansible-vault view group_vars/srv-mail.yml
```
> Sert dans des cas précis où on veut consulter ou modifier temporairement un secret chiffré, sans le laisser en clair durablement.

### 4.6. Lancer un playbook avec Vault

```bash
ansible-playbook setup-mail.yml --ask-vault-pass
```

Ou utiliser un fichier de mot de passe :

```bash
ansible-playbook setup-mail.yml --vault-password-file ~/.vault_pass.txt
```

---

## 5. Bonnes pratiques générales

* Organiser les rôles et variables par service
* Ne jamais dupliquer des informations : centralisez via `group_vars/`
* Éviter les `shell:`/`command:` au profit de modules dédiés
* Toujours chiffrer les secrets avec Vault
* Tester les rôles avec des cibles de test (ex. VMs ou containers)
* Maintenir un inventaire clair, documenté et versionné

---

## 6. Exemple de commande typique

```bash
ansible-playbook playbooks/setup-mail.yml -i inventory/hosts.ini --ask-vault-pass
```

--- 

## Conclusion
Ansible s’avère être un outil extrêmement pratique et puissant pour gérer une infrastructure informatique moderne. Grâce à sa capacité à automatiser le déploiement, la configuration et la maintenance des serveurs, il permet non seulement de gagner un temps précieux, mais aussi d’éviter les erreurs humaines et d’assurer une homogénéité dans les environnements.

En cas de faille de sécurité, de corruption système ou même de perte totale d’un serveur, Ansible permet de réinstaller et reconfigurer une machine en quelques minutes à partir de zéro, simplement en relançant les bons playbooks. Cela garantit une résilience opérationnelle et une réduction drastique du temps d’indisponibilité.

De plus, son intégration avec Ansible Vault permet de gérer en toute sécurité les données sensibles (mots de passe, clés, tokens…), ce qui renforce la sécurité globale de l’infrastructure.
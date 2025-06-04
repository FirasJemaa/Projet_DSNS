# **SRV-MAIL** (`srv-mail.itway.local `)

[Playbook Ansible complet](playbookici)

## 1. Objectifs

Le serveur **SRV-MAIL** agit comme **MDA (Mail Delivery Agent)** et **serveur IMAP/POP**.
Il est chargé de :

* Stocker les mails des utilisateurs dans des boîtes aux lettres locales,
* Permettre leur consultation via IMAP/POP (par des clients comme Thunderbird),
* Recevoir les messages transmis par la passerelle SMTP externe (DMZ-SMTP),
* Centraliser la gestion des utilisateurs via une base MariaDB.

## 2. Présentation du service

Le rôle de SRV-MAIL est d’assurer la **réception, la consultation et l’archivage sécurisé** des mails internes à l’entreprise.
Il comprend plusieurs briques essentielles :

| Service          | Rôle                                                          |
| ---------------- | ------------------------------------------------------------- |
| **Postfix**      | MDA local (réception et distribution de mail vers les boîtes) |
| **Dovecot**      | Serveur IMAP/POP (consultation des mails par les clients)     |
| **MariaDB**      | Stockage des comptes, domaines, et mots de passe              |
| **PostfixAdmin** | Interface web d’administration pour Postfix et Dovecot        |

Le serveur n’envoie **pas de mails directement vers l’extérieur**. Ce rôle est délégué au [**serveur DMZ-SMTP**](../dmz/smtp.md).

### 2.1 Qu’est-ce qu’un serveur mail ?
Un **serveur mail** est une machine qui gère l’envoi, la réception, la distribution et la consultation des courriels.

On distingue **plusieurs rôles** :

| Rôle                            | Description                                                         |
| ------------------------------- | ------------------------------------------------------------------- |
| **MTA** (*Mail Transfer Agent*) | Transfère les mails entre serveurs (SMTP)                           |
| **MDA** (*Mail Delivery Agent*) | Dépose les mails dans les boîtes locales                            |
| **IMAP/POP**                    | Permet aux clients (Thunderbird, Outlook…) de consulter leurs mails |

---

### 2.2 Les services utilisés dans le projet

#### 1. **Postfix** (MDA)

* **Fonction** : reçoit les mails (en provenance de DMZ-SMTP) et les dépose dans les boîtes locales sur `/home/vmail/`.
* **Il agit aussi comme MTA (envoi sortant)**, mais **dans le projet, ce rôle est délégué à DMZ-SMTP**.
* **Fichier principal** : `/etc/postfix/main.cf`

**Ports utilisés** :

| Port | Protocole  | Utilisation                                           | TLS                    |
| ---- | ---------- | ----------------------------------------------------- | ---------------------- |
| 25   | SMTP       | Standard MTA (inbound mail)                           | Pas de TLS obligatoire |
| 465  | SMTPS      | SMTP chiffré directement                              | TLS dès la connexion   |

> Postfix est surtout sollicité **en interne** (port 25 entre DMZ-SMTP → SRV-MAIL).

---

#### 2. **Dovecot** (IMAP/POP)

* **Fonction** : permet aux utilisateurs de **lire leurs mails** (messages déposés par Postfix)
* Supporte :
       * **IMAP** (synchronisation des mails sur plusieurs appareils)
       * **POP3** (téléchargement local des mails, sans conservation sur le serveur)

**Ports utilisés** :

| Port | Protocole | Utilisation          | TLS      |
| ---- | --------- | -------------------- | -------- |
| 143  | IMAP      | IMAP classique       | STARTTLS |
| 993  | IMAPS     | IMAP avec TLS direct | TLS      |
| 110  | POP3      | POP non sécurisé     | Aucun    |
| 995  | POP3S     | POP sécurisé         | TLS      |

> Dans une config sécurisée, **on privilégie IMAP sur TLS (993)**.

---

#### 3. **MariaDB** (Backend des comptes mails)

* **Fonction** : stocke les utilisateurs, mots de passe, domaines, etc.
* Utilisé par :
       * Postfix (pour savoir "cet utilisateur existe-t-il ?")
       * Dovecot (pour valider l’authentification IMAP)
       * PostfixAdmin (interface de gestion)

> Les mots de passe y sont stockés **hachés** (SHA512 ou MD5).

---

#### 4. **PostfixAdmin** (interface Web PHP)

* **Fonction** : permet à l’admin de :

  * créer des utilisateurs
  * gérer les domaines
  * ajouter des redirections, alias, etc.
* **Fonctionne avec Apache2** ou Nginx, en PHP

---

### 2.3 Schéma simplifié de fonctionnement

```
+----------------+           +-------------+            +--------------+
| Utilisateur    |  IMAP     |   Dovecot   |            |              |
| (Thunderbird)  +---------> | (IMAP/POP)  | <--------+ | /home/vmail/ |
+----------------+           +-------------+            +--------------+
                                                             ▲
                                                             |  
                                                   (Maildir = boîtes locales)
                                                             ▲
+------------------+    SMTP     +------------+              |
| DMZ-SMTP         +-----------> |  Postfix   +--------------+
| (relais d’entrée)|             |  (MDA)     |
+------------------+             +------------+
                                        |
                          Utilise MySQL/MariaDB pour comptes
```

---

## 3. Rôle dans l'infrastructure ITWay
Dans l’infrastructure **ITWay**, le **serveur SRV-MAIL** joue un rôle **essentiel dans la gestion des courriels internes**. Voici une explication claire de **ce qu’il fait précisément** :

1. **Réception des mails internes et externes**
       * Il reçoit les mails **depuis DMZ-SMTP** (le relais en DMZ),
       * Et il les distribue aux boîtes mail des utilisateurs.
2. **Stockage des messages**
       * Les mails sont **conservés en base de données** et accessibles à tout moment via **IMAP sécurisé (port 993)**.
3. **Distribution aux clients mail**
       * Il sert les mails aux clients (Roundcube) via **Dovecot** (IMAP),
       * Les utilisateurs accèdent à leurs mails en toute sécurité, depuis leur poste.
4. **Gestion des utilisateurs et domaines**
       * Grâce à **PostfixAdmin**, l’administrateur peut créer/modifier :
              
              - les adresses mail,
              - les domaines,
              - les boîtes aux lettres virtuelles.
       
5. **Utilisation de bases SQL (MySQL/MariaDB)**
       * Les comptes et mots de passe sont **stockés dans une base sécurisée**,
       * Cela permet une gestion **souple, automatisée et évolutive** des utilisateurs.

---

## 4. Justification des choix

* **MTA/MDA** : Postfix (mode boîte aux lettres locales)
* **IMAP/POP** : Dovecot (accès aux messages)
* **Base de données** : MariaDB (authentification, boîtes)
* **Interface web** : PostfixAdmin
* **Outils** : Ansible pour l'automatisation

**Pourquoi ces choix ?**

* Compatibilité entre services (Postfix + Dovecot + PostfixAdmin),
* Open-source, documenté, standard,

---

## 5. Structure du projet

```plaintext
├── ansible.cfg
├── inventory/
│   ├── hosts.ini
|   └── group_vars/
|       └── srv-mail/
|           ├── main.yml
|           └── vault.yml
├── playbooks/
│   └── setup-mail.yml
└── roles/
    └── srv-mail/
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   ├── main.yml
        |   ├── apache_https.yml
        |   ├── cert.yml
        |   ├── dovecot-ldap.yml
        |   ├── dovecot-sql.yml
        |   ├── mysql_setup.yml
        |   └── postfixadmin.yml
        └── templates/
            ├── dovecot-ldap.conf.ext.j2 
            ├── dovecot-sql.conf.ext.j2
            ├── dovecot.conf.j2
            ├── postfixadmin-ssl.conf.j2
            ├── main.cf.j2
            └── master.cf.j2 
```

---

## 6. Ce que fait le Playbook Ansible
### Résumé global des tâches du rôle `srv-mail`

Le rôle `srv-mail` automatise l'installation complète d’un **serveur de messagerie sécurisé**, incluant Postfix, Dovecot, MariaDB, LDAP, PostfixAdmin et HTTPS.


### 1. Installation des paquets nécessaires

Les composants principaux :

* **Postfix** (serveur SMTP)
* **Dovecot** (serveur IMAP/POP3)
* **MariaDB** (base de données)
* **PostfixAdmin** (interface web de gestion)
* **Modules LDAP & SQL** pour l'authentification

---

### 2. Génération et déploiement du certificat SSL

* Création de la **clé privée** et de la **CSR** sur SRV-MAIL
* Envoi de la CSR vers SRV-PKI pour **signature**
* Récupération et déploiement du certificat **signé** + **CA**

> Permet de chiffrer les connexions (IMAPS/SMTPS/HTTPS)

---

### 3. Configuration de Postfix

* Déploiement des fichiers `main.cf` et `master.cf` via templates
* Redémarrage automatique de Postfix en cas de modification

---

### 4. Configuration de Dovecot

* Déploiement de `10-ssl.conf` (connexion sécurisée)
* Support de l'**authentification LDAP** via `dovecot-ldap.yml`
* *(Commenté)* Auth SQL si activé via `dovecot-sql.yml`
* Sécurisation du fichier `10-auth.conf` (authentification stricte)
* Ajout d’un socket partagé pour compatibilité avec PostfixAdmin

---

### 5. Configuration de MariaDB

* Installation de MariaDB + client + dépendance Python
* Lancement manuel en environnement conteneurisé (sans systemd)
* Création des bases de données et utilisateurs :

       * `roundcube` (webmail)
       * `postfixadmin` (interface admin)

---

### 6. Configuration de PostfixAdmin (interface web)

* Préparation du cache (`templates_c`)
* Lien symbolique vers le répertoire de cache
* Création de la base et de l'utilisateur SQL
* Activation de la configuration HTTPS via Apache :

       * Module SSL
       * Fichier `postfixadmin-ssl.conf`
       * Site Apache sécurisé activé (`a2ensite`)

---

## 7. Fichiers de configuration

### `dovecot-ldap.conf.ext.j2`
Ce fichier configure Dovecot pour l’authentification des utilisateurs via un annuaire LDAP.
Au lieu de stocker les utilisateurs localement ou dans une base SQL, Dovecot va interroger un serveur LDAP (comme Active Directory) pour :

vérifier les identifiants de connexion des utilisateurs,

récupérer des informations comme leur répertoire de mails (maildir) et leur UID/GID système.

Ce mode est particulièrement utilisé dans les infrastructures où l’identité des utilisateurs est centralisée (par ex. : un seul compte pour mail, VPN, intranet…).

#### Contenu du fichier
```ini
uris = {{ dovecot_ldap_uri }}

dn = {{ dovecot_ldap_bind_dn }}
dnpass = {{ dovecot_ldap_bind_pw }}

auth_bind = yes
ldap_version = 3

base = {{ dovecot_ldap_base }}
scope = subtree

user_attrs = \
  =home=/var/vmail/%d/%n, \
  =mail=maildir:/var/vmail/%d/%n/Maildir, \
  =uid=vmail, \
  =gid=vmail

user_filter = (&(objectClass=user)(sAMAccountName=%n))
pass_filter = (&(objectClass=user)(sAMAccountName=%n))
```

#### Description des paramètres
| Directive      | Description                                                                                                                                                          |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `uris`         | L'**adresse du serveur LDAP**, au format `ldap://<hôte>:<port>` (ex : `ldap://ldap.itway.local`) — c’est ici que Dovecot va se connecter pour interroger l’annuaire. |
| `dn`           | Le **Distinguished Name (DN)** du compte utilisé pour se connecter à LDAP (souvent un compte de service).                                                            |
| `dnpass`       | Le **mot de passe** associé au DN ci-dessus.                                                                                                                         |
| `auth_bind`    | Active la liaison dynamique : l’utilisateur LDAP s’authentifie avec ses propres identifiants (et non ceux du service).                                               |
| `ldap_version` | Version du protocole LDAP utilisée (3 = la plus récente, standard).                                                                                                  |
| `base`         | Le DN de base de recherche dans LDAP (ex : `dc=itway,dc=local`).                                                                                                     |
| `scope`        | Étendue de la recherche (`subtree` = recherche récursive dans tous les sous-niveaux du DN de base).                                                                  |
| `user_attrs`   | Attributs LDAP mappés vers les attributs internes de Dovecot (chemin maildir, UID, GID, etc.).                                                                       |
| `user_filter`  | Filtre LDAP pour retrouver l’utilisateur, basé sur son `sAMAccountName`. `%n` = login sans domaine.                                                                  |
| `pass_filter`  | Même chose, mais pour trouver le mot de passe correspondant.                                                                                                         |


---

### `dovecot.conf.j2`
Ce fichier permet de configurer la sécurité TLS/SSL de Dovecot, le serveur IMAP/POP3 utilisé pour accéder aux boîtes mail.
Il assure que toutes les connexions entre les clients de messagerie (Thunderbird, Roundcube…) et le serveur passent par une connexion chiffrée grâce à un certificat signé par une autorité de certification (CA).

> Sans ce fichier ou sans configuration SSL, les mots de passe des utilisateurs peuvent transiter en clair.

#### Contenu du fichier :

```ini
ssl = required
ssl_cert = <{{ ssl_dir }}/{{ crt_filename }}
ssl_key = <{{ ssl_dir }}/{{ key_filename }}
ssl_ca  = <{{ ssl_dir }}/{{ ca_filename }}
```

---

#### Description des paramètres :

|Directive|Description|
|---|---|
|`ssl = required`|Force l’utilisation de **SSL/TLS** pour toutes les connexions (empêche les connexions non sécurisées).|
|`ssl_cert`|Chemin vers le **fichier de certificat SSL/TLS** du serveur Dovecot, signé par la CA (`srv-mail.crt`).|
|`ssl_key`|Chemin vers la **clé privée** du certificat (`srv-mail.key`). Ce fichier doit rester confidentiel.|
|`ssl_ca`|Chemin vers le **certificat de l’autorité de certification (CA)** qui a signé le certificat serveur. Permet aux clients de **vérifier l’authenticité** de la connexion.|

---

### `main.cf.j2`
Ce fichier est le **fichier principal de configuration de Postfix**.  
Il définit :

- l’identité du serveur,
    
- comment il envoie/reçoit des e-mails,
    
- comment il gère la sécurité TLS/SSL,
    
- et quelles sont les restrictions d’accès pour éviter d’être utilisé comme relais ouvert (SPAM).
    
#### Contenu du fichier

```ini
myhostname = {{ mail_fqdn }}
mydomain = itway.local
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
relayhost = [mail.itway.fr]:25

# TLS
smtpd_tls_cert_file = {{ ssl_dir }}/{{ crt_filename }}
smtpd_tls_key_file  = {{ ssl_dir }}/{{ key_filename }}
smtpd_tls_CAfile    = {{ ssl_dir }}/{{ ca_filename }}
smtpd_use_tls = yes
smtpd_tls_security_level = encrypt

# Restrictions d'accès
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

# Sécurité supplémentaire
disable_vrfy_command = yes
smtpd_helo_required = yes
smtpd_tls_auth_only = yes
#smtpd_tls_loglevel = 1
tls_random_source = dev:/dev/urandom
```

---

#### Description des paramètres

|Directive|Description|
|---|---|
|`myhostname`|Nom FQDN du serveur mail (ex. `srv-mail.itway.local`).|
|`mydomain`|Domaine auquel appartient le serveur (souvent identique à `itway.local`).|
|`myorigin`|Domaine utilisé pour les mails sortants (ex : `user@itway.local`).|
|`inet_interfaces = all`|Postfix écoute sur toutes les interfaces réseau (utile pour VM/conteneurs).|
|`mydestination`|Liste des noms que le serveur considère comme locaux (il accepte les mails pour ces domaines).|
|`relayhost`| La directive `relayhost = [mail.itway.fr]:25` permet de rediriger tous les mails sortants du serveur interne vers le relais SMTP situé en DMZ (DMZ-SMTP), configuré en passerelle sécurisée vers Internet.
|

---

#### Section TLS

|Directive|Description|
|---|---|
|`smtpd_tls_cert_file`|Chemin vers le certificat SSL public.|
|`smtpd_tls_key_file`|Chemin vers la clé privée du serveur.|
|`smtpd_tls_CAfile`|Certificat de la CA utilisée pour valider les connexions.|
|`smtpd_use_tls`|Active le TLS pour les connexions entrantes (clients ou autres serveurs SMTP).|
|`smtpd_tls_security_level = encrypt`|Imposera le chiffrement (TLS) lors des échanges.|

---

#### Restrictions d'accès et sécurité 
|Directive|Description|
|---|---|
|`smtpd_relay_restrictions`|Règles de relais : n'autorise l'envoi de mails **qu'à partir du réseau local ou si l'utilisateur est authentifié**.|
|`disable_vrfy_command`|Désactive la commande SMTP `VRFY` pour empêcher la détection de comptes valides par les bots.|
|`smtpd_helo_required`|Oblige les clients SMTP à s'identifier avec `HELO`.|
|`smtpd_tls_auth_only`|Refuse l’authentification en clair si TLS n’est pas activé.|
|`tls_random_source`|Source d’aléa utilisée pour générer des clés de session TLS.|

---

### `master.cf.j2`
Le fichier `master.cf` complète la configuration de Postfix.  
Il définit les **services de transport réseau** que Postfix propose, notamment :

- le **SMTP standard** (port 25),
    
- le **SMTPS** (port 465),
    
- et le **Submission** (port 587).
    

Chaque service peut avoir ses propres options, notamment en ce qui concerne le **chiffrement TLS**, l’**authentification SASL**, et le mode de fonctionnement (wrapper vs starttls).

> Ce fichier est crucial pour permettre aux **clients mail externes** (Thunderbird, Roundcube, etc.) d’envoyer des messages via un **canal sécurisé**.


#### Contenu du fichier 
```ini
smtp      inet  n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=may

smtps     inet  n       -       y       -       -       smtpd
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_security_level=encrypt

submission inet n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
```

#### Description des blocs de service :

##### 1. **SMTP (port 25)**

|Champ|Valeur|Explication|
|---|---|---|
|`smtp`|`inet`|Active le service SMTP sur le port standard 25.|
|`smtpd_tls_security_level=may`|-|Le chiffrement TLS est **optionnel**.|


##### 2. **SMTPS (port 465)**

|Champ|Valeur|Explication|
|---|---|---|
|`smtps`|`inet`|Active le SMTPS sur le port 465.|
|`smtpd_tls_wrappermode=yes`|-|TLS est actif **dès le début de la connexion** (pas de STARTTLS, c’est un tunnel TLS direct).|
|`smtpd_sasl_auth_enable=yes`|-|Active l’authentification pour les clients mail.|
|`smtpd_tls_security_level=encrypt`|-|Le chiffrement est **obligatoire**.|


##### 3. **Submission (port 587)**

|Champ|Valeur|Explication|
|---|---|---|
|`submission`|`inet`|Port recommandé pour les **clients authentifiés** (ex : webmail, Outlook).|
|`smtpd_tls_security_level=encrypt`|-|TLS obligatoire (via STARTTLS).|
|`smtpd_sasl_auth_enable=yes`|-|Authentification activée.|

> C’est le **standard moderne** pour l’envoi de mails authentifiés par les utilisateurs.

---

### `postfixadmin-ssl.conf.j2`
Ce fichier est un **fichier de configuration Apache (VirtualHost)** destiné à activer le **site web sécurisé PostfixAdmin** sur le serveur `srv-mail.itway.local`, via HTTPS (port 443).

- **PostfixAdmin** est une interface web de gestion des utilisateurs, domaines et alias pour Postfix.
    
- Cette configuration permet d’**exposer l’interface en HTTPS** avec des certificats signés par la CA interne.
    
- Le fichier est utilisé avec la commande `a2ensite` pour l’activer.
    

#### Contenu du fichier

```apache
<VirtualHost *:443>
    ServerName srv-mail.itway.local

    DocumentRoot /usr/share/postfixadmin
    Alias /postfixadmin /usr/share/postfixadmin/public

    SSLEngine on
    SSLCertificateFile    /etc/postfix/ssl/srv-mail.crt
    SSLCertificateKeyFile /etc/postfix/ssl/srv-mail.key
    SSLCACertificateFile  /etc/postfix/ssl/ca.crt

    <Directory /usr/share/postfixadmin/public>
        Options FollowSymLinks
        AllowOverride All
        Require ip 172.16.10.0/29
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/postfixadmin_error.log
    CustomLog ${APACHE_LOG_DIR}/postfixadmin_access.log combined
</VirtualHost>
```

#### Description des directives

|Directive|Description|
|---|---|
|`<VirtualHost *:443>`|Déclare un site écoutant sur le port 443 (HTTPS).|
|`ServerName srv-mail.itway.local`|Nom DNS utilisé pour accéder au site.|
|`DocumentRoot`|Répertoire principal du site PostfixAdmin.|
|`Alias /postfixadmin`|Permet d'accéder à l'interface via `https://srv-mail.itway.local/postfixadmin`.|
|`SSLEngine on`|Active le SSL/TLS pour ce site.|
|`SSLCertificateFile`|Chemin vers le **certificat public** du serveur.|
|`SSLCertificateKeyFile`|Clé privée associée au certificat (doit rester confidentielle).|
|`SSLCACertificateFile`|Certificat de l’autorité de certification utilisée (CA interne).|
|`<Directory>`|Contrôle les droits d’accès et le comportement du répertoire contenant l’interface web.|
|`AllowOverride All`|Autorise l’usage de `.htaccess` pour modifier dynamiquement certaines règles Apache.|
|`Require ip 172.16.10.0/29`|Permet l’accès seulement au VLAN **IT NETWORK**.|
|`ErrorLog`, `CustomLog`|Chemins vers les fichiers de logs spécifiques pour PostfixAdmin.|

**A savoir :**
Ce fichier s’active avec `a2ensite postfixadmin-ssl.conf`, puis `service apache2 restart`.

---

## 8. Sécurité & conformité
- Chiffrement TLS activé pour Postfix, Dovecot et l’interface web PostfixAdmin (certificats signés par une CA interne).
- Accès à PostfixAdmin restreint au réseau interne (option via `Require ip` dans Apache).
- Authentification sécurisée (via LDAP ou SQL) avec mots de passe chiffrés en `SHA512-CRYPT`.
- Rejet des connexions non sécurisées (auth impossible sans TLS, `disable_plaintext_auth = yes`).
- Permissions strictes sur les fichiers sensibles (clés privées, configs).
- Relais SMTP contrôlé : seuls les utilisateurs internes ou authentifiés peuvent envoyer des mails.

---

## 9. Problèmes rencontrés & solutions

### **Problème 1** : Résolution DNS de `srv-mail.itway.local`

* **Symptôme** : `Name or service not known` lors d’une tentative de connexion TLS (`openssl s_client`).
* **Cause** : Le serveur **DMZ-SMTP ne parvenait pas à résoudre** le nom du serveur interne.
* **Solution** : Ajout d’une entrée manuelle dans `/etc/hosts` sur le serveur DMZ :

  ```
  172.16.50.4 srv-mail.itway.local
  ```

---

### **Problème 2** : Port 465 (SMTPS) inaccessible

* **Symptôme** : `Connection refused` en testant avec `openssl s_client -connect srv-mail.itway.local:465`.
* **Cause** : Le service **Postfix n’écoutait pas sur le port SMTPS (465)**.
* **Solution** :

       * Vérification de la configuration dans `master.cf`.
       * Ajout du bloc `smtps` avec `smtpd_tls_wrappermode=yes`.
       * Redémarrage de Postfix.

---

### **Problème 3** : Postfix ne démarre pas dans le conteneur

* **Erreurs** :

  * `System has not been booted with systemd`
  * `postsuper: Permission denied on /var/spool/postfix/...`

* **Causes** :

       * Le serveur SRV-MAIL tourne dans un **conteneur sans `systemd`**.
       * Les **permissions de `/var/spool/postfix/` avaient été modifiées** (probablement par des tâches Ansible).
       * Tous les fichiers appartenaient à `ubuntu:ubuntu` au lieu de `postfix:postfix`.

* **Solution** :

  ```bash
  sudo chown -R postfix:postfix /var/spool/postfix
  postfix start
  ```

---

## Conclusion

Le serveur **SRV-MAIL** remplit son rôle de serveur de **distribution interne des courriers électroniques**.
En s’appuyant sur un stack open-source éprouvé (Postfix + Dovecot + MariaDB), il offre :

* Une solution **robuste** et sécurisée,
* Un **accès simple et centralisé** à la messagerie des utilisateurs,
* Une **interface d’administration conviviale** (PostfixAdmin),
* Une **intégration propre avec la passerelle SMTP DMZ**.

**Manquant** : 
Mettre pop3s et sans supprimé les données du serveur.


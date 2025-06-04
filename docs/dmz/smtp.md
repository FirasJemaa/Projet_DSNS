# **DMZ-SMTP** (`mail.itway.fr`)

[Playbook Ansible complet](https://github.com/FirasJemaa/Projet_DSNS_playbook)

## 1. Objectifs

Le serveur **DMZ-SMTP** agit comme **passerelle SMTP sécurisée** entre Internet et le réseau interne de l’entreprise. Il est conçu pour :

* **Recevoir les mails externes** et les relayer vers le serveur interne (SRV-MAIL),
* **Relayer les mails sortants** du serveur interne vers Internet,
* **Filtrer les spams et les menaces** avec SpamAssassin,
* **Fournir un webmail (Roundcube)** accessible depuis l’extérieur via HTTPS.

---

## 2. Présentation du service

Un **serveur SMTP** (Simple Mail Transfer Protocol) est un **composant essentiel de l’envoi d’emails**.

### 2.1 Qu'est-ce qu'un serveur SMTP ?
C’est un **serveur qui envoie, reçoit ou transfère** des emails d’un expéditeur vers un ou plusieurs destinataires via le protocole SMTP.


### 2.2 Que fait concrètement un serveur SMTP ?

| Rôle                   | Description                                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Envoi d’e-mails** | Reçoit les mails des clients (ex : Outlook, Thunderbird, Roundcube) et les transmet au serveur du destinataire. |
| **Relais**          | Transfère les mails entre plusieurs serveurs SMTP (par ex. d’un serveur interne à un relais en DMZ).            |
| **Réception**       | Peut aussi recevoir des mails d'autres serveurs SMTP (Internet → entreprise).                                   |

### 2.3 Comment ca marche dans l'insfrastructure ?
#### Fonctionnement d’un envoi de mail
1. Un utilisateur envoie un mail via Roundcube (webmail) :
    * Roundcube utilise le port 587 et le protocole SMTP avec authentification (SMTP AUTH).
    * Le mail est transmis au serveur SRV-MAIL (Postfix).
2. SRV-MAIL agit comme MTA (Mail Transfer Agent) :
    * Il relaye le mail à DMZ-SMTP (Postfix) via le port 25 (SMTP classique).
    * Le trafic est souvent chiffré avec STARTTLS.
3. DMZ-SMTP transfère le mail vers Internet :
    * Il contacte le serveur SMTP du destinataire et livre le message via le port 25.

#### Réception de mails depuis Internet
1. Un mail provenant de `user@gmail.com` est envoyé à `user@itway.fr` :
    * Le serveur SMTP de Google se connecte au serveur DMZ-SMTP (Postfix) sur le port 25.
    * Le message est reçu et filtré par SpamAssassin.
2. DMZ-SMTP le relaie à SRV-MAIL (Postfix) en interne via SMTP.
3. SRV-MAIL stocke le mail avec Dovecot, accessible via :
    * IMAP (port 143 ou 993) pour les clients lourds (Thunderbird, Outlook),
    * Webmail Roundcube (HTTP/S) pour consultation en ligne.


### 2.4 Pourquoi un relais SMTP en DMZ ?

| Objectif                | Explication                                                                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------- |
| **Sécurité**            | Le serveur DMZ agit comme un tampon entre l’extérieur et le réseau interne.                              |
| **Filtrage**            | Intègre **SpamAssassin** pour bloquer les spams avant qu’ils n’atteignent le cœur du réseau.             |
| **Conformité**          | Permet de contrôler et de journaliser les flux mail entrants/sortants.                                   |
| **Haute disponibilité** | En cas de coupure du serveur interne, le relais peut temporairement mettre en file d’attente les emails. |

### Application utilisées

| Service          | Rôle                                                             |
| ---------------- | ---------------------------------------------------------------- |
| **Postfix**      | Serveur SMTP en mode relais (entrées/sorties)                    |
| **SpamAssassin** | Détection et marquage des spams                                  |
| **Roundcube**    | Webmail pour consulter les courriels (via HTTPS + reverse proxy) |
| **iptables**     | Pare-feu limitant les flux entrants (SMTP, HTTPS uniquement)     |

---

## 3. Rôle dans l'infrastructure ITWay
Dans l’infrastructure **ITWay**, le **serveur SMTP DMZ-SMTP** joue un rôle **central de 

Le serveur `DMZ-SMTP` (**mail.itway.fr**) a pour fonction principale de **transporter les courriers électroniques** entre :

* **l’extérieur (Internet)**
* **l’intérieur (réseau ITWay, via SRV-MAIL)**

### 3.1 Réception des mails entrants (depuis Internet)

* Il reçoit les emails destinés à `@itway.fr` depuis les serveurs des expéditeurs externes (Gmail, Orange, etc.).
* Les mails sont d’abord filtrés (**SpamAssassin**), puis **relayés vers `SRV-MAIL`**, qui les stocke via **Dovecot**.

### 3.2 Relais des mails sortants

* Quand un utilisateur interne envoie un email :

    * Il est d’abord envoyé à `SRV-MAIL` via SMTP (avec authentification).
    * Puis, `SRV-MAIL` transfère le message à `DMZ-SMTP`.
    * Le serveur **DMZ-SMTP délivre** alors le mail vers Internet (MTA externe).

### 3.3 Interface d’accès web via Roundcube

* Les utilisateurs peuvent accéder à leurs mails via **le Webmail Roundcube** installé sur `DMZ-SMTP`.
* Ce service est **accessible depuis Internet et l’interne**, via un reverse proxy sécurisé.

---

## 4. Justification des choix

* **Isolation** du relais SMTP dans une DMZ,
* **Postfix** pour sa robustesse et sa compatibilité avec les filtres,
* **SpamAssassin** intégré via Milter pour le filtrage des spams,
* **Roundcube** comme client web complet et simple,
* **Pare-feu** pour restreindre strictement les accès.

---

## 5. Structure Ansible

```plaintext
├── inventory/group_vars/dmz-smtp/
│   ├── main.yml
│   └── vault.yml
├── roles/dmz-smtp/
│   ├── tasks/
│   │   ├── main.yml
│   │   ├── cert.yml
│   │   ├── postfix.yml
│   │   ├── iptables.yml
│   │   └── roundcube.yml
│   ├── templates/
│   │   ├── main.cf.j2
│   │   ├── master.cf.j2
│   │   ├── webmail-ssl.conf.j2
│   │   ├── roundcube-config.inc.php.j2
│   │   └── rules.v4.j2
│   ├── handlers/main.yml
│   └── files/
│       └── transport
```

---

## 6. Ce que fait le Playbook Ansible

Le rôle `dmz-smtp` configure automatiquement un **serveur de messagerie en DMZ**, jouant le rôle de **relais SMTP sécurisé** avec :

* Certificat SSL via SRV-PKI
* Postfix (en mode relais)
* Antispam (SpamAssassin, SPF, DMARC)
* Pare-feu via iptables
* Webmail Roundcube (HTTPS via Apache)

### 1. Certificat SSL (`cert.yml`)

* Génération clé + CSR sur DMZ-SMTP
* Signature via SRV-PKI (`openssl ca`)
* Récupération & déploiement du certificat + CA

### 2. Services essentiels (`main.yml`)

* Installation de Postfix + antispam
* Démarrage manuel de `spamass-milter` si absent
* Import des autres tâches

### 3. Configuration SMTP (`postfix.yml`)

* Déploiement `main.cf`, `master.cf`
* Configuration de la table de transport
* Correction des permissions Postfix (essentiel pour conteneurs)

### 4. Pare-feu (`iptables.yml`)

* Déploiement `rules.v4` pour IPv4
* Activation automatique au boot

### 5. Roundcube (`roundcube.yml`)

* Installation Apache + PHP + Roundcube
* Déploiement HTTPS (vhost)
* Vérifications d’accès locales & distantes

---

## 7. Fichiers de configuration

### `main.cf.j2`

Ce fichier configure **Postfix** en tant que **serveur relais SMTP sécurisé**. Il détermine les domaines relayés, l'authentification, le TLS, les restrictions, et l'intégration antispam.

#### Contenu

```ini
myhostname = {{ smtp_fqdn }}
mydomain   = {{ smtp_domain }}
myorigin   = $mydomain

inet_interfaces = all
mydestination = localhost
relay_domains = $mydomain
transport_maps = hash:/etc/postfix/transport
```

#### Explication des directives principales

| Directive                    | Description                                                             |
| ---------------------------- | ----------------------------------------------------------------------- |
| `myhostname`                 | Nom d’hôte du serveur                                                   |
| `mydomain`                   | Domaine géré                                                            |
| `inet_interfaces`            | Interface d’écoute (`all` = toutes)                                     |
| `mydestination`              | Postfix ne distribue pas localement, seulement localhost                |
| `relay_domains`              | Domaine pour lequel Postfix agit comme **MX relais**                    |
| `transport_maps`             | Table personnalisée des destinations (ex: envoyer tout à SRV-MAIL)      |
| `mynetworks`                 | Adresses IP autorisées à relayer (interne uniquement)                   |
| `smtpd_relay_restrictions`   | Règles anti-relay : n’autoriser que le LAN ou utilisateurs authentifiés |
| `smtpd_tls_*` / `smtp_tls_*` | Paramètres TLS côté serveur et client (certificats SSL)                 |
| `smtpd_sasl_auth_enable`     | Active l’authentification SASL (SMTP Auth)                              |
| `smtpd_milters`              | Intègre **SpamAssassin** via socket `spamass-milter`                    |
| `disable_vrfy_command`       | Désactive la commande `VRFY` (sécurité)                                 |
| `smtpd_helo_required`        | Requiert une commande `HELO` pour commencer la session SMTP             |

---

### `master.cf.j2` 
Ce fichier définit les **services que Postfix écoute**, comme `SMTP`, `SMTPS`, `submission`, ainsi que les options spécifiques à chaque protocole.

#### Contenu du fichier

```ini
# Postfix master.cf – services écoutés
smtp      inet  n  -  y  -  -  smtpd
  -o smtpd_milters=unix:/var/spool/postfix/var/run/spamass-milter.sock

submission inet n - y - - smtpd
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_milters=unix:/var/spool/postfix/var/run/spamass-milter.sock

smtps     inet n - y - - smtpd
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_milters=unix:/var/spool/postfix/var/run/spamass-milter.sock
```

#### Explication des blocs

| Service      | Port | Description                                                             |
| ------------ | ---- | ----------------------------------------------------------------------- |
| `smtp`       | 25   | Reçoit les mails depuis Internet et autres serveurs SMTP                |
| `submission` | 587  | Port pour les **clients mail authentifiés** (ex. Thunderbird, Outlook)  |
| `smtps`      | 465  | Port SMTPS (SMTP sur TLS implicite) – alternative sécurisée au port 587 |


#### Options utilisées

| Option                             | Description                                                              |
| ---------------------------------- | ------------------------------------------------------------------------ |
| `smtpd_tls_security_level=encrypt` | Force le chiffrement TLS                                                 |
| `smtpd_tls_wrappermode=yes`        | Utilise TLS dès la connexion (mode SMTPS)                                |
| `smtpd_sasl_auth_enable=yes`       | Active l'authentification SASL (nécessaire pour `submission` et `smtps`) |
| `smtpd_milters=...`                | Applique le filtre antispam (SpamAssassin via milter) sur chaque service |

---

### `roundcube-config.inc.php.j2`
Ce fichier configure **Roundcube**, l’interface webmail utilisée dans la DMZ pour accéder aux mails de manière sécurisée. Il définit la connexion à la base de données, au serveur IMAP/SMTP, les paramètres de langue, sécurité TLS, et autres options système.


#### Contenu du fichier

```php
<?php

$config['db_dsnw'] = 'mysql://{{ roundcube_db_user }}:{{ roundcube_db_pass }}@tcp(srv-mail.itway.local:3306)/roundcube';
$config['product_name'] = 'ITWay Webmail';
$config['language'] = 'fr_FR';

$config['log_driver'] = 'file';
$config['log_dir'] = '/var/log/roundcube/';

$config['default_host'] = 'tls://srv-mail.itway.local';
$config['default_port'] = 993;

$config['smtp_host'] = 'tls://srv-mail.itway.local';
$config['smtp_port'] = 587;

$config['smtp_user'] = '%u';
$config['smtp_pass'] = '%p';

$config['imap_conn_options'] = [
  'ssl' => [
    'verify_peer'       => true,
    'verify_peer_name'  => true,
    'cafile'            => '/etc/ssl/certs/itway-ca.crt',
  ],
];

$config['smtp_conn_options'] = $config['imap_conn_options'];

$config['enable_installer'] = false;
$config['log_logins'] = true;
$config['debug_level'] = 1;
$config['charset'] = 'UTF-8';

$config['plugins'] = [];

$config['des_key'] = '{{ roundcube_secret_key }}';

?>
```

#### Explication des principaux paramètres

| Paramètre                       | Description                                                              |
| ------------------------------- | ------------------------------------------------------------------------ |
| `db_dsnw`                       | Connexion MySQL à la base Roundcube sur SRV-MAIL                         |
| `product_name`                  | Nom affiché dans l’interface webmail                                     |
| `language`                      | Langue par défaut                                                        |
| `log_driver` / `log_dir`        | Active la journalisation des connexions dans `/var/log/roundcube/`       |
| `default_host` / `default_port` | Serveur IMAP utilisé (connexion chiffrée via TLS sur le port 993)        |
| `smtp_host` / `smtp_port`       | Serveur SMTP utilisé (port 587 avec authentification STARTTLS)           |
| `smtp_user` / `smtp_pass`       | Permet l’authentification avec les identifiants de l’utilisateur `%u/%p` |
| `imap_conn_options`             | Exige la vérification du certificat TLS côté client                      |
| `smtp_conn_options`             | Reprend les mêmes options que pour IMAP                                  |
| `enable_installer`              | Désactive l’assistant d’installation (sécurité)                          |
| `log_logins`                    | Journalise les connexions                                                |
| `plugins`                       | Liste des plugins Roundcube (ici vide mais modifiable)                   |
| `des_key`                       | Clé de chiffrement utilisée pour stocker certains paramètres sensibles   |

---

### `rules.v4.j2`
Ce fichier configure le **pare-feu IPv4** via `iptables`, pour restreindre les connexions entrantes sur le serveur **DMZ-SMTP**. Il définit précisément quels ports et réseaux sont autorisés à communiquer, et bloque tout le reste.

#### Contenu du fichier

```ini
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# États établis
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH seulement depuis VLAN admin (exemple 172.16.0.0/24)
-A INPUT -p tcp -s 172.16.0.0/24 --dport 22 -j ACCEPT

# SMTP + SMTPS + SUBMISSION
-A INPUT -p tcp --dport 25  -j ACCEPT
-A INPUT -p tcp --dport 465 -j ACCEPT
-A INPUT -p tcp --dport 587 -j ACCEPT

# HTTPS Roundcube (appelé par le reverse-proxy)
-A INPUT -p tcp --dport 443 -j ACCEPT

# ICMP optionnel
-A INPUT -p icmp -j ACCEPT
COMMIT
```


#### Explication des règles

| Règle / Section                 | Description                                                            |
| ------------------------------- | ---------------------------------------------------------------------- |
| `:INPUT DROP`                   | Tout le trafic entrant est bloqué par défaut                           |
| `:OUTPUT ACCEPT`                | Tout le trafic sortant est autorisé                                    |
| `--ctstate ESTABLISHED,RELATED` | Autorise les connexions déjà établies ou connexions associées          |
| `SSH - 172.16.0.0/24`           | Autorise l’accès SSH uniquement depuis le VLAN d’administration        |
| `--dport 25 / 465 / 587`        | Ouvre les ports pour SMTP, SMTPS et Submission                         |
| `--dport 443`                   | Ouvre le port HTTPS (pour accéder à Roundcube depuis le reverse proxy) |
| `icmp`                          | Autorise le ping (pour le débogage)            |

---

### `webmail-ssl.conf.j2` 
Ce fichier définit le **VirtualHost Apache** pour exposer Roundcube en HTTPS sur le serveur **DMZ-SMTP**, via le nom de domaine `webmail.itway.fr`. Il active le **chiffrement SSL**, configure les chemins des certificats, et permet l’accès web au webmail.

#### Contenu du fichier

```apache
<VirtualHost *:443>
    ServerName webmail.itway.fr
    DocumentRoot /var/lib/roundcube

    SSLEngine on
    SSLCertificateFile {{ smtp_ssl_dir }}/{{ smtp_crt_filename }}
    SSLCertificateKeyFile {{ smtp_ssl_dir }}/{{ smtp_key_filename }}
    SSLCertificateChainFile {{ smtp_ssl_dir }}/{{ smtp_ca_filename }}

    <Directory /var/lib/roundcube>
        Options +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/webmail_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/webmail_ssl_access.log combined
</VirtualHost>
```

#### Description des directives

| Directive                 | Rôle                                               |
| ------------------------- | -------------------------------------------------- |
| `VirtualHost *:443`       | Écoute sur le port HTTPS (443)                     |
| `ServerName`              | Nom DNS associé au site (ici : `webmail.itway.fr`) |
| `DocumentRoot`            | Répertoire où se trouve l’application Roundcube    |
| `SSLEngine on`            | Active le SSL                                      |
| `SSLCertificateFile`      | Fichier du certificat TLS signé                    |
| `SSLCertificateKeyFile`   | Clé privée associée au certificat                  |
| `SSLCertificateChainFile` | Chaîne de certification (inclut la CA)             |
| `<Directory ...>`         | Règles d’accès HTTP sur le répertoire Roundcube    |
| `ErrorLog` / `CustomLog`  | Journaux d’erreurs et d’accès Apache               |

---

Parfait ! Voici la dernière partie :

---

### `transport`
Le fichier `transport` est utilisé par **Postfix** pour définir **comment router les e-mails** vers des serveurs spécifiques en fonction du domaine. Ici, ce fichier permet au serveur **DMZ-SMTP** de rediriger tous les e-mails du domaine `itway.fr` vers le serveur interne `srv-mail.itway.local`.

#### Contenu du fichier

```ini
# transport_maps – tout le domaine va vers le serveur mail interne
itway.fr   smtp:[srv-mail.itway.local]:25
```

#### Explication des paramètres

| Ligne                            | Signification                                                        |
| -------------------------------- | -------------------------------------------------------------------- |
| `itway.fr`                       | Domaine concerné par cette règle                                     |
| `smtp:[srv-mail.itway.local]:25` | Utilise le protocole SMTP pour envoyer vers le port 25 de `srv-mail` |
| `[...]` autour de l'hôte         | Empêche la résolution MX, envoie directement à l'hôte spécifié       |

---

#### Utilisation avec Postfix

Ce fichier est transformé en base de données `.db` via la commande Ansible :

```bash
postmap /etc/postfix/transport
```

Et activé dans `main.cf` via :

```ini
transport_maps = hash:/etc/postfix/transport
```

Cela garantit que tout mail destiné à `itway.fr` passe **obligatoirement** par `srv-mail.itway.local`.

---

## 8. Sécurité & conformité

* Chiffrement TLS actif sur tous les services,
* Fichiers sensibles avec permissions restrictives (`postfix:postfix`),
* Filtrage par adresse IP, anti-relay (`mynetworks`),
* Webmail en HTTPS uniquement,
* Services séparés en DMZ pour protéger le réseau interne.

---

## 9. Problèmes rencontrés & solutions
| Problème constaté                                   | Solution appliquée                                                      |
| --------------------------------------------------- | ----------------------------------------------------------------------- |
| Certificat non reconnu dans le navigateur        | Ajout manuel des entrées `subjectAltName` dans la configuration OpenSSL |
| Accès web Roundcube refusé                       | Activation du VirtualHost HTTPS via `a2ensite` et redémarrage d’Apache  |
| Impossible d'envoyer un mail (port 465 fermé)    | Vérification de la configuration `master.cf` + redémarrage Postfix      |
| Transport non pris en compte par Postfix         | Génération manuelle de la table de hash avec `postmap`                  |
| SpamAssassin inactif au démarrage                | Ajout d’un lancement manuel dans Ansible en cas d’absence de processus  |
| Permissions incorrectes sur `/var/spool/postfix` | Correction des droits avec `chown` et `chmod` adaptés                   |


---

## Conclusion
La mise en place du serveur **DMZ-SMTP** permet de sécuriser et maîtriser les flux de messagerie entre l’extérieur et l’intérieur du réseau ITWay. En jouant le rôle de relais **SMTP**, il filtre les e-mails via SpamAssassin, chiffre les communications avec des certificats TLS signés par une autorité interne, et assure un relais fiable vers le serveur de messagerie principal.

L’installation de **Roundcube**, accessible depuis Internet via HTTPS, offre une interface conviviale aux utilisateurs tout en conservant un bon niveau de sécurité grâce au reverse proxy et au chiffrement TLS.

Les choix techniques et les automatisations via Ansible garantissent :

- une configuration reproductible,
- une sécurité cohérente avec la politique du système d'information,
- et une maintenance simplifiée.

Ce serveur constitue ainsi un maillon essentiel dans l’architecture de messagerie sécurisée de l’entreprise.

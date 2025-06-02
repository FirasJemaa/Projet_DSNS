# Serveur DMZ-WEB (`dmz-web.itway.fr`)

---

## 1. Objectifs

Mettre en place un **serveur Web sécurisé** dans la **DMZ**, accessible :

* Depuis Internet (site vitrine publique)
* Depuis tous les réseaux internes de confiance

Ce serveur devra :

* Publier le site principal **WordPress** de l’entreprise : `https://itway.fr`
* Servir le trafic en **HTTPS** (certificat signé par la **PKI interne – SRV‑PKI**)
* Reposer sur **Nginx** (choisi pour ses performances & sa simplicité) sur **Debian**
* Être entièrement **automatisé** via **Ansible** (rôle `dmz-web`)

---

## 2. Serveur Web

### 2.1  C’est quoi un serveur Web ?
Un serveur Web (Apache, Nginx…) écoute sur un port HTTP(S) et **renvoie des pages HTML** aux navigateurs.

### 2.2  Pourquoi un serveur Web dans la DMZ ?

| Besoin                       | Rôle du serveur DMZ-WEB                                  |
| ---------------------------- | -------------------------------------------------------- |
| Héberger le site vitrine     | Stocke le code WordPress et ses médias                   |
| Sécurité réseau              | Placé dans la DMZ : isolation vis‑à‑vis du LAN           |
| Sécurité applicative         | Mises à jour WordPress / plugins via Ansible             |
| Chiffrement                  | Terminaison TLS locale (certificat signé par SRV‑PKI)    |
| Haute disponibilité possible | Peut être répliqué / load‑balancé par DMZ‑RPROXY (Nginx) |

---

## 3. Justification des choix

| Composant       | Choix effectué                    | Justification                                                     |
| --------------- | --------------------------------- | ----------------------------------------------------------------- |
| Serveur HTTP    | **Nginx**                         | Léger, performant, configuration claire (reverse proxy + static)  |
| CMS             | **WordPress 6.8.1**               | Facile à utiliser pour un site vitrine, plugins & thèmes nombreux |
| Base de données | **MariaDB**                       | Remplaçant libre / performant de MySQL                            |
| PHP             | **PHP‑FPM 8.3**                   | Compatible WP ; isolé du serveur Web                              |
| Certificats     | **PKI interne (SRV‑PKI)**         | Conformité, contrôle complet, pas de dépendance externe           |
| Automatisation  | **Ansible** + rôles réutilisables | Déploiement idempotent, versionnage Git                           |

---

## 4. Structure du projet Ansible

```text
├── inventory/
|   ├── hosts.ini
│   └── group_vars/
│       └── dmz-web/
│           ├── main.yml 
│           └── vault.yml 
├── roles/
│   └── dmz-web/
│       ├── files/
│       │   ├── functions.php
│       │   ├── footer.php
│       │   ├── header.php
│       │   ├── index.php
│       │   ├── main.js
│       │   └── style.css
│       ├── handlers/
│       │   └── main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── wordpress.yml
│       │   ├── php-fpm.yml
│       │   └── cert.yml
│       ├── templates/
│       │   ├── default.conf.j2
│       │   ├── nginx.conf.j2
│       │   ├── ssl_params.conf.j2
│       │   └── wp-config.php.j2
│       └── vars/
│           └── main.yml
```

> Les rôles sont **isolés**, facilitant les réutilisations et les mises à jour.

---

## 5. Variables principales

Extrait de `roles/dmz-web/vars/main.yml` :

```yaml
# Ports & chemins
dmz_web_port_http: 80
dmz_web_port_https: 443
dmz_web_ssl_dir: "/etc/nginx/ssl"

# FQDN & SAN
dmz_web_fqdn: "dmz-web.itway.fr"
dmz_web_alt_names:
  - itway.fr
  - dmz-web.itway.fr

# Versions logicielles
wp_version: "6.8.1"
php_fpm_version: "8.3"
```

Variables sensibles (mots de passe DB, etc.) sont stockées dans **`group_vars/dmz-web/vault.yml`** (chiffré avec **Ansible Vault**).

---

## 6. Rôle et organisation des playbooks

Ce rôle Ansible `dmz-web` est structuré autour de plusieurs fichiers de tâches qui définissent clairement les différentes étapes du déploiement.

### 6.1 Fichier principal `main.yml`

Ce fichier constitue le point d'entrée du rôle `dmz-web`. Il orchestre l'exécution des sous-tâches suivantes :

1. Installation de Nginx via le module apt
2. Importation du rôle de gestion de certificat TLS (`cert.yml`)
3. Importation du rôle de déploiement WordPress (`wordpress.yml`)
4. Déclenchement conditionnel des handlers si nécessaire

### 6.2 Gestion des certificats `cert.yml`

Ce fichier permet de gérer le cycle de vie complet du certificat TLS signé par la **PKI interne** :

1. Génération de la clé privée locale
2. Création de la CSR (Certificate Signing Request)
3. Transfert vers le serveur SRV-PKI via `delegate_to`
4. Récupération et déploiement du certificat signé (CRT + chaîne)
5. Placement dans le bon répertoire {{ dmz_web_ssl_dir }} sur la cible

### 6.3 Déploiement de WordPress `wordpress.yml`

Ce fichier automatise l’installation et le durcissement de **WordPress** :

1. Installation de MariaDB, PHP-FPM et dépendances WP
2. Téléchargement de WordPress et du CLI associé (wp-cli)
3. Création de la base de données et des utilisateurs MariaDB
4. Déploiement des fichiers du site vitrine (thème, config, index…)
5. Génération du wp-config.php avec clés aléatoires
6. Installation automatique du site (titre, URL, admin, etc.)
7. Sécurisation des accès (DISALLOW_FILE_EDIT, xmlrpc, wp-config)
8. Fixation des permissions correctes (chown www-data)

---

## 7. Gestion du certificat TLS (`tasks/cert.yml`)

### 7.1 Détail du processus automatisé

Ce rôle met en œuvre un processus complet et sécurisé d'obtention de certificat via notre PKI interne. Voici les étapes détaillées :

1. **Génération de la clé privée & CSR sur DMZ-WEB**
    - La cible DMZ-WEB génère localement une clé privée RSA (2048 bits) avec `openssl genrsa`
    - Une CSR (Certificate Signing Request) est générée à partir de cette clé avec `openssl req -new`
    - Ces fichiers sont créés localement sur la cible pour éviter toute fuite de la clé privée
2. **Récupération de la CSR par le contrôleur Ansible**
    - Le fichier `.csr` est récupéré grâce au module fetch, ce qui copie le fichier depuis **DMZ-WEB** vers le contrôleur (machine Ansible)
3. **Signature par la PKI interne via `delegate_to`**
    - Le playbook exécute une commande de signature X.509 sur le serveur `SRV-PKI` grâce à `delegate_to`
    - Typiquement : `openssl ca -in csr.pem -out cert.crt -config /etc/pki/openssl.cn`
    - La PKI utilise une CA locale, et signe avec sa propre clé, en respectant les contraintes du domaine **itway.fr**
4. **Récupération du certificat signé + certificat de la CA**
    - Une fois le certificat signé par SRV-PKI, le contrôleur Ansible récupère les fichiers `cert.crt` et `ca.crt`
    - Ces deux fichiers sont nécessaires pour constituer une chaîne de confiance complète (`fullchain`)
5. **Déploiement sur DMZ-WEB**
    - Les fichiers sont ensuite copiés sur **DMZ-WEB** avec `copy/template`
    - Le certificat client (`cert.crt`) est concaténé avec `ca.crt` pour produire un `fullchain.crt` utilisé par Nginx
    - Les droits sont fixés pour être lisibles uniquement par `www-data`

### 7.2 Schéma du flux PKI

```
sequenceDiagram
    participant A as DMZ-WEB (cible)
    participant B as Contrôleur Ansible
    participant C as SRV-PKI

    A->>A: Génère clé privée + CSR
    A-->>B: fetch CSR
    B->>C: Délègue la signature de la CSR
    C-->>B: Retourne le certificat signé (CRT) + ca.crt
    B-->>A: Copie fullchain.crt sur DMZ-WEB
```

> Aucun flux direct DMZ ↔ PKI : tout est orchestré par Ansible via delegate_to, avec un contrôle total du processus et une sécurité maximale.

---

## 8. Configuration Nginx

La configuration Nginx est entièrement générée via **templates Jinja2** dans Ansible. Chaque fichier `.j2` est automatiquement rendu avec les variables personnalisées et copié dans `/etc/nginx/`.

---

### 8.1 `nginx.conf.j2`

Le fichier `nginx.conf.j2` correspond à la configuration **globale** de Nginx. Il est responsable des comportements principaux du serveur :

#### Bloc 1 : Paramètres globaux

```nginx
user  www-data;
worker_processes auto;
pid /run/nginx.pid;
```

| Ligne                    | Description                                                  |
| ------------------------ | ------------------------------------------------------------ |
| `user www-data;`         | Définit l’utilisateur système utilisé par Nginx              |
| `worker_processes auto;` | Nombre de processus géré automatiquement selon les cœurs CPU |
| `pid /run/nginx.pid;`    | PID stocké dans ce fichier pour gérer le service             |

#### Bloc 2 : Gestion des connexions

```nginx
events {
    worker_connections 768;
}
```

| Ligne                     | Description                                     |
| ------------------------- | ----------------------------------------------- |
| `worker_connections 768;` | Nombre max de connexions simultanées par worker |

#### Bloc 3 : Bloc HTTP principal

```nginx
http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  sendfile      on;
  keepalive_timeout 65;

  access_log /var/log/nginx/access.log;
  error_log  /var/log/nginx/error.log;

  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

| Ligne                                    | Description                                            |
| ---------------------------------------- | ------------------------------------------------------ |
| `include /etc/nginx/mime.types;`         | Types MIME standards (html, jpg, pdf…)                 |
| `default_type application/octet-stream;` | Type par défaut en absence de détection                |
| `sendfile on;`                           | Optimisation pour fichiers statiques                   |
| `keepalive_timeout 65;`                  | Délai d’inactivité avant fermeture de connexion        |
| `access_log`, `error_log`                | Fichiers journaux pour les accès / erreurs             |
| `include conf.d/*.conf`                  | Inclut configs supplémentaires (SSL…)                  |
| `include sites-enabled/*`                | Charge les virtualhosts (dont celui du site WordPress) |

---

### 8.2 `default.conf.j2`

Ce fichier est le **virtualhost principal** de Nginx qui sert le site WordPress. Il force la redirection HTTPS et configure tous les chemins nécessaires à l’exécution du CMS.

#### Bloc 1 : Redirection HTTP vers HTTPS

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name itway.fr dmz-web.itway.fr;
    return 301 https://$host$request_uri;
}
```

| Ligne                            | Description                                  |
| -------------------------------- | -------------------------------------------- |
| `listen 80;` / `listen [::]:80;` | Écoute en IPv4/IPv6 sur le port HTTP         |
| `server_name`                    | FQDN du serveur (site principal + alias DMZ) |
| `return 301 https://...`         | Redirige tout le trafic vers HTTPS           |

#### Bloc 2 : Virtualhost HTTPS

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name itway.fr dmz-web.itway.fr;

    ssl_certificate     {{ dmz_web_ssl_dir }}/fullchain.crt;
    ssl_certificate_key {{ dmz_web_ssl_dir }}/{{ dmz_web_key_filename }};
    include /etc/nginx/snippets/ssl_params.conf;

    root {{ wp_root }}/wordpress;
    index index.php;

    client_max_body_size 32M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php{{ php_fpm_version }}-fpm.sock;
    }

    location ~* wp-config\\.php { deny all; }
    location = /xmlrpc.php     { deny all; }

    location ~* \\.(jpg|jpeg|gif|png|css|js|ico|webp)$ {
        expires 365d;
        access_log off;
    }
}
```

| Élément                     | Description                                                  |
| --------------------------- | ------------------------------------------------------------ |
| `ssl_certificate / ssl_key` | Chemins vers les certificats TLS                             |
| `include ssl_params.conf`   | Paramètres de chiffrement à part                             |
| `root` + `index`            | Répertoire WordPress et fichier par défaut                   |
| `location /`                | Route par défaut qui relance vers `index.php`                |
| `location ~ \\.php$`        | Exécution sécurisée des fichiers PHP via FPM                 |
| `deny all`                  | Sécurité : empêche l’accès à `wp-config.php` et `xmlrpc.php` |
| `expires 365d`              | Cache long pour les ressources statiques                     |

---

### 8.3 `ssl_params.conf.j2`

Ce fichier contient les **paramètres TLS** appliqués à tous les serveurs `ssl` de Nginx.

```nginx
ssl_protocols {{ dmz_web_ssl_protocols }};
ssl_ciphers   {{ dmz_web_ssl_ciphers }};
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
```

| Directive                      | Description                                        |
| ------------------------------ | -------------------------------------------------- |
| `ssl_protocols`                | Protocoles acceptés (ex. TLSv1.2, TLSv1.3)         |
| `ssl_ciphers`                  | Liste de suites cryptographiques autorisées        |
| `ssl_prefer_server_ciphers on` | Privilégie les choix du serveur sur ceux du client |
| `ssl_session_cache`            | Cache partagé pour les connexions SSL              |
| `ssl_session_timeout`          | Durée de validité d’une session TLS                |


### 8.4 `wp-config.php.j2`

Ce fichier est le **template principal de configuration WordPress**, généré dynamiquement avec Ansible. Il permet de connecter WordPress à la base de données, d’activer certaines sécurités, et d’injecter automatiquement les clés de salage.

#### Contenu du template

```php
<?php
// Définition des informations de connexion à la base de données
define('DB_NAME', '{{ wp_db_name }}');           // Nom de la base de données
define('DB_USER', '{{ wp_db_user }}');           // Nom d'utilisateur de la base
define('DB_PASSWORD', '{{ wp_db_pass }}');       // Mot de passe utilisateur
define('DB_HOST', '127.0.0.1');                  // Adresse du serveur MariaDB
define('DB_CHARSET', 'utf8mb4');                 // Encodage pour WordPress
define('DB_COLLATE', '');                        // Collation (vide = valeur par défaut)

// Sécurité : forcer l'administration en HTTPS
define('FORCE_SSL_ADMIN', true);                 // Accès au dashboard uniquement en HTTPS
define('DISALLOW_FILE_EDIT', true);              // Désactive l'éditeur de thème/plugin depuis l'admin WP

// Préfixe des tables WordPress dans la base
$table_prefix = 'wp_';                           // Personnalisable pour plus de sécurité

/* Clés de salage WordPress – utilisées pour sécuriser les sessions et cookies */
// Ces clés sont générées dynamiquement via Ansible
{% for key in ['AUTH_KEY','SECURE_AUTH_KEY','LOGGED_IN_KEY','NONCE_KEY','AUTH_SALT','SECURE_AUTH_SALT','LOGGED_IN_SALT','NONCE_SALT'] %}
define('{{ key }}', '{{ lookup("password", "/dev/null length=64 chars=ascii_letters,digits") }}'); // Clé {{ key }}
{% endfor %}

// Désactive le mode debug 
define('WP_DEBUG', false);

// Définition du chemin absolu vers WordPress
if ( ! defined('ABSPATH') ) {
    define('ABSPATH', __DIR__ . '/');            // Définit le dossier racine de WP
}

// Chargement des fichiers principaux de WordPress
require_once ABSPATH . 'wp-settings.php';

```

#### Explication des blocs

| Élément                          | Description                                                               |
| -------------------------------- | ------------------------------------------------------------------------- |
| `DB_NAME`, `DB_USER`, etc.       | Variables injectées depuis Ansible pour connecter WordPress à MariaDB     |
| `FORCE_SSL_ADMIN`                | Force l'accès à l'interface admin via HTTPS uniquement                    |
| `DISALLOW_FILE_EDIT`             | Désactive l’éditeur de thème/plugin via le tableau de bord (sécurité)     |
| `$table_prefix`                  | Préfixe des tables WP, personnalisable pour éviter les injections         |
| Clés `AUTH_KEY` ... `NONCE_SALT` | Clés de sécurité générées dynamiquement (aléatoires à chaque déploiement) |
| `WP_DEBUG`                       | Debug désactivé en production                                             |
| `ABSPATH` + `wp-settings.php`    | Démarre le cœur WordPress                                                 |

---

## 9. Installation & durcissement WordPress (`wordpress.yml`)

| Étape | Action                                                                      |
| ----- | --------------------------------------------------------------------------- |
| 1     | Installation paquets **PHP‑FPM**, **MariaDB**, bibliothèques indispensables |
| 2     | Téléchargement & installation **WP‑CLI**                                    |
| 3     | Démarrage **MariaDB** (service) + création comptes / base                   |
| 4     | Téléchargement et extraction **WordPress** `{{ wp_version }}`               |
| 5     | Génération **wp‑config.php** (salts uniques)                                |
| 6     | Installation WP en ligne de commande (titre, admin, URL)                    |
| 7     | Déploiement du **thème ITWay** (files/style.css, etc.)                      |
| 8     | Fixation des **permissions** (www‑data)                                     |

> **Plus de sécurité** : `DISALLOW_FILE_EDIT` dans `wp‑config.php` désactive l’éditeur de code WordPress côté admin.

---

## 10. Sécurité & conformité

| Mesure appliquée                | Description                                                     |
| ------------------------------- | --------------------------------------------------------------- |
| Chiffrement forcé (301 → HTTPS) | Aucune page en clair                                            |
| TLS 1.2/1.3 uniquement          | `ssl_protocols`                                                 |
| Ciphers forts                   | `ssl_ciphers`                                                   |
| Headers de sécurité             | `X‑Frame‑Options`, `X‑Content‑Type‑Options`, HSTS (optionnel)   |
| Isolation réseau                | Serveur placé en DMZ, non joignable depuis Internet en SSH      |
| WP hardening                    | `xmlrpc.php` bloqué, `wp‑config.php` inaccessible, mises à jour |
| Least privilege                 | Service `www-data`, pas de root                                 |
| Healthcheck automatisé          | Échec = alerte dans le playbook Ansible                         |

---

## 11. Tests fonctionnels

### 11.1 Redirection HTTP → HTTPS

```sh
curl -I http://itway.fr
# HTTP/1.1 301 Moved Permanently
```

### 11.2 Page d’accueil WordPress

```sh
curl -k https://itway.fr | grep "<title>"
# <title>ITWay – Just another WordPress site</title>
```

---

## 12. Problèmes rencontrés

| Problème                                     | Résolution                                              |
| -------------------------------------------- | ------------------------------------------------------- |
| `service nginx start` indisponible (systemd) | Utiliser `nginx -s reload` via handler                  |
| MariaDB ne démarre pas dans le conteneur     | Lancer `service mariadb start` + socket `/run/mysqld`   |
| Erreur "test failed" `nginx -t`              | Vérifier présence du **CRT** avant le test (`stat`)     |
| WP‑CLI timeouts (DNS)                        | Forcer `--path` + désactiver validation certs si besoin |

---

## 13. Liens utiles

* [Playbook complet Ansible et rôles](#structure-du-projet-ansible)
* [Référence officielle Nginx TLS](https://nginx.org/en/docs/http/configuring_https_servers.html)
* [Documentation WordPress Hardening](https://wordpress.org/support/article/hardening-wordpress/)

---

## Conclusion

La mise en place de **DMZ‑WEB** assure la **publication sécurisée** du site vitrine : certificat interne, CMS WordPress à jour, rôle Ansible idempotent. Placé derrière **DMZ‑RPROXY**, le serveur profite d’une **double couche** : filtrage du reverse proxy et durcissement applicatif local.

Grâce à cette architecture, l’entreprise garantit :

* **Disponibilité** du site (DMZ dédiée, supervision Healthcheck)
* **Confidentialité & intégrité** (TLS 1.3, ciphers forts)
* **Facilité d’exploitation** (Ansible + PKI interne)

Le rôle `dmz-web` peut être ré‑exécuté à tout moment pour appliquer des mises à jour ou déployer de nouvelles instances.

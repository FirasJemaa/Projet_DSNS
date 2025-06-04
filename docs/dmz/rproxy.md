# Serveur RPROXY (Reverse Proxy)

---

## 1. Objectifs

Mettre en place un **serveur Reverse Proxy sécurisé (DMZ-RPROXY)** dans la DMZ, accessible depuis :

* **Internet**
* **Tous les réseaux internes de confiance**

Ce serveur a pour mission de :

* **Exposer les services web** internes de manière sécurisée via HTTPS :
    * `itway.fr`
    * `webmail.itway.fr`
* **Rediriger** les requêtes vers les serveurs internes correspondants :
    * `DMZ-WEB` (ex : 10.10.10.2)
    * `Serveur mail/webmail` (ex : 10.10.10.3)
* **Gérer les certificats SSL/TLS** via une **PKI interne** (`SRV-PKI`)
* Appliquer des **règles de sécurité avancées** (headers, protocoles, ciphers)

---

## 2. Reverse Proxy
### 2.1 C'est quoi un Reverse Proxy
Un **reverse proxy** est un serveur **intermédiaire** entre un client (navigateur) et un ou plusieurs serveurs d'application.

### Fonctionnement :

1. Le client se connecte au reverse proxy.
2. Le proxy **reçoit la requête**, la **redirige vers le bon serveur** (en DMZ ou LAN).
3. Il **reçoit la réponse** et la renvoie **au client comme si elle venait de lui**.

### Avantages :

| Bénéfice                    | Détail                                                     |
| --------------------------- | ---------------------------------------------------------- |
| **Sécurité**                | Les serveurs réels sont masqués                            |
| **HTTPS centralisé**        | Un seul point de gestion du chiffrement                    |
| **Load balancing possible** | Peut répartir les requêtes entre plusieurs backends        |
| **Contrôle du trafic**      | Possibilité de filtrer, journaliser, réécrire les requêtes |

---

## 3. Justification des choix

| Composant               | Choix effectué                   | Justification                                 |
| ----------------------- | -------------------------------- | --------------------------------------------- |
| Reverse Proxy           | **Nginx**                        | Léger, rapide, facile à configurer            |
| SSL/TLS                 | Certificat signé par **SRV-PKI** | Sécurité, intégration dans ton infrastructure |
| Automatisation          | **Ansible** + rôles              | Déploiement reproductible et versionné        |
| Séparation des domaines | `itway.fr`, `webmail.itway.fr`   | Permet une gestion claire des services        |

---

## 4. Structure du projet Ansible


```
roles/
└── dmz-rproxy/
    ├── tasks/
    │   ├── main.yml
    │   └── cert.yml
    ├── templates/
    │   ├── nginx.conf.j2
    │   ├── default.conf.j2
    │   ├── webmail.conf.j2
    │   ├── ssl_params.conf.j2
    │   └── security_headers.conf.j2
    ├── vars/
    │   └── main.yml
    └── handlers/
        └── main.yml
```

---

## 5. Gestion du certificat TLS

Le certificat TLS est :

* **Généré localement** sur DMZ-RPROXY (`.key`)
* Une **CSR** est créée (`.csr`)
* Elle est envoyée et **signée par `SRV-PKI`**
* Le certificat signé (`.crt`) + la CA (`ca.crt`) sont récupérés
* Un fichier `fullchain.crt` est assemblé

> Ceci est **entièrement automatisé** via Ansible (`tasks/cert.yml`)

---

## 6. Configuration Nginx

### 6.1 `nginx.conf.j2`

Le fichier nginx.conf.j2 est le fichier de configuration principal de Nginx.
Il est déployé automatiquement via Ansible grâce au template .j2.

Ce fichier configure les comportements globaux de Nginx :

- Utilisateur système
- Nombre de processus
- Gestion des connexions
- Fichiers de logs
- Chargement des fichiers de configuration supplémentaires

#### Explication Bloc 1 : Paramètres globaux 

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

```

| Ligne                    | Signification                                                                    |
| ------------------------ | -------------------------------------------------------------------------------- |
| `user www-data;`         | Définit l’utilisateur système pour Nginx (par défaut sur Debian : `www-data`)    |
| `worker_processes auto;` | Nombre de processus = auto-adapté selon les cœurs CPU disponibles                |
| `pid /run/nginx.pid;`    | Fichier contenant le PID principal de Nginx (utilisé pour la gestion du service) |

#### Explication Bloc 2 : Gestion des connexions

```nginx
events {
    worker_connections 1024;
}
```

| Ligne                      | Signification                                                   |
| -------------------------- | --------------------------------------------------------------- |
| `worker_connections 1024;` | Chaque worker peut accepter jusqu’à 1024 connexions simultanées |

#### Explication Bloc 3 : Bloc HTTP principal

```nginx
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

| Ligne                                    | Signification                                                             |
| ---------------------------------------- | ------------------------------------------------------------------------- |
| `include /etc/nginx/mime.types;`         | Charge la liste des types MIME (text/html, image/png...)                  |
| `default_type application/octet-stream;` | Type de fichier par défaut si non reconnu (binaire générique)             |
| `sendfile on;`                           | Utilise une méthode optimisée pour envoyer les fichiers statiques         |
| `keepalive_timeout 65;`                  | Garde les connexions HTTP ouvertes pendant 65 secondes (timeout)          |
| `access_log /var/log/nginx/access.log;`  | Journalise toutes les requêtes reçues                                     |
| `error_log /var/log/nginx/error.log;`    | Journalise les erreurs rencontrées                                        |
| `include /etc/nginx/conf.d/*.conf;`      | Charge les fichiers de configuration globaux (certificats, headers, etc.) |
| `include /etc/nginx/sites-enabled/*;`    | Charge les vhosts activés (itway.fr, webmail.itway.fr, etc.)              |


### 6.2 `default.conf.j2`
Le fichier `default.conf.j2` est un template Ansible qui génère la configuration Nginx pour le site principal itway.fr.
Il définit :

- Une redirection automatique de HTTP vers HTTPS,
- Une configuration sécurisée pour le reverse proxy en HTTPS,
- La connexion sécurisée vers le backend web_front (le serveur web interne),
- L’intégration de la sécurité TLS (certificats, headers).

Vhost principal pour :

* `itway.fr` (redirection HTTP ➜ HTTPS)
* Reverse proxy vers `10.10.10.2`

#### Explication Bloc 1 : Redirection HTTP vers HTTPS
```nginx
# Redirection HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name itway.fr www.itway.fr;
    return 301 https://$host$request_uri;
}
```

| Ligne                                   | Signification                                                                    |
| --------------------------------------- | -------------------------------------------------------------------------------- |
| `listen 80;`                            | Nginx écoute sur le port 80 (HTTP)                                               |
| `listen [::]:80;`                       | Idem pour IPv6                                                                   |
| `server_name itway.fr www.itway.fr;`    | Ce bloc concerne ces domaines                                                    |
| `return 301 https://$host$request_uri;` | Redirige toutes les requêtes HTTP vers HTTPS (code 301 = redirection permanente) |

> But : forcer tous les utilisateurs à passer par une connexion sécurisée.


#### Explication Bloc 2 : Définition du backend
```nginx
# Bloc principal HTTPS
upstream web_front {
    server 10.10.10.2:443;
}
```

| Ligne                    | Signification                                                    |
| ------------------------ | ---------------------------------------------------------------- |
| `upstream web_front`     | Définit un groupe de serveurs backend, ici appelé `web_front`    |
| `server 10.10.10.2:443;` | Ce backend est le **serveur DMZ-WEB** interne, écoutant en HTTPS |

#### Explication Bloc 3 : Serveur principal HTTPS
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name itway.fr www.itway.fr;

    ssl_certificate           {{ rproxy_ssl_dir }}/{{ rproxy_fullchain_filename }};
    ssl_certificate_key       {{ rproxy_ssl_dir }}/{{ rproxy_key_filename }};
    ssl_trusted_certificate   {{ rproxy_ssl_dir }}/{{ rproxy_ca_filename }};

    include {{ rproxy_nginx_snippet_dir }}/ssl_params.conf;
    include {{ rproxy_nginx_snippet_dir }}/security_headers.conf;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        proxy_pass https://web_front;
        proxy_ssl_verify off;
        proxy_read_timeout 30s;
    }
}
```

| Ligne                        | Signification                                        |
| ---------------------------- | ---------------------------------------------------- |
| `listen 443 ssl http2;`      | Le serveur écoute en HTTPS sur IPv4                  |
| `listen [::]:443 ssl http2;` | Idem pour IPv6                                       |
| `server_name ...`            | Ce serveur s’applique à `itway.fr` et `www.itway.fr` |
| `ssl_certificate`         | Le certificat TLS (signé par ta PKI)                      |
| `ssl_certificate_key`     | La clé privée associée                                    |
| `ssl_trusted_certificate` | La chaîne de confiance (CA) pour vérification côté client |
| `Host`              | Nom de domaine original         |
| `X-Real-IP`         | Adresse IP du client            |
| `X-Forwarded-For`   | Historique des proxys traversés |
| `X-Forwarded-Proto` | Protocole utilisé (http/https)  |
| `proxy_pass`             | Redirige les requêtes vers `https://web_front` (défini plus haut)                   |
| `proxy_ssl_verify off`   | Désactive la vérification TLS côté backend (car c’est un certif interne non public) |
| `proxy_read_timeout 30s` | Timeout de lecture des réponses (au-delà de 30s → erreur)                           |



### 6.3 `webmail.conf.j2`

Même principe que `default.itway.fr` pour `webmail.itway.fr` ➜ vers `10.10.10.3` (SMTP).

---

## 7. Sécurité HTTPS

### 7.1 `ssl_params.conf.j2`
Le fichier ssl_params.conf.j2 est un fichier de configuration inclus dans chaque server HTTPS de Nginx.
Il est utilisé pour configurer la sécurité TLS/SSL du reverse proxy.
On y définit les protocoles autorisés, les chiffrements (ciphers), et des protections comme OCSP Stapling ou HSTS.

#### À quoi sert ce fichier ?
Il permet de :

- Sécuriser la connexion HTTPS
- Empêcher l’utilisation de protocoles obsolètes
- Définir quelles suites de chiffrement (ciphers) le serveur doit accepter
- Améliorer la vitesse et la fiabilité avec OCSP stapling
- Protéger les utilisateurs avec HSTS (forcer HTTPS sur longue durée)

#### Contenu du fichier 
- Activer uniquement les protocoles TLS spécifiés dans les variables Ansible :
```nginx
ssl_protocols {{ rproxy_ssl_protocols }};
```

- Permettre au client (navigateur) de choisir le chiffrement préférée parmi celles proposées :
```nginx
ssl_prefer_server_ciphers off;
```

- Définir les ciphers autorisés (suites de chiffrement TLS) :
```nginx
ssl_ciphers {{ rproxy_ssl_ciphers }};
```
> Objectif : accepter uniquement les ciphers modernes, sûrs, et résistants aux attaques (ex : ECDHE-RSA-AES256-GCM-SHA384)

- OSCP Stapling :
```nginx
ssl_stapling on;
ssl_stapling_verify on;
```
    - `ssl_stapling on;` : Permet de fournir une preuve que le certificat est valide (revocation check rapide)

    - `ssl_stapling_verify on;` : Vérifie la signature de cette preuve

- HSTS – Force HTTPS côté client :
```nginx
{% if rproxy_hsts_max_age|int > 0 %}
add_header Strict-Transport-Security "max-age={{ rproxy_hsts_max_age }}; includeSubDomains";
{% endif %}
```

> Active HSTS (HTTP Strict Transport Security) uniquement si rproxy_hsts_max_age > 0 et envoie un header au client pour forcer l’usage de HTTPS pour une durée définie



### 7.2 `security_headers.conf.j2`
Le fichier security_headers.conf.j2 est un fichier inclus dans la configuration Nginx HTTPS, servant à appliquer des en-têtes HTTP de sécurité.

L'objectif est de renforcer la sécurité côté client en envoyant des instructions supplémentaires au navigateur via des HTTP headers.
Ces headers ne modifient pas le contenu des pages mais imposent des règles de sécurité.

#### À quoi sert ce fichier ?

Ce fichier configure les en-têtes HTTP de sécurité que le serveur Nginx va ajouter à chaque réponse.

Il est inclus dans chaque bloc server HTTPS pour renforcer la protection des clients (navigateurs) contre :

- Les attaques XSS (Cross-Site Scripting)
- Le clickjacking
- L’injection de contenu malveillant
- La fuite d’informations sensibles

> Il améliore la posture de sécurité du site sans toucher au code applicatif.

#### Contenu du fichier

- Empêcher le site d’être chargé dans une iframe depuis un autre domaine :
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```
> Protège contre le clickjacking (attaques qui piègent l’utilisateur avec des boutons invisibles).

- Empêcher le navigateur d’interpréter le type de contenu différemment de celui annoncé par le serveur :
```nginx
add_header X-Content-Type-Options "nosniff" always;
```
> Protège contre les attaques MIME-type sniffing.

- Contrôler ce que le navigateur envoie comme en-tête Referer lors d’un lien vers un autre site :
```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```
> Protège la vie privée de l’utilisateur et limite les fuites d’URL internes.

- Politique CSP : elle définit ce que le navigateur a le droit de charger et d’afficher
```nginx
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'self'; upgrade-insecure-requests" always;

```
> Empêche les scripts externes non autorisés (contre XSS) et force l’usage du HTTPS.

- Contrôler les API sensibles du navigateur (géolocalisation, micro, etc.)
```nginx
add_header Permissions-Policy "geolocation=(), microphone=()" always;

```
> Limite les fonctions dangereuses et protège la vie privée.

**A Savoir** :

`always` : ajoute l'en-tête même si le code HTTP n’est pas 200 OK (utile pour erreurs 404, redirections, etc.)

---

## 8. Tests fonctionnels

### Redirection HTTP ➜ HTTPS :

```sh
curl -I http://itway.fr
# HTTP/1.1 301 Moved Permanently
```

### Test HTTPS :

```sh
curl -k https://itway.fr
# Affiche la page web servie par DMZ-RPROXY
```

### Test HTTPS `webmail.itway.fr` :

```sh
curl -vk https://webmail.itway.fr
# Connexion TLS OK
```

---

### Test en externe (Internet)

* Via `nslookup`, j'obtiens :

```sh
Name: itway.fr
Address: 10.10.10.5
```

> En testant depuis l’extérieur :

```sh
curl -vk https://192.168.122.24
```

> Résultat obtenu : connexion TLS valide, contenu HTML renvoyé

---

---

## 9. Liens utiles

* [Playbook complet Ansible et rôles](#structure-du-projet-ansible)
* [Référence officielle Nginx TLS](https://nginx.org/en/docs/http/configuring_https_servers.html)

---

## 10. Résumé du rôle de DMZ-RPROXY

| Étape | Action                                                  |
| ----- | ------------------------------------------------------- |
| 1     | Le client (navigateur) appelle `https://itway.fr`       |
| 2     | DMZ-RPROXY répond avec son certificat SSL               |
| 3     | Il **redirige la requête en interne** vers `10.10.10.2` |
| 4     | Le contenu est renvoyé au client via DMZ-RPROXY         |

---

## Conclusion
La mise en place du serveur DMZ-RPROXY joue un rôle crucial dans l'architecture réseau de l’entreprise. Il sert de point d’entrée sécurisé entre les utilisateurs (internes ou externes) et les services web de l’infrastructure, notamment itway.fr et webmail.itway.fr.

Grâce à l’utilisation de Nginx comme reverse proxy, cette solution permet de :

- Centraliser la gestion du protocole HTTPS et des certificats TLS,
- Protéger les serveurs backend en les rendant inaccessibles directement depuis Internet,
- Appliquer des règles de sécurité strictes à toutes les connexions HTTP/HTTPS (headers, HSTS, chiffrement moderne),
- Et assurer une compatibilité, une performance et une maintenabilité élevées via un déploiement automatisé avec Ansible.

Le reverse proxy constitue donc un composant fondamental de la DMZ, garantissant à la fois la sécurité, la disponibilité et la transparence des services web exposés.




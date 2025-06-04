# **SRV-PKI** (`srv-pki.itway.local`)

[Playbook Ansible complet](https://github.com/FirasJemaa/Projet_DSNS_playbook)


## 1. Objectifs

Le serveur **SRV-PKI** a pour rôle d’être **l’autorité de certification interne** de l’entreprise ITWay. Il est chargé :

* Générer des certificats numériques (SSL/TLS),
* Signer les demandes de certificats pour les services internes (VPN, Webmail, etc.),
* Publier des Listes de Révocation de Certificats (CRL).

---

## 2. Présentation du service

### 2.1 Qu'est-ce qu'une PKI ?
Une PKI (Infrastructure à Clés Publiques) est un système cryptographique utilisé pour :

- Authentifier l’identité des utilisateurs et des machines,
- Chiffrer les communications pour les sécuriser,
- Signer numériquement les échanges (garantie d’intégrité et non-répudiation).

Elle repose sur l’utilisation de paires de clés :

- Une clé privée : conservée secrètement,
- Une clé publique : diffusée librement.

L’objectif principal : fournir des certificats numériques valides aux entités internes pour sécuriser les services (HTTPS, VPN, authentification, etc.).

Le serveur gère aussi :

- Les **CSR** (Certificate Signing Requests), qui sont des demandes de certificats créées par les services,
- Les **SAN** (Subject Alternative Name), qui permettent d’avoir plusieurs noms DNS dans un même certificat.

### 2.2 Les composants d'une PKI

| Élément                               | Rôle principal                                               |
| ------------------------------------- | ------------------------------------------------------------ |
| **CA (Certificate Authority)**        | Entité de confiance qui signe les certificats                |
| **CRL (Certificate Revocation List)** | Liste des certificats révoqués                               |
| **Certificats**                       | Fichiers d'identité numériques pour utilisateurs/machines    |
| **Clé privée**                        | Secrète, utilisée pour signer ou décrypter                   |
| **Clé publique**                      | Accessible, utilisée pour vérifier une signature ou chiffrer |
| **CSR (Certificate Signing Request)** | Demande de certificat signée par une entité                  |

### 2.3 Cycle de vie d’un certificat
1. Génération d’une clé privée/publique
2. Création d’une CSR
3. Signature de la CSR par la CA → création du certificat
4. Utilisation du certificat par un serveur (ex : HTTPS, VPN, RDP…)
5. Révocation (si compromission, départ, etc.) → ajout à la CRL
6. Renouvellement avant expiration

### 2.4 Structure du PKI dans mon projet
```pgsql
/etc/pki/
├── certs/        ← Certificats délivrés
├── private/      ← Clés privées (permissions strictes)
├── crl/          ← Listes de révocation
├── newcerts/     ← Certificats récemment signés
├── index.txt     ← Historique des certificats
├── serial        ← Numéro de série courant
├── openssl.cnf   ← Fichier de configuration central
```

### 2.5 Différents dossiers et commandes
➤ `/etc/pki/certs/`

Il contient tous les certificats délivrés par la **CA**, y compris le certificat de la **CA** (`ca.crt`). Et les certificats signés pour les services (`webmail.itway.fr.crt` et autre).

**Création manuelle :**
```sh
mkdir -p /etc/pki/certs
chmod 755 /etc/pki/certs
```
Mode `0755` : Lecture/exec publique. Parfait pour des dossiers standards.

➤ `certs/ca.crt`
C'est un certificat d’autorité (CA) auto-signé, ce qui est souvent la première étape pour mettre en place une infrastructure PKI interne.

Commande pour générer un certificat auto-signé de la CA : 
```sh
openssl req -x509 -new -key /etc/pki/private/ca.key -sha256 -days 3650 -config /etc/ssl/openssl.cnf -extensions v3_ca -out /etc/pki/certs/ca.crt
```

➤ `/etc/pki/private/`

Il contient les **clés privées**, qui sont strictement confidentielles. Par exemple (`ca.key` ou `webmail.itway.fr.key` )

**Création manuelle :**
```sh
mkdir -p /etc/pki/private
chmod 700 /etc/pki/private
```
Mode `0700` : Privé (admin seulement). Dossier private/, accès restreint.

➤ `/etc/pki/private/ca.key`
C'est le fichier qui contient la clé de déchiffrement.

C'est la clé privé, la commande suivante permet d'en créé une : 
```sh
openssl genrsa -out /etc/pki/private/ca.key 4096
```

➤ `/etc/pki/crl/`

C'est le stocke des **listes de Révocation de Certificats** (CRL), le fichier principal est le **`crl.pem`**.

La commande de création de la CRL :
```sh
openssl ca -gencrl -config /etc/ssl/openssl.cnf -out /etc/pki/crl/crl.pem
```

| Option    | Explication                                                |
| --------- | ---------------------------------------------------------- |
| `ca`      | Utilise le mode "CA" d'OpenSSL (autorité de certification) |
| `-gencrl` | Génére une liste de révocation (CRL)                       |
| `-config` | Utilise le fichier `openssl.cnf`                           |
| `-out`    | Où stocker la CRL générée                                  |

➤ `crl/crl.pem`
C'est un fichier de liste de révocation de certificats, c'est une liste CRL (Certificate Revocation List), c’est-à-dire une liste de certificats révoqués par une autorité de certification (CA). 
La commande suivante permet d'en créé une :  
```sh
openssl ca -gencrl -config /etc/ssl/openssl.cnf -out /etc/pki/crl/crl.pem
```

| Élément                        | Rôle                                                                               |
| ------------------------------ | ---------------------------------------------------------------------------------- |
| `openssl`                      | Outil en ligne de commande pour la cryptographie (librairie OpenSSL).              |
| `ca`                           | Sous-commande pour gérer une **autorité de certification**.                        |
| `-gencrl`                      | Demande de **générer une CRL** (liste de révocation).                              |
| `-config /etc/ssl/openssl.cnf` | Fichier de configuration contenant les infos de la CA (chemins, politiques, etc.). |
| `-out /etc/pki/crl/crl.pem`    | Fichier de sortie de la CRL générée, au format **PEM**.                            |


➤ `/etc/pki/newcerts/`

Il contient les **certificats générés récemment**, avec des noms de fichiers numérotés automatiquement (par le fichier `serial`).

**Création manuelle :**
```sh
mkdir -p /etc/pki/newcerts
chmod 755 /etc/pki/newcerts
```

➤ `/etc/pki/serial`

Il contient le numéro de série courant à attribuer au prochein certificat signé. Il commence généralement à `1`.

**Création manuelle :**
```sh
echo 01 > /etc/pki/serial
chmod 644 /etc/pki/serial
```

➤ `/etc/pki/index.txt`

C’est la base de données des certificats. Elle garde une trace de tous les certificats signés par la CA, avec leurs état (Validés, Expirés, Révoqués).

**Création manuelle :**
```sh
touch /etc/pki/index.txt
chmod 644 /etc/pki/index.txt
```
Mode `0644` : Lecture pour tous. Pour les certificats `.crt`.

Au départ le contenu il est vide et ensuite il est rempli automatiquement par `openssl ca`.

➤ `/etc/pki/crlnumber`

Le fichier crlnumber est utilisé pour versionner chaque CRL qu'on émets. C’est comme un numéro de série spécifique aux listes de révocation. Il est obligatoire pour être conforme à la norme `X.509` et à chaque nouvelle CRL (le numéro est incrémenté automatiquement).

**Exemple de génération une CRL** : 
```sh
openssl ca -gencrl -config /etc/ssl/openssl.cnf -out /etc/pki/crl/crl.pem
```

| Option    | Explication                                                |
| --------- | ---------------------------------------------------------- |
| `ca`      | Utilise le mode "CA" d'OpenSSL (autorité de certification) |
| `-gencrl` | Génére une liste de révocation (CRL)                       |
| `-config` | Utilise le fichier `openssl.cnf`                           |
| `-out`    | Où stocker la CRL générée                                  |


**Création manuelle :**
```sh
echo 01 > /etc/pki/crlnumber
chmod 644 /etc/pki/crlnumber
```

➤ `reqs/*.csr`
C'est un fichier généré lorsqu’on souhaite obtenir un certificat numérique signé par une autorité de certification (CA).

Demande de certificat pour un service (ex: Webmail) :
```sh 
openssl req -new -key /etc/pki/private/webmail.itway.fr.key -out /etc/pki/reqs/webmail.itway.fr.csr -config /etc/pki/openssl_webmail.cnf
```

➤ `certs/*.fullchain.crt`
Certificat du service + certificat de la CA (chaîne complète), ce contient la chaîne complète de certificats, ce qui est essentiel pour assurer la confiance du client (navigateur, OS, etc.) lors d'une connexion HTTPS. Ce fichier contient plusieurs certificats empilés.
```sh 
cat webmail.itway.fr.crt ca.crt > webmail.itway.fr.fullchain.crt
```

**Pourquoi on concatène un certificat** :
Lorsqu’un client (navigateur, système, application) se connecte à un site HTTPS :

- Il reçoit le certificat du serveur (ex. : itway.fr)
- Il doit vérifier l’authenticité de ce certificat via une chaîne de confiance :
    - Le certificat du serveur a été signé par une CA intermédiaire
    - Elle-même a été signée par une CA racine (connue du système client)

Si un maillon est manquant, le client affiche une erreur de certificat non fiable.

➤ `/etc/ssl/openssl.cnf`

C'est le fichier de configuration **central pour OpenSSL**. Il définit les chemins, durées de validité et l'usages autorisés. Il contient également les sections **v3_ca, v3_server, v3_client**. Il sert à la création et signature du certificat.

**Exemple d'utilisation** : 
```sh
openssl req -new -x509 -days 3650 -config /etc/ssl/openssl.cnf -key private/ca.key -out certs/ca.crt
```

| Option        | Explication                                              |
| ------------- | -------------------------------------------------------- |
| `req`         | Lance une requête de certificat                          |
| `-x509`       | Crée un certificat **auto-signé** (pas une CSR)          |
| `-new`        | Génère un nouveau certificat                             |
| `-key`        | Spécifie la clé privée                                   |
| `-sha256`     | Algorithme de signature                                  |
| `-days`       | Durée de validité                                        |
| `-config`     | Fichier `openssl.cnf` utilisé                            |


### 2.6 Rappels des extensions 

| Extension        | Signification                     | Contenu                                                            |
| ---------------- | --------------------------------- | ------------------------------------------------------------------ |
| `.key`           | Clé privée                        | Doit rester **secrète**, utilisée pour signer                      |
| `.csr`           | Certificate Signing Request       | Demande envoyée à la CA pour obtenir un certificat                 |
| `.crt`           | Certificat signé (clé publique)   | Contient la clé publique + signature de la CA                      |
| `.fullchain.crt` | Certificat + CA                   | Utilisé pour les services web, inclut toute la chaîne de confiance |
| `.pem`           | Format texte de tous les fichiers | Peut contenir clé, certificat ou CRL                               |

### 2.7 Résumé des tâches du **PKI**
1. Générer une clé privée :
```bash
openssl genrsa -out webmail.itway.fr.key 4096
```
2. Créer une demande de certificat (CSR) :
```bash
openssl req -new -key webmail.itway.fr.key -out webmail.itway.fr.csr -config openssl_webmail.cnf
```
3. Signer avec la CA :
```bash
openssl x509 -req -in webmail.itway.fr.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out webmail.itway.fr.crt -days 3650 -sha256
```
4. Créer un fullchain : 
```bash
cat webmail.itway.fr.crt ca.crt > webmail.itway.fr.fullchain.crt
```

---

## 3. Ce que fait le serveur SRV-PKI dans ITWay
- Génère la clé privée de la CA
- Crée et signe un certificat auto-signé
- Gère les demandes de certificats (CSR) internes
- Produit les certificats valides pour :
    - Web internes (HTTPS)
    - Accès VPN
    - Interfaces de supervision
    - Authentification LDAPS, etc.
- Publie la liste CRL, accessible en lecture pour les autres services
- Ne communique pas vers l’extérieur, sauf en consultation interne

---

## 4. Justification des choix

* **Outils principaux** :
    * `OpenSSL` pour la génération et la gestion des certificats,
    * `Ansible` pour le **déploiement automatisé** (via rôles et playbook),
* **Structure** conforme aux standards (arborescence OpenSSL, séparations des rôles et CRL).

**Pourquoi ces choix ?**

* OpenSSL : open-source, complet, bien documenté.
* Ansible : permet l’automatisation, la reproductibilité, la cohérence des installations.

---

## 5. Structure du projet Ansible
```plaintext
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── setup-pki.yml
└── roles/
    └── srv-pki/
        ├── files/
        │   └── openssl_webmail.cnf     
        ├── handlers/
        │   └── main.yml                     
        ├── tasks/
        │   ├── main.yml                    
        │   └── webmail.yml                  
        ├── templates/
        │   └── openssl.cnf.j2              
        └── vars/
            └── main.yml

```

---

## 6. Ce que le Playbook execute dans `SRV-PKI`

### 1. Configuration des variables PKI
Pour que le playbook s'execute correctement il faut mettre en place le fichiers de variables centralisées dans (`vars/main.yml`), qui contient les paramètre qui peuvent évoluer selon le besoin.

### 2. Déploiement automatisé avec Ansible 
1.  Installation des paquets nécessaires :
    - `openssl`
    - `ca-certificate`
2. Création de l’arborescence PKI, répertoires : 
    - certs, 
    - crl, 
    - newcerts, 
    - private
> Permissions adaptées selon la sensibilité du dossier
3. Initialisation des fichiers de base :
    - `index.txt`, 
    - `serial`, 
    - `crlnumber`
> Générés uniquement s’ils n’existent pas, avec valeurs par défaut (01)
4. Déploiement du fichier de configuration OpenSSL : 
    - Utilisation d’un template Jinja2 (`openssl.cnf.j2`)
    - Inclusion de profils `v3_ca`, `v3_server`, `v3_client`, avec gestion de SAN. 
5. Génération de la CA :
    - Clé privée : `ca.key`
    - Certificat auto-signé : `ca.crt`
    - Extensions spécifiques au rôle de CA (`v3_ca`)
    - Sécurisation automatique des fichiers via un handler
6. Génération du fichier aléatoire OpenSSL :
    - `private/.rand` : nécessaire pour la génération cryptographique
7. Génération du CRL initial :
    - `crl.pem` vide, prêt à être utilisé pour la révocation
8. Vérification finale : 
    - Affichage lisible du certificat CA et du CRL avec openssl x509 et openssl crl
9. Génération et signature d’un certificat serveur (pour Webmail) :
    1. Génère une clé privée dédiée au service webmail (webmail.itway.fr).
    2. Copie un fichier de configuration OpenSSL spécifique au webmail sur le serveur.
    3. Crée une CSR (Certificate Signing Request) pour le domaine webmail.itway.fr à l'aide de la clé générée et du fichier de configuration.
    4. Signe la CSR avec le certificat de l'autorité de certification (CA) précédemment créée.
    5. Génère un certificat fullchain, c’est-à-dire le certificat du webmail concaténé avec celui de la CA, prêt à être utilisé sur un serveur web sécurisé.

---

## 7. Fichiers de configuration
Le fichier `openssl` est essentiel, car il dicte comment sont créés et signés les certificats.

#### `[ ca ]`

Il définit que la configuration par défaut de l’autorité de certification (CA) est celle nommée `CA_default`. C’est le point d’entrée global utilisé par les commandes comme `openssl ca`.
```ini
[ ca ]
default_ca = CA_default
```

#### `[ CA_default ]`

C’est la section principale de configuration de la CA. 
```ini
[ CA_default ]
dir             = {{ pki_ca_dir }}
certs           = $dir/certs
crl_dir         = $dir/crl
new_certs_dir   = $dir/newcerts
database        = $dir/{{ pki_index_file }}
serial          = $dir/{{ pki_serial_file }}
RANDFILE        = $dir/private/.rand
private_key     = $dir/private/{{ pki_ca_key_name }}
certificate     = $dir/certs/{{ pki_ca_cert_name }}
default_md      = {{ pki_default_md }}
policy          = policy_match
email_in_dn     = no
default_days    = {{ pki_ca_days }}
default_crl_days = {{ pki_crl_days }}
crl_days        = {{ pki_crl_days }}
unique_subject  = no
x509_extensions = v3_ca
```

Voici le rôle de chaque directive :

| Directive         | Description                                                               |
| ----------------- | ------------------------------------------------------------------------- |
| `dir`             | Répertoire de base de la PKI (`/etc/pki`)                                 |
| `certs`           | Où stocker les certificats signés                                         |
| `crl_dir`         | Où stocker les CRL (listes de révocation)                                 |
| `new_certs_dir`   | Où mettre les nouveaux certificats signés (historique)                    |
| `database`        | Fichier index de la CA (trace de tous les certificats)                    |
| `serial`          | Fichier avec le prochain numéro de série                                  |
| `RANDFILE`        | Fichier d'entropie utilisé par OpenSSL pour générer des clés              |
| `private_key`     | Fichier de la clé privée de la CA                                         |
| `certificate`     | Fichier du certificat public de la CA                                     |
| `default_md`      | Algorithme de hachage par défaut (`sha256`)                               |
| `policy`          | Règles de politique pour le contenu des certificats                       |
| `email_in_dn`     | Ne pas inclure l’email dans le DN                                         |
| `default_days`    | Validité par défaut des certificats signés                                |
| `crl_days`        | Durée de validité de la CRL                                               |
| `x509_extensions` | Extensions à utiliser pour les certificats signés par la CA (ici `v3_ca`) |


#### `[ policy_match ]`

Définit quelles infos doivent être présentes et valides dans les CSR (Certificate Signing Requests) :

```ini
[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

| Champ      | Rôle                                 |
| ---------- | ------------------------------------ |
| `match`    | Doit correspondre exactement à la CA |
| `supplied` | L'utilisateur doit le fournir        |
| `optional` | Peut être vide                       |

#### `[ v3_ca ]`

Extensions utilisées pour le certificat racine (CA) :
```ini
[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

- `basicConstraints = CA:true` → ce certificat est une autorité de certification.
- `keyUsage` → Ce que le certificat a le droit de faire : signer d'autres certificats (`keyCertSign`), signer des CRL (`cRLSign`).
- `critical` → L’extension est obligatoire pour que le certificat soit accepté.

#### `[ v3_server ]`

Extensions utilisées pour les certificats de serveurs (ex : HTTPS) :
```ini
[ v3_server ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:itway.fr, DNS:dmz-web.itway.fr, DNS:srv-mail.itway.local, DNS:srv-mail
```

- `CA:false` → Ce certificat ne peut pas signer d'autres certificats.
- `keyEncipherment` → Permet le chiffrement des échanges.
- `extendedKeyUsage = serverAuth` → Utilisable pour un serveur.
- `subjectAltName` (SAN) → Liste les domaines que le certificat couvre.

#### `[ v3_client ]`

Pour des certificats utilisés par des clients (utilisateurs, VPN, etc.) :
```ini
[ v3_client ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature
extendedKeyUsage = clientAuth
```

- `clientAuth` → Permet l’authentification côté client (par ex. dans un VPN).
- Pas de SAN ici, car ce n’est pas lié à un nom de domaine.

#### `[ alt_names ]`

Liste des noms DNS alternatifs autorisés dans les certificats signés :
```ini
[ alt_names ]
DNS.1 = itway.fr
DNS.2 = dmz-web.itway.fr
DNS.3 = srv-mail.itway.local
DNS.4 = srv-mail
```

Ces noms apparaîtront dans le champ `Subject Alternative Name` du certificat, indispensable pour HTTPS moderne (le `CommonName` seul ne suffit plus).

---

## 8. Sécurité & conformité
- Les clés privées sont protégées avec des permissions strictes (`0600`) dans un répertoire sécurisé (`0700`).
- Les fichiers sensibles (`serial`, `crlnumber`, `index.txt`) sont créés uniquement s’ils n’existent pas, évitant toute perte de données.
- Les certificats `X.509` sont générés avec :
    - Algorithme de signature : `SHA-256`
    - Longueur de clé : 4096 bits
    - Extensions conformes (`v3_ca`, `v3_server`) avec basicConstraints et keyUsage définis.
- Le processus est entièrement automatisé et reproductible grâce à Ansible.
- Accès restreint : seuls les utilisateurs autorisés (ex. : ansible, root) peuvent manipuler les fichiers clés.

---

## 9. Problèmes rencontrés et solutions

### 1. Problème d'encodage YAML

* **Erreur** : `Unexpected Exception: 'utf-8' codec can't encode character '\udcc3'`
* **Cause** : Caractères accentués mal encodés (souvent copiés depuis Word)
* **Solution** : Conversion en UTF-8 avec `iconv` :

```sh
find . -name "*.yml" -exec iconv -f ISO-8859-1 -t UTF-8 "{}" -o "{}" \;
```

### 2. Erreur `Missing sudo password`

* **Cause** : L'utilisateur `ansible` n'avait pas les droits `sudo` sans mot de passe
* **Solution** :

Ajouter l'utilisateur dans `/etc/sudoers` :

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

### 3. Problème permissions `/usr/bin/sudo`

* **Erreur** : `sudo must be owned by uid 0`
* **Solution** :

  ```bash
  chown root:root /usr/bin/sudo
  chmod 4755 /usr/bin/sudo
  ```

### 4. Erreur de certificat HSTS avec DMZ-SMTP
Lors de l'accès via navigateur à un service derrière `dmz-smtp`, une erreur HSTS persistante apparaissait, empêchant toute connexion sécurisée.

**Cause** : Le champ `subjectAltName` n'était pas reconnu dans le certificat généré, bien que la section `[ alt_names ]` existait dans le fichier `openssl.cnf.j2` :
```ini
[ alt_names ]
DNS.1 = itway.fr
DNS.2 = dmz-web.itway.fr
DNS.3 = srv-mail.itway.local
DNS.4 = srv-mail
```
**Résolution (partielle)** : Après plusieurs tentatives et recherches (24h), une solution de contournement a été de créer un fichier de configuration spécifique au service concerné (`openssl_webmail.cnf`) et de l'utiliser dans les commandes OpenSSL.

**Solution définitive (trouvée plus tard)** : Pour le serveur `srv-mail`, le problème a été trouvé et corrigé en ajoutant explicitement la ligne suivante dans la section `[ v3_server ]` du fichier `openssl.cnf.j2` :
```ini
subjectAltName = DNS:itway.fr, DNS:dmz-web.itway.fr, DNS:srv-mail.itway.local, DNS:srv-mail
```
Cela a permis de générer un certificat fonctionnel avec les bons SANs, reconnu sans erreur HSTS par les navigateurs.

> **Remarque** : Par manque de temps, cette correction n'a pas été appliquée rétroactivement au certificat dmz-smtp, qui utilise toujours le fichier dédié (`openssl_webmail.cnf`).

---

## Conclusion

Le serveur **SRV-PKI** est désormais capable de :

* Générer des certificats pour tous les services internes
* Gérer la révocation (CRL)
* Garantir une sécurité forte grâce à la structure de fichiers, aux permissions strictes et à l’utilisation d’Ansible

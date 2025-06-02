
# **SRV-PKI** (`srv-pki.itway.local`)

---

### 1. Objectif

Le serveur **SRV-PKI** a pour rôle d’être **l’autorité de certification interne** de l’entreprise ITWay. Il est chargé :

* De la génération, distribution et révocation des **certificats SSL/TLS**,
* De la **gestion des clés** publiques/privées,
* De la publication des **Listes de Révocation de Certificats (CRL)**.

---

### 2. Présentation du service

#### 2.1 Qu'est-ce qu'une PKI ?
Une PKI (Infrastructure à Clés Publiques) est un système cryptographique utilisé pour :
- Authentifier l’identité des utilisateurs et des machines,
- Chiffrer les communications pour les sécuriser,
- Signer numériquement les échanges (garantie d’intégrité et non-répudiation).

Elle repose sur l’utilisation de paires de clés :
- Une clé privée : conservée secrètement,
- Une clé publique : diffusée librement.

L’objectif principal : fournir des certificats numériques valides aux entités internes pour sécuriser les services (HTTPS, VPN, authentification, etc.).

#### 2.2 Les composants d'une PKI

| Élément                               | Rôle principal                                               |
| ------------------------------------- | ------------------------------------------------------------ |
| **CA (Certificate Authority)**        | Entité de confiance qui signe les certificats                |
| **CRL (Certificate Revocation List)** | Liste des certificats révoqués                               |
| **Certificats**                       | Fichiers d'identité numériques pour utilisateurs/machines    |
| **Clé privée**                        | Secrète, utilisée pour signer ou décrypter                   |
| **Clé publique**                      | Accessible, utilisée pour vérifier une signature ou chiffrer |
| **CSR (Certificate Signing Request)** | Demande de certificat signée par une entité                  |

#### 2.3 Cycle de vie d’un certificat
1. Génération d’une clé privée/publique
2. Création d’une CSR
3. Signature de la CSR par la CA → création du certificat
4. Utilisation du certificat par un serveur (ex : HTTPS, VPN, RDP…)
5. Révocation (si compromission, départ, etc.) → ajout à la CRL
6. Renouvellement avant expiration

#### 2.4 Structure du PKI dans mon projet
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

---

### 3. Ce que fait le serveur SRV-PKI dans ITWay
- Génère la clé privée de la CA
- Crée et signe un certificat auto-signé
- Gère les demandes de certificats (CSR) internes
- Produit les certificats valides pour :
    - Web internes (HTTPS)
    - Accès VPN
    - Interfaces de supervision
    - Authentification LDAP, etc.
- Publie la liste CRL, accessible en lecture pour les autres services
- Ne communique pas vers l’extérieur, sauf en consultation interne

### 4. Solutions choisies

* **Outils principaux** :
    * `OpenSSL` pour la génération et la gestion des certificats,
    * `Ansible` pour le **déploiement automatisé** (via rôles et playbook),
* **Structure** conforme aux standards (arborescence OpenSSL, séparations des rôles et CRL).

**Pourquoi ces choix ?**

* OpenSSL : open-source, complet, bien documenté.
* Ansible : permet l’automatisation, la reproductibilité, la cohérence des installations.

---

### 5. Méthodologie de mise en place

#### ➤ Étapes globales :

1. Préparation de l’infrastructure Ansible
2. Configuration des variables (organisation, durée, chemins, droits)
3. Déploiement automatisé :
    * Installation des paquets
    * Création des répertoires et fichiers de base (index, serial, crlnumber…)
    * Génération de la CA (clé + certificat)
    * Déploiement du fichier de configuration OpenSSL
    * Génération du CRL initial

#### ➤ Structure Ansible utilisée :

```plaintext
├── ansible.cfg
├── inventory/
│   └── hosts
├── playbooks/
│   └── pki.yml
└── roles/
    └── pki/
        ├── files/
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   └── openssl.cnf.j2
        └── vars/
            └── main.yml
```

---

### 6. Détail de la configuration (par blocs)

#### Variables PKI (`vars/main.yml`)

* Informations d'organisation
* Permissions sécurisées
* Algorithmes de chiffrement (`SHA256`, 4096 bits)
* Validité : 10 ans pour la CA, 30 jours pour le CRL

#### Configuration OpenSSL (`templates/openssl.cnf.j2`)

* Sections définies :
    * `v3_ca` : pour CA
    * `v3_server`, `v3_client` : pour extensions spécifiques
    * `alt_names` : pour SAN DNS

#### Tâches (`tasks/main.yml`)

* Installation des paquets
* Sécurisation des fichiers (`chmod`, `chown`)
* Initialisation des fichiers `serial`, `crlnumber`, `index.txt`
* Génération :
    * clé privée : `openssl genrsa`
    * certificat CA auto-signé : `openssl req -x509`
    * CRL initiale : `openssl ca -gencrl`

---

### 7. Gestion de la sécurité

* **Accès SSH uniquement via clé**
* **Dossiers sécurisés** (`/etc/pki/private` : `0700`)
* **Clé privée CA** : mode `0600`, possédée par l’utilisateur `ansible`
* **Web UI/accès distant** : **aucun exposé**
* **Authentification d’accès PKI** réservée au réseau IT
* **Signature de certificats** exclusivement via Ansible ou commandes locales

---

### 8. Problèmes rencontrés et solutions

#### 1. Problème d'encodage YAML

* **Erreur** : `Unexpected Exception: 'utf-8' codec can't encode character '\udcc3'`
* **Cause** : Caractères accentués mal encodés (souvent copiés depuis Word)
* **Solution** : Conversion en UTF-8 avec `iconv` :

```sh
find . -name "*.yml" -exec iconv -f ISO-8859-1 -t UTF-8 "{}" -o "{}" \;
```

#### 2. Erreur `Missing sudo password`

* **Cause** : L'utilisateur `ansible` n'avait pas les droits `sudo` sans mot de passe
* **Solution** :

  * Ajouter l'utilisateur dans `/etc/sudoers` :

    ```bash
    ansible ALL=(ALL) NOPASSWD: ALL
    ```

#### 3. Problème permissions `/usr/bin/sudo`

* **Erreur** : `sudo must be owned by uid 0`
* **Solution** :

  ```bash
  chown root:root /usr/bin/sudo
  chmod 4755 /usr/bin/sudo
  ```

---

### 9. Supervision et maintenance

* Fichier CRL à régénérer toutes les 30 jours :

  ```bash
  openssl ca -gencrl ...
  ```

* Rotation des certificats planifiée (fonction de la durée)
* Surveillance des fichiers :
    * `/etc/pki/certs/ca.crt`
    * `/etc/pki/crl/crl.pem`
* Journalisation manuelle recommandée pour les signatures et révocations

---

### Conclusion

Le serveur **SRV-PKI** est désormais capable de :

* Générer des certificats pour tous les services internes
* Gérer la révocation (CRL)
* Garantir une sécurité forte grâce à la structure de fichiers, aux permissions strictes et à l’utilisation d’Ansible

**Objectif du cahier des charges respecté à 100 %** :

* Signature des certificats internes
* Accessibilité restreinte depuis le réseau IT
* Publication du CRL pour les autres services


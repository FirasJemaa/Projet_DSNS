# Serveur DNS (`dmz-dns.itway.fr`)

## 1. Objectifs

L'objectif du DNS dans le projet :

* Déployer un **serveur DNS public** dans la **DMZ** nommé `dmz-dns.itway.fr`.
* Ce serveur gère la **résolution DNS** de la zone publique `itway.fr` ainsi que de la zone interne `int.itway.fr`.
* La zone **itway.fr** doit être **accessible depuis Internet**.
* La zone **int.itway.fr** est strictement **privée**, non exposée.
* Aucune **communication entre DNS interne et DMZ** n’est autorisée.
* L’outil choisi est **BIND9**, déployé automatiquement via **Ansible**.

---

## 2. DNS (Domain Name System)
### 2.1  C'est quoi un DNS
C’est le "répertoire téléphonique d’Internet".

Il sert à traduire des noms de domaine lisibles par l’homme (comme google.com ou itway.fr) en adresses IP (comme 142.250.74.238), que les ordinateurs utilisent pour communiquer.

### 2.2 Pourquoi on a besoin du DNS dans le projet
On doit créer un serveur DNS dans la DMZ, accessible depuis Internet et les réseaux internes de confiance.

Il sert à :

| Domaine demandé        | Ce qu’il renvoie            | Où ça sert                    |
| ---------------------- | --------------------------- | ----------------------------- |
| `itway.fr`             | adresse IP du reverse proxy | sur Internet                  |
| `webmail.itway.fr`     | adresse IP du mail ou proxy | sur Internet et internes      |
| `dmz-web.int.itway.fr` | IP du serveur web privé     | **uniquement réseau interne** |
| `dmz-rt.int.itway.fr`  | IP du routeur privé         | **uniquement interne**        |

---

## 3 Ce que fait le DMZ-DNS dans ITWAY
1. Répondre aux requêtes DNS publiques pour itway.fr (serveur dans DMZ)
2. Répondre aux requêtes DNS privées pour int.itway.fr (mais uniquement pour les internes)
3. Ne pas laisser les zones internes visibles depuis Internet
4. Ne pas communiquer avec le DNS interne pour éviter les fuites de données

---

## 4. Justification des choix

| Élément        | Choix effectué                                                     | Raisons                                               |
| -------------- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| Serveur DNS    | BIND9                                                              | Référence open-source, très flexible, bien documentée |
| Automatisation | Ansible + rôles + Jinja2                                           | Réutilisable, versionnable, idempotent                |
| Organisation   | Rôles séparés (`dmz-dns`)                                          | Isolation de la logique métier et clarté              |
| Sécurité       | Récursion restreinte, pas de transfert de zone, DNSSEC             | Réduction des vecteurs d’attaque                      |

---

## 5. Structure du projet Ansible

```
.
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── setup-dmz-dns.yml
└── roles/
    └── dmz-dns/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── templates/
        │   ├── named.conf.local.j2
        │   ├── named.conf.options.j2
        │   ├── db.itway.fr.j2
        │   ├── db.int.itway.fr.j2
        │   └── db.10.10.10.rev.j2
```

---

## 6. Ce que le Playbook execute dans `dmz-dns` 
Vous pouvez visiter les playbook [ici](link).

### Explication du rôle du playbook étape par étape

1. **Installation de Bind9 :** Installer Bind9 et les paquets nécessaires.
2. **Déploiement de la configuration locale :** Déclarer les zones DNS à gérer. [(Plus en détail)](#namedconflocalj2)
3. **Déploiement des options globales :** Configurer les forwarders, la récursion, les IPs autorisées, etc. [(Plus en détail)](#namedconfoptionsj2)
4. **Fichiers de zone :** Configurer le fichier de zone (SOA, CNAME, MX, TXT). [(Plus en détail)](#dbitwayfrj2)
5. **Zone interne (privée) :**  Configurer le fichier de zone interne "Intranet" (SOA, CNAME, MX, TXT). [(Plus en détail)](#-dbintitwayfrj2)
6. **Zone de résolution inversée** [(Plus en détail)](#-db101010revj2)
7. **Vérification de Bind9 :** Vérifier que Bind9 tourne correctement.
8. **Démarrage conditionnel :** Redémarrage de Bind9

---

## 7. Fichiers de configuration

### 7.1 `named.conf.options.j2`
Le fichier `named.conf.options.j2` définit les options globales de configuration du serveur BIND.
Il est utilisé pour régler des paramètres généraux comme :

- Où se trouve le cache,
- Quels serveurs DNS sont utilisés comme relais (forwarders),
- Qui est autorisé à poser des questions récursives (récursion DNS),
- Les comportements réseau (interfaces, IPv6, sécurité, etc.).

#### Contenu du fichiers 

```
options {
    directory "/var/cache/bind";

    // Forward DNS non gérés vers l’extérieur
    forwarders { 8.8.8.8; 8.8.4.4; };

    // Autoriser la récursion UNIQUEMENT à nos réseaux internes :
    allow-recursion {
        10.10.10.0/28;      // DMZ
        172.16.10.0/27;     // IT           
        172.16.50.0/29;     // SRV         
        172.16.85.0/27;     // NET-SRV   
        10.11.11.0/30;      // STORM-NET 
        192.168.20.0/28;    // VLAN 20   
        192.168.30.0/28;    // VLAN 30   
        192.168.40.0/26;    // VLAN 40   
    };

    recursion yes;               // mais uniquement pour les réseaux listés
    allow-transfer { none; };    // bloque les transferts de zones (sécurité)
    dnssec-validation auto;      // active la validation DNSSEC (si utilisé)
    listen-on { any; };          // écoute toutes les interfaces IPv4
    listen-on-v6 { any; };       // écoute toutes les interfaces IPv6
};
```

#### Explication

| Ligne                               | Élément           | Explication                                                                                                                                              |
| ----------------------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `directory "/var/cache/bind";`      | Cache DNS         | Emplacement où BIND stocke son cache de requêtes DNS.                                                                                                    |
| `forwarders { 8.8.8.8; 8.8.4.4; };` | Redirecteurs DNS  | Si le serveur ne connaît pas la réponse, il la demande à ces serveurs DNS (Google DNS).                                                                  |
| `allow-recursion { ... };`          | Contrôle d'accès  | **Limite les requêtes récursives** aux adresses IP internes uniquement. Cela évite que le serveur soit utilisé comme un DNS ouvert (faille de sécurité). |
| `recursion yes;`                    | Récursion activée | Autorise la récursion DNS (nécessaire pour les clients internes), mais **seulement pour les IP listées ci-dessus**.                                      |
| `allow-transfer { none; };`         | Sécurité zone DNS | Interdit les **transferts de zone** (AXFR) aux autres serveurs DNS → protection contre la copie de données.                                              |
| `dnssec-validation auto;`           | Sécurité DNSSEC   | Active automatiquement la validation de DNSSEC (si disponible). Cela permet de vérifier l'authenticité des réponses DNS.                                 |
| `listen-on { any; };`               | Interfaces IPv4   | Le serveur DNS écoute sur **toutes les interfaces réseau IPv4**.                                                                                         |
| `listen-on-v6 { any; };`            | Interfaces IPv6   | Idem pour IPv6.                                                                                                                                          |

### 7.2 `named.conf.local.j2`
Le fichier `named.conf.local.j2` contient la déclaration des zones DNS dont le serveur BIND est maître (master).
C’est ici qu'on lies chaque domaine DNS à son fichier de données (fichier de zone).

**Il joue un rôle fondamental** : sans ce fichier, le serveur DNS ne saurait pas quelles zones il doit gérer, ni où trouver leurs enregistrements.

Ce fichier est un template Ansible (.j2), qui sera transformé à l’exécution en `/etc/bind/named.conf.local`.

#### Contenu 
```jinja
//
// named.conf.local
//

// Vue publique (zone exposée)
zone "itway.fr" {
    type master;
    file "/etc/bind/db.itway.fr";
    allow-transfer { none; };
};

// Interne uniquement (zone privée)
zone "int.itway.fr" {
    type master;
    file "/etc/bind/db.int.itway.fr";
    allow-transfer { none; };
};

zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10.10.10.rev";
    allow-transfer { none; };
};
```

#### Déclarations des zones :

##### Qu'est-ce qu'un fichier de Zone DNS ?
Un fichier de zone décrit tous les enregistrements DNS pour un domaine donné.
Par exemple : **db.itway.fr** est le fichier de zone pour le domaine **itway.fr**.

* **Publique** `itway.fr` :

| Élément                        | Signification                                                                                              |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `zone "itway.fr"`              | Déclare que le serveur gère la zone `itway.fr`                                                             |
| `type master;`                 | Le serveur est **maître** (responsable principal) pour cette zone                                          |
| `file "/etc/bind/db.itway.fr"` | Le fichier de zone contenant les enregistrements se trouve à cette adresse                                 |
| `allow-transfer { none; };`    | Interdit tout **transfert de zone** à d'autres serveurs DNS (protection contre la copie ou les fuites DNS) |

> Cette zone est publique : elle contient des noms accessibles depuis Internet (www.itway.fr, mail.itway.fr, etc.).

* **Privée** `int.itway.fr` :

| Élément                            | Signification                                                      |
| ---------------------------------- | ------------------------------------------------------------------ |
| `zone "int.itway.fr"`              | Déclare la zone DNS **privée** interne                             |
| `type master;`                     | Le serveur est aussi maître ici                                    |
| `file "/etc/bind/db.int.itway.fr"` | Fichier contenant les noms internes (`dmz-web.int.itway.fr`, etc.) |
| `allow-transfer { none; };`        | Toujours interdit, pour sécuriser la zone                          |

> Cette zone est interne uniquement : elle contient des noms non visibles sur Internet, pour éviter de dévoiler des détails de l’infrastructure


* **Reverse DNS** `10.10.10.in-addr.arpa`

| Élément                                      | Signification                                                |
| -------------------------------------------- | ------------------------------------------------------------ |
| `zone "10.10.10.in-addr.arpa"`               | Zone DNS **inverse**, utilisée pour traduire une IP → nom    |
| `file "/etc/bind/db.10.10.10.rev"`           | Fichier contenant les enregistrements PTR                    |
| `type master` et `allow-transfer { none; };` | Identique aux autres zones : autorité principale et sécurité |

> Cette zone est essentielle pour la résolution inverse DNS : elle permet, par exemple, à 10.10.10.3 de répondre qu’il s’agit de mail.itway.fr.

#### Le fichier `named.conf.local.j2` sert à :
- Déclarer quelles zones le serveur DNS va gérer (itway.fr, int.itway.fr, reverse DNS),
- Associer chaque zone à son fichier de données DNS,
- Interdire les transferts de zone, pour des raisons de sécurité.
- Il est indispensable pour que BIND sache comment et quoi servir en tant que DNS maître.

### 7.3 `db.itway.fr.j2`
Le fichier `db.itway.fr.j2` est un élément central dans la configuration DNS. 

##### Le contenu du fichier :
###### A. Le serveur DNS principal (SOA)
>SOA = Start of Authority

C’est le premier enregistrement obligatoire dans un fichier de zone DNS.
Il indique qui est l’autorité principale pour cette zone, et comment elle se gère (temps, rafraîchissement, etc.).

Il déclare le serveur DNS maître pour cette zone et définit des paramètres de gestion comme : 

- Quand les DNS secondaires doivent se synchroniser,
- Quel est le contact admin,
- Quel est le numéro de version de la zone (sérial).

SOA définit dans le playbook : 
```jinja
$TTL 1h
@   IN  SOA ns1.itway.fr. admin.itway.fr. (
        2025052905 ; Serial auto
        1h   ; Refresh
        15m  ; Retry
        1w   ; Expire
        1h ) ; Neg TTL
```

**Explication** :

| Élément             | Description                                                       |
| ------------------- | ----------------------------------------------------------------- |
| `@`                 | Représente le domaine courant (`itway.fr`)                        |
| `IN`                | Classe Internet                                                   |
| `SOA`               | Début d’autorité                                                  |
| `ns1.itway.fr.`     | Nom du serveur DNS maître (serveur principal)                     |
| `admin.itway.fr.`   | Contact admin (→ `admin@itway.fr`)                                |
| `2025052901`        | **Serial** = version du fichier de zone                           |
| `1h` (Refresh)      | Tous les 1h, un esclave vérifie si la zone a changé               |
| `15m` (Retry)       | Si l’esclave échoue, il réessaie dans 15 minutes                  |
| `1w` (Expire)       | Si aucune maj en 1 semaine, il considère la zone invalide         |
| `1h` (Negative TTL) | Durée pour laquelle une réponse "non trouvée" est gardée en cache |


###### B. Les noms et leurs IPs (ex : mail.itway.fr → 10.10.10.3)
C’est ici que l’on associe les noms des machines (hostnames) à leurs adresses IP. Cela permet au serveur DNS de répondre aux clients en traduisant un nom de domaine en adresse IP.

| Élément      | Nom du paramètre   | Signification                                                 |
| ------------ | ------------------ | ------------------------------------------------------------- |
| `ns1`        | **Nom** (ou label) | C’est le nom de l’hôte (dans ce cas, `ns1.itway.fr`)          |
| `IN`         | **Classe**         | Signifie "Internet" (valeur presque toujours `IN`)            |
| `A`          | **Type**           | Type d’enregistrement DNS : ici, un `A` pour une adresse IPv4 |
| `10.10.10.4` | **Valeur**         | C’est l’adresse IP associée au nom `ns1`                      |

- `@` : est un raccourci qui représente le nom de domaine racine de la zone (ici, itway.fr). L’enregistrement MX indique à quel serveur envoyer les emails destinés au domaine.
- `ns1` : ns1.itway.fr pointe vers l’IP 10.10.10.4
- `CNAME` : signifie que **www.itway.fr** redirige vers ce que rproxy.itway.fr pointe (donc évite la redondance).
- `TXT` : Un enregistrement TXT (Text) permet d’ajouter du texte libre dans une zone DNS. Il est souvent utilisé pour la sécurité (SPF, DKIM, DMARC), la vérification de domaine, ou des informations spécifiques aux services.
**Exemple dans le projet :**
```
@   IN  TXT "v=spf1 mx -all"
```
`v=spf1 mx -all` → contenu du texte, ici une politique SPF
>SPF = Sender Policy Framework
C’est un mécanisme de protection contre le spoofing (falsification d’expéditeur dans les emails). Il indique quels serveurs ont le droit d’envoyer des emails au nom du domaine.

### 7.4 `db.int.itway.fr.j2`
Le fichier **db.int.itway.fr.j2** définit une zone DNS privée, non accessible depuis Internet.
Il est utilisé pour la résolution des noms internes à l’entreprise, notamment ceux qui ne doivent être visibles que depuis les réseaux internes de confiance.

#### Qu'est-ce qu'une zone DNS interne ?
C’est un fichier de zone qui décrit des noms et adresses IP internes, souvent associés à des services ou infrastructures non publics.
Dans notre cas, cette zone couvre le domaine **int.itway.fr**, utilisé pour désigner des machines internes de la DMZ.

#### Contenu du fichier :

```jinja
$TTL 1h
@   IN  SOA ns1.int.itway.fr. admin.itway.fr. (
        {{ ansible_date_time.date | regex_replace('-','') }}01
        1h 15m 1w 1h )
        IN  NS  ns1.int.itway.fr.

ns1     IN  A   10.10.10.4
dmz-web IN  A   10.10.10.2
dmz-rt  IN  A   10.10.10.5
```

#### Pourquoi faire un deuxième fichier au lieu de tout mettre dans un seul ?
Parce que itway.fr est une zone publique mais int.itway.fr est une zone privée (interne). Ce sont deux zones DNS distinctes, avec des usages, des accès et des contraintes de sécurité différents. 

La séparation permet une meilleure sécurité, une meilleure organisation, et respecte le fonctionnement technique de Bind9.

>   Il ne faut pas oublier qu'on à déclarer deux zones dans [`named.conf.local.j2`](#namedconflocalj2) et on limite les requêtes récursives dans dans [`named.conf.options.j2`](#namedconfoptionsj2).


### 7.5 `db.10.10.10.rev.j2`
Ce fichier est une zone de résolution inverse (reverse DNS).
Il permet de faire l’inverse de ce que fait un enregistrement A :
Au lieu de traduire un nom en IP, il traduit une IP en nom.

#### Qu'est ce qu'une zone inverse ?
Quand une machine connaît l’IP (ex. 10.10.10.3) et veut connaître le nom de domaine associé, elle effectue une requête PTR (Pointer Record) via une zone dite inverse.

Cette zone est toujours associée à des IPs et suit le format :
x.x.x.in-addr.arpa
```jinja
    2   IN  PTR dmz-web.itway.fr.
    3   IN  PTR mail.itway.fr.
    4   IN  PTR ns1.itway.fr.
    5   IN  PTR rproxy.itway.fr.
```

#### L'avantage d'une zone inverse ? 
- Certains serveurs refusent des connexions SMTP s’il n’y a pas de PTR valide.
- Cela améliore la légitimité DNS de tes services aux yeux d’Internet.
- Utile pour l’analyse de logs réseau.

---

## 7. Sécurité & conformité

| Sécurité appliquée                    | Description                                         |
| ------------------------------------- | --------------------------------------------------- |
| Récursion filtrée                     | `allow-recursion` restreint aux réseaux internes    |
| Pas de transfert                      | `allow-transfer { none; }` pour éviter vol de zones |
| Zone interne privée                   | `int.itway.fr` non exposée via firewall et config   |
| Pas de communication avec DNS interne | Isolation réseau stricte                            |


---

## 8. Liens utiles

* [Playbook complet Ansible et rôles](#structure-du-projet-ansible)

---

## 9. Problème rencontré : permissions `sudo` brisées

### Solution appliquée :

```bash
chown root:root /usr/bin/sudo
chmod 4755 /usr/bin/sudo
```

---

## Conclusion

Ce projet met en place un serveur DNS sécurisé et bien segmenté :

* Zones publiques et internes distinctes.
* Isolation DMZ/infrastructure.
* Ansible pour fiabilité et industrialisation.
* Conforme au cahier des charges.


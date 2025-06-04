# SRV

## SRV – Réseau des Serveurs Internes

### 1. En quoi consiste le SRV-NETWORK ?

Le **SRV-NETWORK** regroupe l’ensemble des **serveurs internes critiques** nécessaires au bon fonctionnement des services métiers et de l’infrastructure de l’entreprise. Contrairement à la DMZ, ces serveurs ne sont **pas accessibles depuis Internet** mais uniquement depuis les réseaux internes de confiance.

Leur rôle est d’assurer **l’authentification**, **la messagerie**, **la gestion des certificats** et la **diffusion de l’information interne**, dans un environnement sécurisé, cloisonné et hautement contrôlé.

---

### 2. Objectifs du SRV-NETWORK dans le projet IT-WAY

* Centraliser les services cœur (authentification, mail, intranet, PKI)
* Sécuriser l’accès et la configuration de chaque serveur
* Garantir un **haut niveau d’intégrité, de confidentialité et de disponibilité**
* Isoler les services sensibles du reste de l’infrastructure

---

### 3. Services présents dans le SRV-NETWORK

| Nom d’hôte   | Rôle                                              | Domaine               |
| ------------ | ------------------------------------------------- | --------------------- |
| SRV-ADS01    | Contrôleur de domaine Active Directory            | srv-ads01.itway.local |
| SRV-INTRANET | Serveur web intranet interne                      | intranet.itway.local  |
| SRV-MAIL     | Serveur de messagerie interne (MDA/IMAP/POP)      | srv-mail.itway.local  |
| SRV-PKI      | Serveur d’autorité de certification interne (PKI) | srv-pki.itway.local   |

---

### 4. Adressage IP – Réseau SRV

Le réseau SRV-NETWORK est adressé selon le plan **10.10.20.0/24**, dédié exclusivement aux serveurs internes.

| Machine      | Adresse IP | Interface accessible                       |
| ------------ | ---------- | ------------------------------------------ |
| SRV-ADS01    | 172.16.50.2 | Réseaux internes de confiance              |
| SRV-INTRANET | 172.16.50.5 | Réseaux internes (site intranet)           |
| SRV-MAIL     | 172.16.50.4 | DMZ-SMTP (relai) + Réseaux internes        |
| SRV-PKI      | 172.16.50.3 | IT-NET (admin) + internes (liste CRL only) |

---

### 5. Résumé technique

* Tous les accès d’administration se font **depuis IT-NET uniquement**
* Les services utilisent des **certificats signés par SRV-PKI**
* **SRV-ADS01** gère l’authentification, les GPO, profils itinérants, IPSec, DNS interne
* **SRV-INTRANET** propose un site sécurisé avec authentification LDAP
* **SRV-MAIL** travaille en collaboration avec **DMZ-SMTP** pour la distribution du courrier
* **SRV-PKI** est le **centre de confiance** pour les certificats SSL/VPN utilisés dans toute l’infrastructure

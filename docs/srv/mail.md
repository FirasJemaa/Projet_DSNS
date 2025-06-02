# Serveur Mail SRV-MAIL : Cours Complet & Guide de Mise en Place

## 1. Introduction : Qu'est-ce qu'un serveur mail ?

Un serveur de messagerie (ou serveur mail) est une machine chargée de **transmettre, recevoir et stocker** les courriels (émails). Il est constitué de plusieurs composants logiciels, chacun assurant une fonction bien définie :

* **MTA (Mail Transfer Agent)** : transmet les mails d'un serveur à un autre (Postfix)
* **MDA (Mail Delivery Agent)** : délivre le mail dans la boîte de réception (Dovecot)
* **IMAP/POP3** : protocoles permettant aux clients (Serveur SMTP) de récupérer leurs mails
* **SMTP** : protocole d’envoi des mails

Dans ce projet, nous construisons un serveur **MDA + IMAP/POP**.

## 2. Rôle de SRV-MAIL dans l'infrastructure

| Hôte     | Domaine              | OS     | Rôle principal                                |
| -------- | -------------------- | ------ | --------------------------------------------- |
| SRV-MAIL | srv-mail.itway.local | Debian | Stockage et distribution des e-mails internes |

Le serveur **SRV-MAIL** reçoit les mails transmis par **DMZ-SMTP**, les stocke dans des boîtes utilisateurs, et les met à disposition via **IMAP/POP**. Il utilise une base **MariaDB** pour gérer les comptes, via l’interface **PostfixAdmin**.

## 3. Solutions logicielles choisies

| Fonction         | Logiciel choisi | Raison du choix                                       |
| ---------------- | --------------- | ----------------------------------------------------- |
| MDA / SMTP local | Postfix         | Léger, très populaire, facile à intégrer avec Dovecot |
| IMAP/POP3        | Dovecot         | Performance, sécurité, support SQL                    |
| Base de données  | MariaDB         | Fiable, communautaire, compatible MySQL               |
| Interface admin  | PostfixAdmin    | Interface web simple pour gérer domaines/utilisateurs |
| Authentification | Dovecot SQL     | Permet l’auth avec une base MariaDB                   |

## 4. Protocoles de messagerie utilisés

| Protocole  | Port | Utilité                                         |
| ---------- | ---- | ----------------------------------------------- |
| SMTP       | 25   | Envoi de mails (entre serveurs)                 |
| SMTPS      | 465  | Envoi de mails avec SSL (depuis clients)        |
| Submission | 587  | Envoi de mails avec authentification (STARTTLS) |
| IMAP       | 143  | Récupération des mails depuis un client         |
| IMAPS      | 993  | Idem, avec chiffrage SSL                        |
| POP3       | 110  | Récupération, suppression locale des mails      |
| POP3S      | 995  | POP3 chiffré                                    |

Dovecot est responsable de la gestion IMAP/POP, tandis que Postfix gère SMTPS (465).

## 5. Architecture fonctionnelle de SRV-MAIL

```
Client mail  <--- IMAP/POP3 (SSL) --->  SRV-MAIL  <--- SMTP --- DMZ-SMTP  <--> Internet
                            ^
                            |
                  MariaDB (auth, boîtes)
                            |
                     PostfixAdmin (admin web)
```

## 6. Mise en place technique

### a. Sécurisation SSL/TLS

* Certificat généré par le serveur **SRV-PKI** interne
* Déployé dans `/etc/postfix/ssl`
* Utilisé pour chiffrer SMTPS/IMAPS

### b. Stockage des mails

* Format **Maildir** dans `/var/mail/vhosts/<domaine>/<utilisateur>/Maildir/`
* Droit utilisateur `vmail` UID/GID 5000

### c. Authentification SQL

* Dovecot interroge MariaDB pour valider les utilisateurs
* Requêtes SQL définies dans `dovecot-sql.conf.ext`

### d. PostfixAdmin

* Interface de gestion des boîtes
* Setup via `config.local.php` + mot de passe chiffré

### e. Intégration avec DMZ-SMTP

* DMZ-SMTP relaie tous les mails entrants vers SRV-MAIL
* SRV-MAIL ne communique pas directement avec Internet

## 7. Bonnes pratiques mises en place

* Services accessibles uniquement en TLS : SMTPS, IMAPS
* Aucune auth en clair autorisée (`disable_plaintext_auth = yes`)
* Utilisation d'un utilisateur système dédié `vmail`
* Accès limités à MariaDB avec utilisateurs spécifiques
* Structure des répertoires conforme Maildir
* Gestion via Ansible et Vault (pas de mot de passe en clair)

## 8. Configuration technique (Playbook Ansible)

➡️ Voir tous les rôles, templates et playbooks ici :
📁 [Répertoire Ansible SRV-MAIL](./roles/srv-mail/)

---

## 9. Conclusion

Le serveur **SRV-MAIL** constitue une brique centrale de l'infrastructure ITWay.

En s'appuyant sur des composants open source robustes (Postfix, Dovecot, MariaDB), il permet de distribuer les e-mails de manière sécurisée à tous les utilisateurs internes, tout en étant administrable et extensible.

La séparation des rôles (DMZ-SMTP pour le relai, SRV-MAIL pour le stockage) permet un **contrôle granulaire de la sécurité**, tout en répondant aux bonnes pratiques système et réseau.

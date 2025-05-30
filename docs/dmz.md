# DMZ

## DMZ – Zone Démilitarisée

### 1. En quoi consiste la DMZ ?

La **DMZ (Demilitarized Zone)** est une zone réseau intermédiaire entre Internet et le réseau interne de l’entreprise. Son rôle principal est de **protéger l’infrastructure interne** tout en permettant **l’accès public** à certains services.

Elle sert de **barrière de sécurité** pour exposer uniquement les services nécessaires (web, DNS, SMTP…) à Internet tout en **isolant** les serveurs internes sensibles.

### 2. Objectifs de la DMZ dans le projet IT-WAY

* Héberger les services accessibles depuis Internet
* Garantir une séparation stricte avec les réseaux internes
* Appliquer des règles de filtrage et de contrôle des flux réseau
* Limiter les risques en cas de compromission d’un service public

---

### 3. Services présents dans la DMZ

| Nom d’hôte | Rôle                                     | Domaine                         |
| ---------- | ---------------------------------------- | ------------------------------- |
| DMZ-RPROXY | Reverse Proxy pour les services web      | itway.fr / webmail.itway.fr     |
| DMZ-WEB    | Serveur web public WordPress (vitrine)   | itway.fr / dmz-web.int.itway.fr |
| DMZ-DNS    | Serveur DNS public pour la zone itway.fr | dns.itway.fr                    |
| DMZ-SMTP   | Serveur relais mail (entrée/sortie)      | mail.itway.fr                   |
| DMZ-RT     | Routeur DMZ (Stormshield EVA)            | dmz-rt.int.itway.fr             |

---

### 4. Adressage IP – Réseau DMZ

Nous avons choisi le **plan d’adressage 10.10.10.0/24** pour le réseau DMZ, en cohérence avec les autres sous-réseaux de l’infrastructure IT-WAY.

| Machine    | Adresse IP   | Interface accessible                           |
| ---------- | ------------ | ---------------------------------------------- |
| DMZ-RPROXY | 10.10.10.5   | Internet, Interne                              |
| DMZ-WEB    | 10.10.10.2   | Internet, Interne                              |
| DMZ-DNS    | 10.10.10.4   | Internet, Interne                              |
| DMZ-SMTP   | 10.10.10.3   | Internet, Interne                              |
| DMZ-RT     | 10.10.10.1   | Connexion vers CORE-RT et Internet             |

---

### 5. Résumé technique

* Routage filtré via **DMZ-RT → CORE-RT**
* Chaque serveur dispose d’un **certificat TLS signé par la PKI interne**
* Tous les accès en SSH ou administration se font **depuis IT-NET uniquement**
* Zone DNS distincte pour `int.itway.fr` non exposée à Internet



# Mise en place d'un Reverse Proxy Sécurisé avec Nginx dans une Infrastructure en DMZ

## 1. Introduction Théorique

### Qu'est-ce qu'un Reverse Proxy ?

Un **reverse proxy** est un serveur intermédiaire qui reçoit les requêtes des clients et les transmet à un ou plusieurs serveurs backend. Contrairement à un proxy classique (forward proxy), qui agit pour le compte de clients internes vers l'extérieur, le reverse proxy agit pour le compte de serveurs internes vers les clients.

### Avantages d'un Reverse Proxy

* **Sécurité** : Le reverse proxy masque l'architecture interne.
* **SSL/TLS** : Gestion centralisée des certificats et du chiffrement HTTPS.
* **Load Balancing** : Peut répartir les requêtes entre plusieurs serveurs backend.
* **Cache** : Peut réduire la charge des serveurs backend.
* **Compression et Optimisation** : Amélioration des performances.

### Rôle dans notre Projet

Dans notre projet, le reverse proxy est situé dans la **DMZ** (zone démilitarisée) et agit comme unique point d'entrée HTTP/HTTPS pour des services internes, notamment :

* `itway.fr` (site institutionnel)
* `webmail.itway.fr` (interface Roundcube + PostfixAdmin)

Ce reverse proxy redirige les requêtes HTTPS entrantes vers les serveurs internes `DMZ-WEB`, `DMZ-SMTP`, etc., tout en appliquant une couche de sécurité via SSL/TLS.

## 2. Choix Techniques

### Serveur HTTP

* **Nginx** est choisi pour sa légèreté, ses performances, et son support natif de la configuration reverse proxy.

### Certificats SSL/TLS

* Générés et signés par une **PKI interne** (SRV-PKI).
* Stockés localement sur DMZ-RPROXY.
* **fullchain.crt** : contient le certificat + chaîne d'intermédiaires jusqu'’à la CA.

### Outils d'automatisation

* **Ansible** : Utilisé pour automatiser l'installation, la configuration et la génération des certificats.

## 3. Architecture de la Solution

```
    [Client] <--HTTPS--> [DMZ-RPROXY] <--HTTP--> [DMZ-WEB/DMZ-SMTP]
                                   |
                                   |---> 10.10.10.2 (Web)
                                   |---> 10.10.10.3 (Webmail)
                                   |---> 172.16.50.4 (PostfixAdmin)
```

## 4. Étapes de Mise en Place

### a) Configuration DNS

* `itway.fr` et `webmail.itway.fr` doivent pointer vers l'adresse IP du reverse proxy : `10.10.10.5`

### b) Génération des Certificats

* Génération de la **clé privée** et de la **CSR** sur `DMZ-RPROXY`.
* Transfert de la CSR vers `SRV-PKI` pour signature.
* Récupération du certificat signé et de la **CA**.
* Création d'un **fullchain.crt** pour la configuration Nginx.

### c) Installation de Nginx

* Installation via apt.
* Activation du module SSL.

### d) Configuration de Nginx

* Mise en place des fichiers suivants :

  * `default.conf` : gère `itway.fr`
  * `webmail.conf` : gère `webmail.itway.fr`
  * `ssl_params.conf` : durcit la configuration SSL (TLS1.2+, ciphers sécurisés)
* Chaque fichier est lié via `sites-enabled/`

### e) Reverse Proxy : `location` et `proxy_pass`

* Redirection des chemins vers les IPs internes.
* Headers HTTP ajoutés pour transparence : `Host`, `X-Real-IP`, `X-Forwarded-Proto`

### f) Test et Vérification

* Commandes CURL depuis l’intérieur et l’extérieur.
* Vérification du certificat via navigateur.
* Vérification du code HTTP (301 pour redirection, 200 pour succès).

### g) Tests Ansible automatisés

* `uri:` module pour tester HTTP et HTTPS
* Code de retour attendu : 301 et 200

## 5. Points de Sécurité

* TLS 1.2 et 1.3 uniquement.
* Clé privée jamais copiée en clair.
* Proxy interne en HTTP (environnement maîtrisé).
* Reverse proxy positionné en DMZ avec firewall.

## 6. Playbooks Ansible

Tous les playbooks liés à la configuration (certificat, nginx, vhost, tests) sont disponibles dans le répertoire `roles/dmz-rproxy/`.

➡️ **Voir les playbooks ici :** \[Lien vers les fichiers Ansible] (indiquez le chemin ou repo Git ici).

---

## 7. Conclusion

Cette architecture assure une **sécurité périmétrique** efficace, une gestion centralisée du chiffrement, et une facilité d’administration via Ansible. Elle permet de séparer les accès publics des services internes tout en conservant un haut niveau de sécurité grâce à l’utilisation d’une PKI interne et des bonnes pratiques SSL/TLS.

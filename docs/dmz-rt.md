# Routeur DMZ – DMZ-RT

## Présentation

Le routeur DMZ-RT est dédié à l’isolation de la zone DMZ. Il permet la communication entre les services exposés à Internet (SMTP, Web, DNS) et l’extérieur, tout en assurant un cloisonnement strict vis-à-vis du réseau interne.

---

## Déploiement

- Matériel utilisé : Stormshield EVA – version 4.3.33
- Interface IN connectée à DMZ-SW
- Interface OUT connectée au routeur CORE-RT

---

## Configuration initiale

- Adresse IP : `192.168.86.2/24`
- Accès administrateur : `admin / FifiNana84+`
- Interface Web activée pour supervision via un poste Ubuntu
- Connexion testée depuis le réseau DMZ avec succès

---

## Services activés

- **Filtrage pare-feu strict** entre la DMZ et l’interne
- **NAT sortant** configuré pour les services de la DMZ (Web, SMTP)
- **Accès entrants filtrés** depuis Internet (port 80/443, SMTP uniquement)
- **DHCP désactivé** (adressage statique utilisé)

---

## Sécurité

- Aucun accès DMZ → Interne, sauf exceptions précises (SMTP vers SRV-MAIL)
- Accès SSH/Web réservé au réseau IT-NET
- Objets réseaux créés pour chaque serveur DMZ
- Règles spécifiques appliquées pour chaque service public exposé
- Sauvegardes manuelles enregistrées localement

---

## Observations

- Routeur testé et fonctionnel
- Intégration réussie avec le reverse proxy, serveur Web et serveur Mail
- Compatible avec les VLANs issus du CORE-SW (via trunking)

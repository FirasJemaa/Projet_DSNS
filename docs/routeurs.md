# Configuration des Routeurs

## Rôle des routeurs dans l'infrastructure

Dans le cadre de l’infrastructure ITWay, nous avons utilisé deux routeurs principaux :

- **CORE-RT** : routeur central connecté à tous les sous-réseaux internes,
- **DMZ-RT** : routeur de la DMZ, isolé du reste du réseau pour sécuriser les services exposés à Internet.

Ces routeurs sont essentiels pour :

- Segmenter les différents réseaux (IT, SRV, DMZ, LAN…),
- Appliquer des politiques de filtrage strictes,
- Gérer le routage entre les sous-réseaux internes,
- Assurer la sécurité grâce à la configuration des ACLs, NAT, VPN, etc.

---

## Routeur CORE-RT (Stormshield EVA)

**Fonctions principales :**

- Connexion entre tous les sous-réseaux internes
- Fourniture de la passerelle par défaut
- Configuration des VLANs et du plan d’adressage
- Politique de filtrage entre zones
- Gestion du VPN SSL pour l’accès distant

**Exemples de configuration appliquée :**

- Interfaces VLAN configurées pour chaque réseau (IT-NET, SRV-NET, DMZ-TUNNEL, etc.)
- Routage inter-VLAN activé
- VPN SSL accessible depuis vpn.itway.fr
- NAT pour la DMZ et les services internes
- Filtrage Web/SSH accessible uniquement depuis IT-NET
- Authentification admin via Active Directory (GRP-SECU-IT)

**Sécurité appliquée :**

- Accès Web/SSH uniquement depuis le réseau IT
- VPN SSL authentifié via certificats internes
- Interfaces d’administration restreintes
- Politique par défaut : tout trafic entrant bloqué
- Protocoles ICMP autorisés dans les deux sens
- Certificats signés par le serveur PKI (SRV-PKI)

## Routeur DMZ-RT (Stormshield EVA)

**Fonctions principales :**

- Séparer le réseau DMZ du réseau interne
- Assurer la communication vers Internet
- Appliquer des règles de filtrage renforcées

**Principes appliqués :**

- Aucun accès de la DMZ vers l’interne sauf exceptions justifiées (SMTP vers SRV-MAIL par exemple)
- Interfaces d’administration accessibles uniquement depuis IT-NET
- NAT configuré pour les services publics (Web, Mail, etc.)

## Conclusion
Les routeurs Stormshield ont été configurés pour garantir compartimentage, sécurité des flux, et gestion centralisée. Leur intégration dans GNS3 a permis de tester et valider toutes les règles avant mise en production simulée.
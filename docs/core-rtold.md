# Routeur Principal – CORE-RT (BORDER-RT)

## Présentation

Le routeur CORE-RT (renommé BORDER-RT) constitue le cœur de l’infrastructure. Il assure la connexion entre les sous-réseaux internes (LAN, IT, SRV) et la DMZ, ainsi que l’accès vers Internet.

Il a été déployé et configuré dans **GNS3** à l’aide du pare-feu **Stormshield EVA**.

---

## Déploiement

- Template importé : Stormshield EVA – version 4.3.33
- Connexion : 
    - Port OUT relié à Internet
    - Port IN relié à un switch CORE-SW
    - Port DMZ relié au DMZ-RT

---

## Configuration initiale

- Accès console via VNC
- Clavier configuré en AZERTY (`sudo loadkeys fr`)
- IP statique sur interface IN : `192.168.84.1/24`
- Interface OUT configurée en DHCP
- Interface Web accessible depuis un poste client Ubuntu

---

## Services activés

- **DHCP** sur les interfaces internes (SRV, LAN, etc.)
- **VPN SSL** pour accès distant via `vpn.itway.fr`
- **Routage inter-VLAN**
- **NAT sortant** pour l'accès Internet depuis les réseaux internes
- **Filtrage pare-feu personnalisé** (par VLAN, par service)
- **Accès administration restreint** à IT-NET uniquement

---

## VLANs configurés
<span style="color:red">REVOIR</span>

| VLAN      | Nom réseau   | Plage IP               | Gateway          |
|-----------|--------------|------------------------|------------------|
| 10        | NET-IT       | 172.16.86.0/27         | 172.16.86.1      |
| 31        | Direction & Administratif | 192.168.90.0/26 | 192.168.90.1 |
| 122       | WAN/OUT      | DHCP (Internet)        | -                |

---

## Sécurité

- Accès Web/SSH uniquement depuis IT-NET
- Authentification via comptes AD membres du groupe `GRP-SECU-IT`
- Filtrage ICMP activé (dans les deux sens)
- NAT configuré pour masquer les IP internes
- Mise en place de sauvegardes automatiques

---

## Sauvegarde

- Sauvegarde hebdomadaire manuelle (sur machine Windows)
- Objectif futur : sauvegarde automatique vers serveur HTTPS
- Test de restauration à prévoir

---

## Observations

- L’infrastructure a été testée avec succès (ping 8.8.8.8 OK)
- Passage du switch Cisco à un switch Ethernet pour compatibilité
- Renommage de l’équipement dans l’interface Stormshield : `BORDER-RT`

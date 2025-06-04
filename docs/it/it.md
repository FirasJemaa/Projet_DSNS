# IT

## IT – Réseau Administratif et Supervision

### 1. En quoi consiste le IT-NETWORK ?

Le **IT-NETWORK** est un sous-réseau dédié exclusivement à l’**administration**, au **déploiement automatisé** et à la **supervision** de l’ensemble de l’infrastructure IT-WAY. Il est isolé des autres réseaux pour garantir un **niveau de sécurité maximal**.

Seuls les administrateurs y ont accès, et tous les services présents dans ce réseau sont critiques pour le bon fonctionnement, la maintenance et le suivi de l’infrastructure.

---

### 2. Objectifs du IT-NETWORK dans le projet IT-WAY

* Centraliser les outils d’administration, de supervision et d’automatisation
* Isoler les outils d’exploitation du reste de l’infrastructure
* Offrir un environnement sécurisé pour les tâches sensibles
* Fournir une **interface unique de gestion** pour tous les services déployés

---

### 3. Services présents dans le IT-NETWORK

| Nom d’hôte  | Rôle                                    | Domaine                 |
| ----------- | --------------------------------------- | ----------------------- |
| IT-ANSIBLE  | Déploiement et gestion de configuration | it-ansible.itway.local  |
| IT-GRAPHANA | Supervision et monitoring de l’infra    | it-graphana.itway.local |
| IT-MGMT     | Poste de travail d’administration       | it-mgmt.itway.local     |

---

### 4. Adressage IP – Réseau IT

Le réseau IT-NETWORK est adressé sur la plage **172.16.10.0/24**, réservé à l’administration uniquement.

| Machine     | Adresse IP | Interface accessible                          |
| ----------- | ---------- | --------------------------------------------- |
| IT-ANSIBLE  | 172.16.10.2 | Déploiement vers tous les réseaux internes    |
| IT-GRAPHANA | 172.16.10.3 | Interface Web accessible uniquement depuis IT |
| IT-MGMT     | 172.16.10.4 | Accès complet à toute l’infrastructure        |

---

### 5. Résumé technique

* Accès **restreint** au réseau IT : seuls les administrateurs y sont autorisés
* **IT-ANSIBLE** déploie les configurations via playbooks Ansible
* **IT-GRAPHANA** supervise l’ensemble des équipements (serveurs, routeurs, services)
* **IT-MGMT** est le poste central d’administration (Windows) avec tous les outils nécessaires
* Tous les équipements du réseau IT sont **administrables uniquement depuis ce réseau**
* Aucun service de ce réseau n’est accessible directement depuis Internet

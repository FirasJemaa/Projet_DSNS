# Conception et Organisation du Projet ITWay

## Introduction

Le projet ITWay a été réalisé dans le cadre de notre formation DSNS – option Cybersécurité. Il s’agit d’un projet de fin d’année ayant pour objectif la conception, le développement et la documentation complète d’une infrastructure système et réseau sécurisée, selon un cahier des charges précis.

Ce document présente l’organisation mise en place, la répartition des tâches, ainsi que le déroulement du projet.

---

## Cahier des charges
Le cahier des charges centralise toutes les attentes fonctionnelles et les contraintes techniques du projet IT-WAY. Il sert de référence tout au long de la mise en œuvre. [Vous pouvez le consulter ici](./others/Projet%20ITWay%20-%20Phase%202.pdf).

---

## Répartition des rôles

Nous avons travaillé en binôme, avec une répartition des responsabilités claire et complémentaire :

- **Ensemble**
    - Conception de l’architecture réseau
    - Configuration des routeurs, des VLAN et du plan d’adressage
    - Mise en place de la sécurité réseau (pare-feu, filtrage, VPN)
    - Supervision (Grafana)
    - Déploiement du service PKI
    - Rédaction de la documentation technique des services réseau

- **Firase JEMAA**
    - Automatisation des configurations avec Ansible
    - Déploiement des services dans la DMZ (serveur web, DNS, reverse proxy, serveur SMTP)
    - Intégration du serveur mail avec le serveur SMTP
    - Rédaction de la documentation

- **Najet BOUKADOUR**
    - Déploiement de l’Active Directory et configuration du DNS interne
    - Mise en place des stratégies de groupe (GPO) et des unités d'organisations (OU)
    - Déploiement du serveur Grafana et Wazuh
    - Rédaction de la documentation


---

## Méthodologie de travail

Nous avons adopté une approche structurée en plusieurs étapes :

Nous avons commencé par concevoir la structure du réseau ensemble, afin que chacun maîtrise les fondamentaux et que nous soyons sur la même longueur d’onde. Par la suite, nous avons réparti les tâches en fonction des points forts de chaque membre de l'équipe.

Nous avons également convenu d’organiser une réunion hebdomadaire de 4 heures pour faire un point sur l’avancement de chacun et échanger sur les éventuelles difficultés rencontrées. Le partage des documentations, des playbooks et autres fichiers s’est fait via un dépôt GitHub commun, tandis que le suivi des tâches et du planning a été géré avec l’outil Trello.

---

## Déroulement du projet
Au départ, nous avons commencé par la mise en place de l’infrastructure, en prenant en compte l’ensemble de l’architecture afin d’avoir une vision globale de chaque section et composant. Cela a impliqué la création et la configuration des VLANs nécessaires.

Une fois cette base posée, nous avons commencé par la configuration des routeurs, afin de permettre la connectivité entre les différentes zones, mais sans appliquer de règles de pare-feu dans un premier temps. Ensuite, nous avons configuré les switches et vérifié que le routage inter-VLAN fonctionnait correctement, notamment à travers le service DHCP hébergé sur le routeur.

Après cela, nous sommes passés à l’infrastructure des machines virtuelles.

Nous avons d’abord déployé la machine Ansible ainsi que la machine PKI, car Ansible nous permettait de déployer et configurer automatiquement les autres serveurs. En parallèle, nous avons également commencé la mise en place de l’Active Directory (AD).

Par la suite, chacun a configuré les services qui lui étaient attribués, tout en partageant régulièrement l’avancement pour coordonner les étapes suivantes et garantir une cohérence dans le déploiement global.
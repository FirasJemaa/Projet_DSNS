## Phase 2 — Conception, Déploiement et Maintenance

### Présentation de la Phase 2

Après l’analyse initiale réalisée lors de la première phase, la **Phase 2** marque l’entrée dans la phase opérationnelle du projet. Cette étape a pour objectif de **concevoir, déployer et documenter une infrastructure système et réseau fonctionnelle**, sécurisée et conforme aux exigences identifiées.

Nous avons mis en œuvre les éléments techniques définis dans la phase précédente, en choisissant les outils et technologies adaptés, en installant et configurant les services, et en assurant la traçabilité des choix et des procédures.

### Objectifs de la phase

* **Choix des technologies** : sélection rigoureuse des solutions logicielles et matérielles (routeurs, serveurs, services réseau, outils de supervision, etc.).
* **Déploiement de l'infrastructure** : installation et configuration des services comme Active Directory, DNS, messagerie, VPN, Reverse Proxy, supervision, etc.
* **Sécurité intégrée** : application des bonnes pratiques en cybersécurité, segmentation réseau, authentification renforcée, politique de mot de passe, monitoring.
* **Documentation technique** : production de fiches techniques, guides d’installation et de configuration pour assurer la maintenance et la reproductibilité du système.
* **Tests et validation** : mise en œuvre de scénarios de test pour s'assurer du bon fonctionnement de l’infrastructure.

### Composants mis en œuvre

Voici quelques éléments clés de l’infrastructure déployée :

* **Réseau segmenté** en trois zones : LAN, DMZ, SRV.
* **Active Directory** avec GPO pour gestion centralisée des utilisateurs et des droits.
* **Serveur VPN IPsec (Stormshield)** intégré à l’annuaire AD.
* **Reverse Proxy NGINX** jouant le rôle de WAF avec sécurisation HTTPS.
* **Serveur de messagerie Postfix** ou **Nextcloud** pour la collaboration.
* **Supervision avec Zabbix / Prometheus**, alertes configurées.
* **Infrastructure de certificats (PKI)** pour sécurisation SSL/TLS.
* **Outils d’automatisation** comme Ansible et Vagrant pour le provisioning.

### Livrables

* **Documentation technique complète** : configuration des services, captures d’écrans, explications des choix techniques.
* **Scripts et fichiers de configuration** (dans un dépôt Git).
* **Plans de tests** et résultats des tests (accès, sécurité, disponibilité).
* **Procédures de maintenance** pour la pérennité de l’infrastructure.


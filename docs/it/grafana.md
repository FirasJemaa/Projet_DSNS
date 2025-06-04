## **IT-GRAFANA** (`it-graphana.itway.local`)

### **Cahier des charges**

Le serveur IT-GRAFANA est un serveur de monitoring qui assure la supervision de l’ensemble des équipements de l’entreprise. Il est accessible pour son interface WEB depuis le réseau IT uniquement. Il est configuré pour superviser l’ensemble des équipements de l’entreprise

### **Objectif et rôle de Grafana**

Grafana est un outil de visualisation et de supervision open source.  
 Il permet de créer des tableaux de bord dynamiques, d’agréger des métriques provenant de multiples sources (systèmes, bases de données, API…), et d’afficher des données sous forme de graphiques, jauges ou alertes visuelles.

Dans ce projet, Grafana a été installé comme point central de supervision de l'infrastructure, permettant aux équipes IT de surveiller les services critiques (pare-feux, serveurs, switches, etc.) au sein du réseau itway.local.

Le serveur `IT-GRAPHANA` est déployé sur l’IP **172.16.10.3**

### 

### **Contraintes rencontrées : incompatibilité avec systemd**

Lors de l’installation, une difficulté majeure est apparue :

L’installation par défaut de Grafana tente de lancer un service `grafana-server` via `systemd`.  
Or, dans un conteneur Docker (environnement utilisé ici), `systemd` n’est pas disponible, ce qui provoquait une erreur critique lors de l’installation avec Ansible.

### **Solution de contournement : usage de supervisord**

Pour contourner cette limitation, nous avons utilisé supervisord, un gestionnaire de processus léger compatible avec les environnements Docker.  
Celui-ci permet de gérer le processus `grafana-server` sans avoir recours à `systemd`.

### **Qu’est-ce que `supervisord` ?**

`supervisord` est un gestionnaire de processus léger, écrit en Python, qui permet de démarrer, surveiller et redémarrer automatiquement des processus (services, scripts, applications...) sur un système Linux.

Contrairement à `systemd`, qui est le système d'initialisation complet des distributions modernes, `supervisord` est indépendant de l'init système. Il est donc idéal dans des environnements où `systemd` n’est pas disponible (comme dans notre cas avec les conteneurs Docker ou autre environnement allégé)

## **Déploiement avec Ansible**

Nous avons conçu un rôle Ansible dédié, `it-grafana`, permettant d'automatiser entièrement l’installation et la configuration du serveur Grafana (reproductibilité et maintenance simplifiées). Voici une vue d’ensemble des étapes clés :

### **Playbook initial** 

Il utilisait systemd ce qui nous causait des erreurs : le démarrage ne fonctionnait pas.  
`- name: Installer Grafana`   
  `apt:`  
    `name: grafana`  
    `state: present`

`- name: Démarrer et activer grafana-server`  
  `systemd:`  
    `name: grafana-server`  
    `enabled: yes`  
    `state: started`

⚠️ **Problème** : Ce code échoue dans Docker car `systemd` n’est pas présent, d'où l’erreur :  
 `System has not been booted with systemd as init system (PID 1). Can't operate.`

### **Playbook final (version corrigée)**

`- name: Installer Grafana sans démarrer le service systemd`  
  `apt:`  
    `name: grafana`  
    `state: present`  
    `update_cache: yes`

`- name: Créer le fichier de configuration supervisord pour grafana`  
  `copy:`  
    `dest: /etc/supervisor/conf.d/grafana.conf`  
    `content: |`  
      `[program:grafana]`  
      `command=/usr/sbin/grafana-server --config=/etc/grafana/grafana.ini --homepath=/usr/share/grafana`  
      `autostart=true`  
      `autorestart=true`  
      `stdout_logfile=/var/log/grafana.log`  
      `stderr_logfile=/var/log/grafana.err`

`- name: Créer les dossiers de log pour Grafana`  
  `file:`  
    `path: "{{ item }}"`  
    `state: directory`  
    `mode: '0755'`  
  `loop:`  
    `- /var/log`  
    `- /var/log/grafana`

`- name: Vérifier si supervisord est lancé`  
  `shell: pgrep supervisord || /usr/bin/supervisord -c /etc/supervisor/supervisord.conf`  
  `changed_when: false`

`- name: Recharger la configuration de supervisord`  
  `command: supervisorctl reread && supervisorctl update`

### **Détail des tâches**

| Tâche | Description |
| ----- | ----- |
| `apt` | Installe Grafana sans tenter de démarrer le service via `systemd` |
| `copy` | Génère un fichier de configuration pour `supervisord` afin de gérer le processus `grafana-server` |
| `file` | S’assure que les répertoires de logs existent pour éviter les erreurs d’exécution |
| `shell` | Démarre supervisord uniquement s’il n’est pas déjà en cours d’exécution |
| `command` | Recharge la configuration pour prendre en compte le nouveau programme supervisé (Grafana) |

### **Sécurité et accessibilité**

Accès restreint à l’interface Grafana via le port `3000`, autorisé uniquement depuis le réseau IT. HTTPS via le reverse proxy n’est donc pas requis ici.  
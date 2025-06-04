# **SRV-ADS01 Windows Server 2022** 

## Objectifs  :

1. **Rôle du serveur** : SRV-ADS01 est un contrôleur de domaine Active Directory chargé de l’authentification des utilisateurs et des machines, accessible depuis tous les réseaux internes de confiance.  
2. **Gestion Active Directory** : Concevoir une architecture AD cohérente pour gérer les utilisateurs, groupes, machines et stratégies de groupe (GPO) selon les besoins de l’entreprise.

**Groupes de sécurité à créer :**

- GRP-SECU-IT : Administrateurs du service informatique.  
- GRP-SECU-COM : Administrateurs du service communication.  
- GRP-SECU-DIR : Administrateurs du service commercial.  
- GRP-SECU-PROD : Administrateurs du service production.  
- Autres groupes selon les besoins identifiés.

**Configurations spécifiques :**

**→Profils utilisateurs** :

- Configurer des profils itinérants stockés dans un dossier dédié.  
- Attribuer un lecteur réseau personnel (X:) à chaque utilisateur, monté automatiquement.  
- Limiter chaque dossier personnel à un quota de 100 Mo.

**→Restrictions et sécurité** :

- Bloquer l’enregistrement de fichiers exécutables (.exe) sur les lecteurs réseau.  
- Afficher un message d’accueil à la connexion : « Les accès à ce poste de travail sont réservés aux utilisateurs autorisés. Toute tentative d’accès non autorisée sera sanctionnée. »  
- Activer l’audit des connexions/déconnexions et des modifications des stratégies.  
- Configurer le chiffrement RDP à un niveau élevé.  
- Désactiver le compte invité sur tous les postes.  
- Implémenter IPSec avec authentification Kerberos.

**→Pare-feu Windows** :

- Autoriser uniquement les services WINRM, RDP, Remote Assistance, Network Discovery et autres services nécessaires.

**→Gestion DNS** :

- Créer une entrée DNS statique pour chaque machine interne sur le serveur DNS interne.  
- Configurer les zones DNS inverses pour les réseaux internes.

## 1\. Mise en place

### 1.1. Préparation du serveur

- `ncpa.cpl` → IP statique  
- DNS primaire : IP du serveur AD  
- DNS secondaire : Google (8.8.8.8) ou Cloudflare (1.1.1.1)  
- Renommer le serveur → `SRV-ADS01` → redémarrage

### 1.2. Installation du rôle ADDS \+ DNS

- Gestionnaire de serveur → Gérer → Ajouter des rôles  
- Cocher : **Active Directory Domain Services** et **DNS Server**  
- Installer \> Fermer après l'installation

### 1.3. Promouvoir en contrôleur de domaine

- Cliquer sur la notification jaune → *Promouvoir ce serveur…*  
- **Nouvelle forêt** → nom : `itway.local`  
- Niveau fonctionnel : `Windows Server 2016`  
- Définir le mot de passe DSRM  
- Valider les options DNS et NetBIOS  
- Suivant jusqu'à la vérification → Installer → Redémarrage

### 1.4. Configurer le DNS 

Gestionnaire DNS → vérifier :  
\-Zone de recherche directe : `itway.local`  
\-Présence de `SOA`, `NS`, `A`

Test de la résolution DNS avec :   
`nslookup itway.local`  
`nslookup [IP du serveur]`

### 1.5. Création zone de recherche inversée

- DNS \> clic droit sur “Zone de recherche inversée” \> Nouvelle zone  
- Zone principale \> enregistrer dans AD \> nom : `IPv4`  
- ID réseau : `172.16.50.2`  
- Créer un enregistrement pointeur (`PTR`) :  
  - IP : 172.16.50.2  
  - Nom d'hôte : `SRV-ADS01.itway.local`

- Vérifier avec : 

`nslookup 172.16.50.2`

## 2\. Création de l’architecture Active Directory

### 2.1 Création des UO

L’objectif est de structurer les utilisateurs et ordinateurs dans des **OU** logiques pour appliquer des GPO ciblées.

Dans la console Utilisateurs et ordinateurs AD, nous avons créé l’arborescence suivante :

ITWAY  
├── Utilisateurs  
├── Groupes  
├── Machines  
├── Services  
    ├── IT  
    ├── COM  
    ├── DIR  
    ├── PROD

### 2.2 Création des groupes de sécurité

L’objectif est de permettre une gestion des droits par service (GPO, partages, accès, etc.)

Dans l’OU **Groupes** :

Clic droit \> Nouveau \> Groupe  
Nom : `GRP-SECU-IT`  
Type : **Sécurité**  
Étendue : **Global**

À reproduire pour les autres groupes

### 2.3 Profils itinérants

L’objectif est de permettre aux utilisateurs de retrouver leur environnement (bureau, documents, etc…) sur n’importe quel poste du domaine.

#### 2.3.1 Création du dossier partagé sur SRV-ADS01

- Dossier : `D:\Profils`  
- Clic droit \> Propriétés \> Partage \> Partage avancé \> Partager \> Nom : `Profils$` (le `$` cache le partage)  
- Autorisations : `Everyone` \> Lecture/Écriture (sera ensuite sécurisé par NTFS)

#### 2.3.2 Création des répertoires utilisateurs :

- Pour chaque utilisateur : créer un dossier `D:\Profils\%username%`  
- Attribuer les droits NTFS : `%username%` \= Contrôle total

#### 2.3.3 Affectation du profil itinérant 

Dans "Utilisateurs et ordinateurs AD" :

- Ouvrir la fiche utilisateur  
- Onglet Profil : Chemin du profil : `\\SRV-ADS01\Profils$\%username%`

#### 2.3.4 Test : 

\- Créer un dossier ou fichier sur le bureau puis se déconnecter

\- Se connecter avec le même utilisateur sur un autre PC du domaine

\- Vérifier que ce fichier apparaît

\- Vérifier que l’utilisateur ne voit pas les autres dossiers

### 2.4 Message d’accueil lors de la connexion

L’objectif est d’afficher un avertissement légal à la connexion d’un utilisateur.

- Créer une GPO : `GPO_message_connexion`

- Éditer :

`Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Options de sécurité`

- **Message texte d’avertissement** : "Les accès à ce poste de travail sont réservés aux utilisateurs autorisés. Toute tentative d’accès non autorisée sera sanctionnée."  
- **Titre** : "Avertissement de sécurité"

Le bannière contenant le message doit s’afficher avant toute tentative de connexion.

### 2.5 Audit des connexions / déconnexions, ainsi que des modifications GPO

L’objectif est de surveiller l'activité des connexions et déconnexions.

Créer une GPO `Audit_AD` :

`Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Stratégie d’audit`

- **Audit des événements d'ouverture de session** : Activé (Succès \+ Échec)  
- **Audit des modifications de stratégie** : Activé

Pour tester, il suffit de se rendre dans les journaux windows \> Sécurité : 

→ constater les Logon et Logoff pour les connexions/déconnexions 

→constater les ID de modification

### 

### 2.5 Chiffrement élevé pour le protocole RDP

L’Objectif est d’assurer la sécurité des sessions RDP.

 **Étapes :**

`Configuration ordinateur > Modèles d’administration > Composants Windows > Services Bureau à distance > Hôte de session Bureau à distance > Sécurité`

- **Chiffrement des connexions client** : Activé, Niveau : Haut


### 2.6 Désactiver le compte invité sur tous les postes

L’objectif est d’éviter l’usage du compte Guest car non sécurisé.

**Étapes :**

`Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Options de sécurité`

→**Comptes : statut du compte invité** : Désactivé

Pour vérifier, depuis un poste client ouvrir `lusrmgr.msc` et vérifier que le compte invité possède une icône avec une flèche vers le bas.

Cette GPO est à appliquer sur l’UO Ordinateurs.

### 2.7 Implémenter IPSec avec authentification Kerberos

L’objectif est de sécuriser les communications entre postes (chiffrement et authentification).

**Étapes :**

- Ouvrir `secpol.msc` \> Stratégies IPSec  
- Créer une stratégie : `IPSec-Kerberos`  
  `→` Mode de sécurité : Kerberos

→ Filtrer les communications sensibles (ex. entre contrôleurs de domaine)

Le déploiement se fait via GPO pour toutes les machines mais hélas nous n’avons pas su trouver le bon filtrage ni réussi à trouver la manière de tester cette GPO qui est donc existante sans que nous sachions si elle fonctionne ou non.

### 2.8. Entrées DNS statiques pour chaque machine

L’objectif est d’assurer une résolution fiable des noms internes.

**Étapes :**

Dans la console DNS (`dnsmgmt.msc`) :

- Aller dans la zone `itway.local`  
- Ajouter une **nouvelle entrée de type A** pour chaque machine :  
  →Nom : `SRV-X`  
  `→`IP : `172.16.X.X`

### 2.9 Création des zones DNS inverses

Cela permet la résolution inverse (IP → nom), cette fonctionnalité est notamment utile pour logs, sécurité, audits.

**Étapes :**

- Console DNS \> Zone de recherche inversée \> Nouvelle zone  
- Exemple : réseau `192.168.10.0/24`  
  `→` Zone : `10.168.192.in-addr.arpa`  
- Ajouter les enregistrements PTR correspondant aux entrées A

## 3\. Configuration complète du serveur PKI sur l’AD : 

Voici l’enregistrement DNS et l’ensemble de la configuration du serveur PKI sur l’active Directory : 

### 3.1 Création de l'enregistrement DNS pour SRV-PKI

Depuis le serveur DNS de l’AD (172.16.50.2) :

- Ouvrir le **Gestionnaire DNS** (`dnsmgmt.msc`)  
- Développer **Zones de recherche directes \> itway.local**  
- Clic droit sur itway.local \> **Nouvel hôte (A ou AAAA)...**  
- Remplir les champs suivants :

- Nom : srv-pki

- Adresse IP : 172.16.50.3

- Cocher la case **"Créer un enregistrement de pointeur associé"**

- Cliquer sur **Ajouter un hôte**  
- Vérification de la résolution DNS pour le nom d'hôte srv-pki.itway.local. 

`nslookup srv-pki.itway.local`

*Résultat attendu : 172.16.50.3 (Cela permet de s'assurer que le système peut **trouver** l'adresse IP associée à ce nom via le serveur DNS configuré)*

### 3.2 Export du certificat racine depuis le serveur PKI 

**Depuis le serveur PKI** :

1. Vérifier les chemins dans le fichier de configuration qui se trouve dans notre fichier de config ansible suivant :

`# Configuration des chemins`  
`pki_ca_dir: "/etc/pki"`  
`pki_ca_key_name: "ca.key"`  
`pki_ca_cert_name: "ca.crt"`  
`pki_openssl_conf: "/etc/ssl/openssl.cnf"`

Donc le chemin complet du ca.crt est  :  
`/etc/pki/certs/ca.crt`

2. Convertir le certificat en format compatible Windows :

Depuis le serveur PKI (Debian) :  
`openssl x509 -in /etc/pki/certs/ca.crt -out  /etc/pki/certs/ca.cer -outform DER`

*Cette commande convertit un certificat depuis un format .crt en PEM vers un .cer en DER compatible avec Windows.*

### 3.3 Transfert du certificat vers le contrôleur de domaine

1. Créer un répertoire de réception sur le contrôleur de domaine pour y stocker notre certificat :

`mkdir C:\Certs`

2. Vérifier que la disponibilité du service SSH :

Nous avons besoin du service ssh sur windows pour effectuer notre transfert de certificat depuis le serveur PKI vers notre contrôleur de domaine. Il faut donc vérifier d’abord s’il est bien installé. Ouvrir PowerShell en administrateur sur le serveur Windows et vérifier s’il est en cours d’exécution :   
`Get-Service sshd`

Si le statut est "Running", alors tout est ok. Sinon, il faut le démarrer :  
`Start-Service sshd`

Et s’il apparaît impossible de trouver un service assorti du nom “sshd”, il faut l’installer.  
`Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'`

Vérifier s’il le service est installés côté client et serveur, sinon installer celui qui est manquant, dans notre cas il nous manquait le serveur que nous avons téléchargé et installé :   
`Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0`

Puis vérifier l’installation :   
`Get-Service sshd`

Si tout s’est bien passé, le service sera listé, probablement avec le statut “Stopped”, alors il faudra le démarrer :   
`Start-Service sshd`

3. Transférer le certificat :

Maintenant que tout est prêt, il faut copier le certificat ca.cer sur le contrôleur de domaine :   
`scp /etc/pki/certs/ca.cer Administrator@172.16.50.2:C:\Certs`

*La commande scp (Secure Copy) est utilisée pour transférer des fichiers de manière sécurisée entre des systèmes (locaux et distants ou bien 2 systèmes distants).*

### 3.4 Déploiement du certificat racine via GPO

Sur le contrôleur de domaine (172.16.50.2) :

1. Ouvrir la console de gestion des stratégies de groupe : gpmc.msc  
2. Créer une nouvelle GPO ou modifier une existante :  
   - Aller dans Forêt \> Domaines \> itway.local  
   - Clic droit sur Objet de stratégie de groupe \> Nouveau  
   - Nommer la GPO : Déploiement Certificat Racine  
3. Éditer la GPO :  
   - Aller dans Configuration ordinateur \> Stratégies \> Paramètres Windows \> Paramètres de sécurité \> Stratégies de clé publique \> Autorités de certification racines de confiance  
   - Clic droit \> Importer  
   - Sélectionner C:\\Certs\\ca.cer et valider  
4. Appliquer la GPO aux unités organisationnelles souhaitées :  
   - Dans la console de gestion des stratégies de groupe (gpmc.msc), naviguer jusqu'à la GPO Déploiement Certificat Racine.  
   - Clic droit sur la GPO \> Lier un objet de stratégie de groupe existant.  
   - Sélectionner l'unité organisationnelle (OU) contenant les ordinateurs concernés.  
   - Valider et fermer la console.  
5. Forcer la mise à jour des stratégies sur le contrôleur de domaine :  
   `gpupdate /force`  
6. Vérification de l’application de la GPO sur le domaine :  
   `gpresult /r`

### 3.5. Vérification sur un poste client

1. Aller dans le **Panneau de configuration \> Options Internet \> Contenu \> Certificats \> Autorités de certification** (ou certmgr.msc dans la barre de recherches)

2. Dans l’onglet Autorités de certification racines de confiance, vérifier la présence du certificat srv-pki.itway.local

Pour terminer : 

- Chaque GPO a été placée dans l’OU qui nous a semblé adaptée ou à la racine  
- Toutes les GPO ont été forcées avec la commande gpupdate /force

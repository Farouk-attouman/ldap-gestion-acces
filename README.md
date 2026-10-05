# LDAP, Gestion des Accès

Projet réalisé dans le cadre de la formation à La Plateforme, Marseille.

L'objectif est de mettre en place un serveur OpenLDAP sur une machine Debian pour centraliser la gestion des utilisateurs et des accès dans une entreprise fictive, DataTrust.

## Contexte

DataTrust est une petite entreprise informatique qui souhaite organiser les accès de ses employés. Au lieu de créer des comptes sur chaque machine individuellement, on met en place un annuaire LDAP centralisé. Un seul endroit pour gérer tous les utilisateurs, tous les mots de passe, tous les groupes.

LDAP (Lightweight Directory Access Protocol) est le protocole qui permet aux applications de communiquer avec cet annuaire. Concrètement, quand un utilisateur se connecte à un service (FTP, SSH, etc.), ce service interroge l'annuaire LDAP pour vérifier les identifiants.

## Architecture

L'infrastructure repose sur trois machines virtuelles Debian, toutes dans le même réseau 172.16.0.0/16 :

| Machine | IP | Rôle |
|---------|------|------|
| VM DHCP/DNS | 172.16.0.10 | Serveur DHCP et DNS (projet précédent) |
| VM FTP/SSH | 172.16.0.20 | Serveur FTP et SFTP sur port 6500 (projet précédent) |
| VM LDAP | 172.16.0.21 | Serveur OpenLDAP, annuaire `faroukcorp.local` |

La VM LDAP a été ajoutée au réseau existant du projet DNS/DHCP/FTP. C'est elle qui héberge l'annuaire et qui répond aux requêtes d'authentification des autres machines.

## Job 1 : Groupes d'utilisateurs

Trois groupes ont été définis pour refléter l'organisation de DataTrust :

**basicusers** (gidNumber 10000) : les utilisateurs standards. Ils peuvent accéder aux fichiers partagés mais n'ont aucun droit d'administration. Cas d'usage : un employé du service comptabilité qui consulte des documents.

**developers** (gidNumber 10001) : les développeurs. Ils ont accès aux outils de développement et aux fichiers partagés, mais pas aux paramètres de sécurité. Cas d'usage : un développeur qui accède au dépôt de code et aux serveurs de test.

**admins** (gidNumber 10002) : les administrateurs. Ils ont tous les droits, y compris la gestion des comptes et la modification des paramètres de sécurité. Cas d'usage : le responsable informatique qui gère l'infrastructure.

## Job 2 : Mise en place technique

### Installation du serveur LDAP

Le serveur OpenLDAP a été installé sur la VM Debian (172.16.0.21). Les paquets installés sont `slapd` (le démon serveur) et `ldap-utils` (les outils en ligne de commande).

La configuration a été faite avec `dpkg-reconfigure slapd` en renseignant le domaine `faroukcorp.local`, ce qui donne le base DN `dc=faroukcorp,dc=local`.

### Structure de l'annuaire

L'annuaire est organisé en deux unités organisationnelles (OU) :

```
dc=faroukcorp,dc=local
├── ou=users        (les comptes utilisateurs)
├── ou=groups       (les groupes)
└── ou=policies     (les politiques de sécurité)
```

### Création des utilisateurs

Trois utilisateurs ont été créés, chacun dans un groupe différent :

| Utilisateur | UID | Groupe | Rôle |
|------------|-----|--------|------|
| Alice | 20000 | basicusers | Utilisateur basique |
| Bob | 20001 | developers | Développeur |
| Claire | 20002 | admins | Administratrice |

Les utilisateurs ont été ajoutés via des fichiers LDIF injectés avec `ldapadd`. Les mots de passe sont hashés en SSHA (Salted SHA), ce qui ajoute un sel aléatoire pour se protéger des attaques par table arc-en-ciel.

### Test des accès

Chaque utilisateur a été testé avec `ldapsearch` en se connectant avec son propre DN et son mot de passe (option -D et -W). Le résultat `0 Success` confirme que l'authentification fonctionne pour les trois comptes.

## Job 3 : Options de sécurité de base

### Gestion des mots de passe

Par défaut, OpenLDAP accepte n'importe quel mot de passe, même un seul caractère. On a donc mis en place une politique de mot de passe avec le module ppolicy.

La mise en place se fait en trois étapes :
1. Chargement du module ppolicy dans OpenLDAP
2. Activation de l'overlay ppolicy sur la base de données
3. Création de la politique elle-même avec `pwdMinLength: 8` et `pwdCheckQuality: 1`

Le test a été fait sur les trois utilisateurs. Quand on essaie de mettre un mot de passe de moins de 8 caractères (par exemple "abc" ou "12345"), le serveur retourne `Constraint violation (19)` avec le message `Password fails quality checking policy`. Quand on met un mot de passe de 8 caractères ou plus, ça passe.

### Déconnexion automatique

LDAP ne gère pas les sessions utilisateurs, seulement l'authentification. La déconnexion automatique est donc configurée à deux niveaux :

Côté serveur LDAP : `olcIdleTimeout: 180` dans la configuration cn=config. Toute connexion LDAP inactive pendant plus de 3 minutes est fermée automatiquement.

Côté système Linux : la variable `TMOUT=180` dans `/etc/profile` ferme les sessions shell après 3 minutes d'inactivité. Le `readonly` empêche les utilisateurs de désactiver cette protection.

## Intégration avec le projet DNS/DHCP/FTP

C'est la partie "aller plus loin" du projet. L'objectif est de connecter le serveur FTP/SSH (172.16.0.20) à l'annuaire LDAP pour que les utilisateurs se connectent en SFTP avec leurs identifiants LDAP.

### Principe

Au lieu d'avoir des comptes locaux sur la machine FTP, on installe des paquets qui permettent au système Linux de parler avec le serveur LDAP :

- `libnss-ldapd` : permet au système de résoudre les noms d'utilisateurs et groupes via LDAP
- `libpam-ldapd` : permet l'authentification des mots de passe via LDAP
- `nslcd` : le démon qui fait le lien entre Linux et LDAP
- `nscd` : cache local pour éviter d'interroger LDAP à chaque commande

### Configuration

Le fichier `/etc/nslcd.conf` pointe vers le serveur LDAP (172.16.0.21) et indique où chercher les utilisateurs, groupes et mots de passe dans l'annuaire.

Le fichier `/etc/nsswitch.conf` a été modifié pour ajouter `ldap` comme source d'information après `files`, ce qui veut dire : cherche d'abord en local, puis demande à LDAP.

La configuration SSH (`/etc/ssh/sshd_config`) a été adaptée pour autoriser les utilisateurs LDAP et les enfermer dans un chroot SFTP avec `ChrootDirectory /home/%u`.

### Preuves que l'authentification passe par LDAP

La commande `grep alice /etc/passwd` ne retourne rien (Alice n'existe pas en local), tandis que `getent passwd alice` retourne ses infos (le système la trouve via LDAP).

Les logs nslcd montrent les requêtes LDAP envoyées lors de chaque connexion SFTP, avec la recherche dans `ou=users,dc=faroukcorp,dc=local` et la réponse du serveur.

Test ultime : quand on arrête le serveur LDAP (`systemctl stop slapd`), les connexions SFTP échouent immédiatement. Quand on le relance, elles fonctionnent à nouveau.

## Difficultés rencontrées

Le format LDIF est très strict. Un espace en trop ou une ligne vide au mauvais endroit et la commande échoue avec un message d'erreur peu explicite. Le réflexe `cat -A` pour afficher les caractères invisibles a été indispensable.

L'intégration ProFTPd/LDAP a d'abord été tentée mais s'est avérée complexe car PAM interceptait l'authentification avant LDAP. La solution finale avec SSH/SFTP et les paquets libnss-ldapd/libpam-ldapd est plus propre et plus complète car elle intègre LDAP au niveau du système entier, pas juste d'un service.

Le chroot SSH impose que le répertoire home appartienne à root avec les permissions 755. Sans ça, SSH coupe la connexion sans message d'erreur clair. Un sous-dossier writable à l'intérieur du home permet à l'utilisateur de déposer ses fichiers.

## Technologies utilisées

- Debian (VM)
- OpenLDAP (slapd + ldap-utils)
- SSH/SFTP (OpenSSH)
- NSS/PAM (libnss-ldapd, libpam-ldapd, nslcd)
- VMware (virtualisation, réseau NAT 172.16.0.0/16)

## Auteur

Projet réalisé par Farouk Attouman dans le cadre de la formation à La Plateforme, Marseille.

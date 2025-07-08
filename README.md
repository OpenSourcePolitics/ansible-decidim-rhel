# Playbook Ansible pour Decidim sur RHEL 8

## Variables d'inventaire ansible
*Se référer au fichier **inventory-example.yml** pour les exemples*

- **ansible_host** : Adresse IP du serveur à provisionner
- **git_organization** : Le nom de l'organisation Git (exemple : `OpenSourcePolitics`)
- **git_root** : URL racine du serveur Git (exemple : https://github.com)
- **git_repository** : Nom du dépot Decidim hébergé sur Git (exemple : `decidim-app`)
- **repository_branch** : Branche git du dépot à déployer (par défaut : `master`)
- **decidim_host** : sous-domaine d'accès à la plateforme Decidim (exemple : `decidim-rhel8.osp.dev`)
- **database_host** : Adresse IP du serveur hébergeant la base de données (vide par défaut pour un accès local)
- **database_port** : Port permettant d'accéder à la base de données (`5432` par défaut)
- **database_name** : Nom de la base de donnée Decidim (par défaut : `osp_app`)
- **database_username** : Utilisateur ayant les acces à la base de donnée (vide par défaut pour un accès local avec l'utilisateur courant)
- **database_password** : Mot de passe de l'utilisateur avec acces à la base de donnée (vide par défaut)
- **secret_key_base:** : La secret key base utilisée avec l'archive de données PostgreSQL importée
- **decidim_deployment_path** : Chemin d'installation pour l'environnement Ruby on Rails et la stack Decidim
- **decidim_home_path** : Chemin du HOME du user
- **ssl_certificate_path** : Chemin vers le certificat SSL pour la configuration Nginx
- **ssl_certificate_key_path** : Chemin vers la clé du certificat SSL pour la configuration Nginx
- **decidim_admin_email** : addresse email d'un administrateur de la plateforme, **il doit exister sur chacune des organisations existantes** 

## Usage

```
ansible-playbook -u sudoer --private-key="~/path/to/ssh-key" -i inventory.yml playbook.yml
```
où : 
- _sudoer_ est l'utilisateur distant qui executera le contenu du playbook
- _~/path/to/ssh-key_ est le chemin local de la clé SSH autorisée sur le serveur distant
- _inventory.yml_ est vore fichier local d'inventaire
- _playbook.yml_ est le playbook à exécuter

## Playbook disponibles
- playbook.yml : installation complête de Decidim sur un serveur RHEL8 
- playbook-update-0.26.yml : mise à jour majeure d'un decidim existant (< 0.26)
- playbook-bump-0.26.yml : mise à jour mineure d'un decidim existant (~ 0.26.x)
- playbook-update-0.27.yml : mise à jour majeure d'un decidim existant (~ 0.26)
- playbook-bump-0.27.yml : mise à jour mineure d'un decidim existant (~ 0.27.x)


## Installation de Decidim avec import de données existante

### Pré-requis
- Avoir a disposition un serveur PostgreSQL avec :
    - Un utilisateur PostgreSQL avec password
    - Une base de donnée vide et donner les droits d'acces à l'utilisateur précedemment créé
    - Restaurer un export de la base de donnée sur la nouvelle  
      `pg_restore -d "<database-name>" --no-owner --role=<database-user> -c <dump-file>`
- Sur le serveur à provisionner (Decidim) :
    - Créer un utilisateur dédié à la gestion de la stack Decidim
    - Ajouter cet utilisateur au groupe sudoers
- Sur votre serveur Ansible
    - Cloner le dépot contenant le role Ansible pour le provisionnement de Decidim
    - Ajouter le serveur à provisionner dans votre inventaire Ansible (**inventory.yml**) en remplissant les variables suivantes avant d'exécuter le playbook
        ```
        decidim-server:
            ansible_host:
            git_organization:
            git_root:
            git_repository:
            repository_branch:
            decidim_host:
            certbot_email_address:
            database_host:
            database_port:
            database_name:
            database_username:
            database_password:
            secret_key_base:
            decidim_deployment_path: 
            decidim_home_path:
            ssl_certificate_path: 
            ssl_certificate_key_path: 
            decidim_admin_email: 
        ```

### Lancer le playbook d'installation

Une fois les pré-requis en place, lancer la commande suivante depuis de le répertoire Ansible pour lancer le déploiement du role Ansible sur le serveur à provisionner
```
ansible-playbook -u decidim --private-key="~/path/to/ssh-key" -i inventory.yml playbook.yml
```

### Résultat attendu

Apres exécution du playbook sur le serveur cible, celui-ci devrait avoir :

- Installer les dépendances liées à Ruby on Rails et Decidim
- Désactiver SELinux
- Installer le service Memcached
- Installer Rbenv, gestionnaire de versions Ruby
- Installer le dépot de code de Ruby
- Installer la derniere version stable de NodeJS
- Installer la derniere version stable de Yarn
- Compléter l'ensemble des étapes liées au bon fonctionnement de Decidim, à savoir : 
    - Téléchargement du dépot de code dans le répertoire créé à partir de la variable d'inventaire **git_repository**
    - Installation de la version de Ruby correspondant à celle spécifiée par le dépot de code de la decidim-app
    - Installation de la version de Bundler correspondant à celle spécifiée par le dépot de code de la decidim-app
    - Installation des gems liées à Decidim (présentes dans le fichier **Gemfile**)
    - Création d'un fichier **.env** contenant les variables d'environnement nécéssaires au bon fonctionnement de l'application et renseignées en amont dans le fichier **inventory.yml**
    - Installation des migrations Rails
    - Précompilation des assets statiques de l'application (CSS et JS)
- Configurer un logrotate pour les logs génerés par Decidim
- Installer Nginx et configurer le sous-domaine d'accès pour Decidim (variable d'inventaire **decidim_host**)
- Installer et configurer Phusion Passenger pour l'application Decidim
- Configurer Sidekiq pour le traitement des taches asynchrones executées par l'application Decidim
- Installer Redis
- Installer Certbot avec création d'un certificat SSL Let's Encrypt pour le sous-domaine d'accès pour Decidim (variable d'inventaire **decidim_host**)

### Finalisation

Extraire l'archive des fichiers uploadés dans le dossier de l'application Decidim qui correspond à 
```
/home/{{ ansible_user }}/{{ git_repository }}/public/uploads
```

exemple : 
```
tar -jxvf /path/to/archive /home/decidim/decidim-app/public/uploads
```

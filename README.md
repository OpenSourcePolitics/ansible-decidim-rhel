# Playbook Ansible pour Decidim sur RHEL / CentOS / Rocky linux en version 9

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
- **sidekiq_service** : nom du service sidekiq qui sera déployé

## Usage

```
ansible-playbook -u sudoer --private-key="~/path/to/ssh-key" -i inventory.yml playbook-xxx.yml
```
où : 
- _sudoer_ est l'utilisateur distant qui executera le contenu du playbook
- _~/path/to/ssh-key_ est le chemin local de la clé SSH autorisée sur le serveur distant
- _inventory.yml_ est vore fichier local d'inventaire
- _playbook-xxx.yml_ est le playbook à exécuter

## 📚 Playbook disponibles
### 📒 playbook-v9-install-commons.yml : installation des dépendances système globales
Ce playbook est à utiliser pour configurer un nouveau serveur ou vérifier que toutes les dépendances sont présentes et à jour. 
- paquets système
- memcached
- rbenv
- ruby
- nodejs
- yarn
- nginx
- passenger
- redis

### 📒 playbook-v9-certbot.yml : création d'un certificat SSL Let's Encrypt
Ce playbook permet la création d'un certificat SSL Let's Encrypt rattaché au nom de domaine sur lequel sera déployé la plateforme. 
Cette étape est optionnelle si vous voulez fournir votre propre certificat SSL. 

Le chemin du certificat doit être ensuite renseigné dans les variables d'inventaire `ssl_certificate_path` et `ssl_certificate_key_path`.
Dans le cas le Let's Encrypt / Certbot, les fichiers de certificat se trouvent par défaut à l'emplacement suivant : 
```
ssl_certificate_path: "/etc/letsencrypt/live/<votre-nom-de-domaine>/fullchain.pem"
ssl_certificate_key_path: "/etc/letsencrypt/live/<votre-nom-de-domaine>/privkey.pem"
```

### 📒 playbook-v9-install-local-postgres.yml : installation et configuration d'un serveur PostgreSQL en local sur le serveur
Ce playbook installe un serveur de base de données PostgrSQL en local sur le serveur. 
Une base de données vide est aussi créée avec les permissions accordées à l'utilisateur courant (`ansible_user`).  

Cette étape est optionnelle si vous voulez utiliser une base de données externe.
Dans le cas d'un import de données d'une plateforme Decidim existante, cette étape devra être faite manuellement **avant** l'installation de Decidim. 
Il faudra alors remplir les variables d'inventaire :  
- `database_host`
- `database_port`
- `database_name`
- `database_username`
- `database_password`

### 📒 playbook-v9-install-decidim.yml : installation de Decidim
Ce playbook installe Decidim sur votre serveur :  
- récupération du code
- installation de la bonne version de ruby si nécessaire
- récupération des dépendances logicielles 
- configuration du [fichier de variables d'environnement](v9_install_decidim/templates/env.j2)
- lancement des migrations de données si nécessaire (y compris pour une nouvelle installation)
- précompilation des assets JS et CSS

### 📒 playbook-v9-update-decidim-0.29.yml : Mise à jour d'une version de Decidim avant la 0.29
Ce playbook est a utiliser dans le cas d'une montée de version sur un Decidim antérieur à la version 0.29

### 📒 playbook-v9-bump-decidim-0.29.yml : Mise à jour classique d'un Decidim en version 0.29
Ce playbook est a utiliser dans le cas d'une montée de version classique sur Decidim en version 0.29

## Import de données existantes
Dans le cas où vous importer des données d'une plateforme Decidim existante il faudra effectuer les opérations manuelles suivantes :  
- S'assurer que vous disposez bien des accès administrateur de la plateforme pour l'interface d'admnistration (`/admin`) et système (`/system`)
- Récupérer l'ancien fichier .env de votre ancienne instalation de Decidim pour le copier sur votre serveur dans le dossier d'installation (`<decidim_deployment_path>/<app_repository>`) et notamment la variable d'environnement `SECRET_KEY_BASE`
- Restaurer un export de la base de donnée sur la nouvelle base qui sera utilisée
  `pg_restore -d "<database-name>" --no-owner --role=<database-user> -c <dump-file>`
- Transférer les fichiers du dossier `/storage` de votre ancienne instalation de Decidim pour les copier sur votre serveur dans le dossier d'installation (`<decidim_deployment_path>/<app_repository>`)

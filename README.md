
# *Docker-Ansible-Gitlab-Challenge-Infra (Léo Meneau)*


## Déroulé

#### ***1. Intitulé du challenge***  
#### ***2. Informations complémentaires***  
#### ***3. Étapes du challenge***  
#### ***4. Conclusion***
#### ***5. Dictionnaire***
#### ***6. Sources***

# ***1. Intitulé du challenge***

#### *Le but de ce challenge est de mettre en place, via Docker, une infrastructure composée de deux conteneurset ensuite effectuer un déploiement d'un serveur Nginx :*

- VM debian12 sous VMware, un conteneur SSH basé sur Debian 12, agissant comme un serveur distant. Il dispose d’un service SSH actif ainsi que d’un utilisateur avec moins de privilèges (leo) configuré avec une authentification par clé SSH.

- Un conteneur Ansible basé sur AlmaLinux 9, qui joue le rôle de machine de contrôle. Grâce au conteneur Ansible nous allons déployer automatiquement un serveur Nginx sur le conteneur SSH, à l’aide d’un playbook Ansible, le tout en utilisant une connexion SSH entre les deux conteneurs.

# ***2. Informations complémentaires***

- Conteneur ssh : sshserv_leo ; 172.17.0.2 connexion ssh hors mot de passe root et non-root ; port 22.

- Conteneur Ansible sous AlmaLinux9 : ansible_leo_container ; port Nginx 80.

- Création de l'utilisateur leo (non-root) pour le sshserv_leo






# ***3. Étapes du challenge*** 
## Create a SSH server Docker image base on Debian-12 image

######  VM Debian 12 sous VMware


### Schéma de la première partie

``` 
┌──────────────┐          ┌──────────────┐
|  (docker)    |  build   |  (docker)    |
|  Debian      |  ====>   |  ssh_server  |
└──────────────┘          └──────────────┘
```
###### Maj VM, installation docker, vérification de la version, création des répertoires, fichiers, création conteneur et connexion SSH
```bash
su - 
apt update
apt upgrade
apt install -y docker.io docker-compose
systemctl enable --now docker
usermod -aG docker $USER
apt install -y docker-compose
systemctl start docker
systemctl enable docker
docker --version
```
#### Création du répertoire et du Dockerfile pour le conteneur SSH ainsi que du contenu du Dockerfile 
```bash
mkdir /root/sshserv_leo
cd /root/sshserv_leo
touch Dockerfile
nano Dockerfile

FROM debian:12

RUN apt-get update 
    apt-get install -y openssh-server sudo python3 
    useradd -m -s /bin/bash leo 
    mkdir /home/leo/.ssh 
    chown -R leo:leo /home/leo/.ssh 
    chmod 700 /home/leo/.ssh

COPY authorized_keys /home/leo/.ssh/authorized_keys
RUN chown leo:leo /home/leo/.ssh/authorized_keys 
    chmod 600 /home/leo/.ssh/authorized_keys

RUN mkdir /var/run/sshd
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```
```bash
cd /root/sshserv_leo
docker build -t sshserv_leo .
```

#### Clé SSH pour l'utilisateur leo + copie de la clé publique dans le répertoire sshserv_leo
```bash
ssh-keygen -t ed25519 -C "leo@docker"
mkdir -p ~/sshserv_leo
cat ~/.ssh/id_ed25519.pub > ~/sshserv_leo/authorized_keys
ls -l ~/sshserv_leo/authorized_keys
```
#### Ensuite on se connecte au conteneur en ssh
```bash
docker run -d --name sshserv_leo -p 2222:22 sshserv_leo
```
#### Connexion avec le compte leo
```bash
ssh leo@localhost -p 2222
```
#### Et via l'adresse ip de la machine 
```bash
ssh leo@192.168.23.135 -p 2222
```
#### La commande hostname identifie le nom de la machine utilisée et whoami affiche l'utilisateur connecté.

```bash
hostname
........
whoami
```

![whoami](https://github.com/user-attachments/assets/6a031caa-bcce-4184-b0ef-8223b96085f2)

#### Création et modification du fichier docker-compose.yml permettant la simplification au niveau lancement de conteneurs.
```bash
nano /root/sshserv_leo/docker-compose.yml

version: '3'
services:
  ssh_server:
    build: .
    container_name: sshserv_leo
    ports:
      - "2222:22"
    volumes:
      - ./authorized_keys:/home/leo/.ssh/authorized_keys
    restart: unless-stopped
```
#### Ensuite il suffit de faire cette commande pour executer
```bash 
docker-compose up -d
```
## 2. Create an Ansible Docker image based on AlmaLinux-9 image

```
┌──────────────┐          ┌────────────────────┐
|  (docker)    |  build   | (docker)           |
|  AlmaLinux   |  ====>   | ansible_controller |
└──────────────┘          └────────────────────┘
```
#### Création du conteneur ansible_leo avec AlmaLinux9 et vérification de la version Ansible
```bash 
mkdir ~/ansible_leo
cd ~/ansible_leo
nano Dockerfile

FROM almalinux:9

RUN yum install -y epel-release
RUN yum update -y 
    yum install -y ansible sshpass python3

```
#### Ajout du conteneur Ansible
```bash 
docker build -t ansible_leo .
docker run -it --rm --name ansible_leo_container --network host -v /root/sshserv_leo:/ansible ansible_leo /bin/bash

```

#### Vérification version Ansible
```bash 
ansible --version
```
#### Création du fichier inventory.ini pour Ansible Il faut créer le fichier inventory.ini dans le conteneur ansible_leo_container puis dans le répertoire ansible:
```bash
[ssh_servers]
localhost ansible_host=localhost ansible_port=2222 ansible_user=leo ansible_ssh_private_key_file=/root/.ssh/id_ed25519
```
  
#### Une fois que vous êtes dans le conteneur Ansible nous pouvons faire le test du ping. (Il faut vérifier si tout est bon dans les fichiers et conteneur sinon vous aurez un message d'erreur en rouge).

```bash

ansible -i /ansible/inventory.ini ssh_servers -m ping

```
#### Exemple message d'erreur
![Exemple message d'errreur](https://github.com/Leo-Meneau/Leo-Meneau-Docker-Ansible-Gitlab-Challenge-Infra/blob/main/Exemple%20message%20d'erreur.png)
#### Message confirmant le ping entre les deux conteneurs (ansible vers ssh).
![Ping ansible vers ssh](https://github.com/user-attachments/assets/aea8eec7-0b9e-4470-ae98-e890654d876a)


#### Arrêter et supprimer le conteneur existant si vous avez un problème de connexion au conteneur
```bash
docker stop ansible_leo_container
docker rm ansible_leo_container

#### Puis la commande d'en haut pour se connecter
```

## 3. Create an Ansible playbook to configure an Nginx web server
#### Configuration Ansible pour le déploiement du serveur nginx vers sshserv_leo grâce au playbook.yml
```
┌────────────── docker-compose ────────────────┐
|                                              |
|  ansible_controller                          |
|     ║                                        |
|     ║ Deploy Nginx server                    |
|     ║ command: ansible-playbook ...          |
|     V                                        |
|  ssh_server                                  |
|                                              |
└──────────────────────────────────────────────┘
``` 

#### Création des répertoires suivants pour le role nginx (mettre dans le conteneur ansible puis répertoire ansible)
```bash
mkdir -p /ansible/roles/nginx/tasks/main.yml
```
#### Il faut créer le fichier playbook.yml dans le conteneur ansible_leo_container puis dans le répertoire ansible.

```bash

nano /ansible/playbook.yml


- name: Installer Nginx avec un rôle
  hosts: sshserv_leo
  become: yes
  roles:
    - nginx


```
#### Modification du fichier main.yml
```bash
nano /ansible/roles/nginx/tasks/main.yml

- name: Install Nginx
  ansible.builtin.yum:
    name: nginx
    state: present

- name: Start and enable Nginx
  ansible.builtin.systemd:
    name: nginx
    state: started
    enabled: yes

```
#### Création et modification du fichier pour le port 80 (nginx).
```bash
nano /ansible/roles/nginx/defaults/main.yml

nginx_listen_port: 80

```
#### Cette commande permet le lancement du déploiement du serveur nginx via conteneur ansible vers sshserv_leo. Si erreur, vous aurez un message en rouge sinon ce sera en vert (voir capture d'écran).
```bash
ansible-playbook -i /ansible/inventory.ini /ansible/playbook.yml
```
![Déploiement Nginx via conteneur Ansible vers conteneur SSH](https://github.com/Leo-Meneau/Leo-Meneau-Docker-Ansible-Gitlab-Challenge-Infra/blob/main/Deploiement-Nginx.png)
#### En cas de problème de droit vous pouvez éditer le fichier visudo et mettre à la toute fin ceci pour le compte leo dans le sshserv_leo

```bash
sudo visudo
leo    ALL=(ALL:ALL) ALL
```
#### Après la réussite du déploiement du serveur nginx il faut vérifier s'il est bien présent dans le sshserv_leo avec cette commande !
```bash
sudo nginx -t
```
#### Puis faire cette commande pour lancer le serveur nginx

```bash
sudo nginx
```
#### Status Nginx, nous pouvons voir que nginx est opérationnel
![Status Nginx](https://github.com/user-attachments/assets/bd79eee9-bd05-478a-a3de-457b6033e316)


#### Commande pour vérifier si nginx est bien sur le port 80.
```bash
sudo ss -tulpn | grep :80
```
#### Ensuite il suffit de lancer le navigateur et de mettre ceci pour vérifier que nginx fonctionne correctement !

```bash
curl http://172.18.0.2:80
```
![Test page nginx](https://github.com/user-attachments/assets/897ebb9e-4a1f-4b4c-ab92-00e9267b4724)


# ***4. Conclusion***

Grâce à ce challenge j'ai pu mettre en place une situation dans un environnement docker, conteneur SSH et Ansible pour automatiser la configuration d’un environnement et déployer un serveur nginx via conteneur ansible vers conteneur ssh + test. Je pense avoir réussi les missions demandées, je suis fier de ce que j'ai pu faire car je connaissais seulement de nom Ansible. J'avais fait un exposé sur Ansible mais jamais utilisé auparavant. Je pense avoir atteint l'objectif, cela me motive davantage pour intégrer votre organisation en tant qu'alternant ingénieur intégrateur Système, Réseau & Sécurité. Cette expérience a été enrichissante pour moi.

# ***5. Dictionnaire*** 

- Docker : Plateforme qui permet de créer, déployer et exécuter des applications dans des conteneurs légers et isolés.

- Nginx : Serveur web open source performant, souvent utilisé comme proxy inverse ou pour héberger des sites web.

- Ansible : Outil de gestion de configuration et d'automatisation, basé sur SSH, qui permet de déployer et configurer des machines à distance.

- YAML : Format de fichier lisible par l’humain, souvent utilisé pour décrire des configurations comme les playbooks Ansible.

- AlmaLinux9 : Système d’exploitation Linux basé sur RHEL (Red Hat Enterprise Linux), compatible entreprise, souvent utilisé pour des serveurs en production.


| Commandes | Explication |
|----------|-------------|
| `docker build -t ssh_server .` | Construit une image Docker nommée `ssh_server` à partir du Dockerfile. |
| `docker run -d --name sshserv_leo -p 2222:22 ssh_server` | Lance un conteneur et redirige le port 22 vers 2222. |
| `ssh leo@localhost -p 2222` | Se connecte en SSH à `sshserv_leo` sur le port 2222 avec l’utilisateur `leo`. |
| `localhost ansible_host=localhost ansible_port=2222 ansible_user=leo ansible_ssh_private_key_file=/root/.ssh/id_ed25519` | Ligne d'inventaire Ansible pour se connecter au conteneur via clé privée. |
| `ansible -i /root/sshserv_leo/inventory.ini ssh_servers -m ping` | Vérifie la connexion SSH avec Ansible vers le groupe `ssh_servers`. |
| `docker-compose up -d` | Lance tous les services définis dans `docker-compose.yml` en arrière-plan. |
| `ansible -i /ansible/inventory.ini ssh_servers -m ping` | Ping tous les serveurs du groupe `ssh_servers` via Ansible. |
| `ansible-playbook -i /ansible/inventory.ini /ansible/playbook.yml` | Exécute un playbook pour configurer ou déployer sur les hôtes. |



# ***6. Sources***

[Déployer Nginx sur Docker avec Ansible - BogoToBogo](https://www.bogotobogo.com/DevOps/Ansible/Ansible-Deploy-Nginx-to-Docker.php)

 [Comment se connecter en SSH à un conteneur Docker deux méthodes](https://fr.linux-console.net/?p=19925)

 #### J'ai également utilisé les diverses ressources mises à ma disposition dans le document fournit (readme du mail).

#### Léo Meneau



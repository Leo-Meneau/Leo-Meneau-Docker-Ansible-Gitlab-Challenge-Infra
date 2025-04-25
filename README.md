
# *Docker-Ansible-Gitlab-Challenge-Infra (Léo Meneau)*


## Déroulé

Intitulé du challenge

Informations complémentaires

Étapes du challenge

Conclusion

Dictionnaire

Sources

# ***1. Intitulé du challenge***

#### *Le but de ce challenge est de mettre en place, via Docker, une infrastructure composée de deux conteneurset ensuite effectuer un déploiement d'un serveur Nginx :*

- VM debian12 sous VMware, un conteneur SSH basé sur Debian 12, agissant comme un serveur distant. Il dispose d’un service SSH actif ainsi que d’un utilisateur avec moins de privilèges (leo) configuré avec une authentification par clé SSH.

- Un conteneur Ansible basé sur AlmaLinux 9, qui joue le rôle de machine de contrôle. Grâce au conteneur Ansible nous allons déployer automatiquement un serveur Nginx sur le conteneur SSH, à l’aide d’un playbook Ansible, le tout en utilisant une connexion SSH entre les deux conteneurs.

# ***2. Informations complémentaires***

- Conteneur ssh : sshserv_leo ; 172.17.0.2 connexion ssh hors mot de passe root et non-root ; port 22 ; mdp : leopass.

- Conteneur Ansible sous AlmaLinux9 : ansible_leo ; 172.17.0.3 ; port Nginx 80.

- Création de l'utilisateur leo (non-root) pour le sshserv_leo

- Arborescence situation
```
/root/challenge-infra
├── ansible-leo
│   ├── ansible.cfg
│   ├── Dockerfile
│   ├── id_rsa
│   ├── id_rsa.pub
│   ├── inventory.ini
│   └── playbook.yml
├── docker-compose.yml
└── ssh-leo
    ├── Dockerfile
    ├── entrypoint.sh
    └── id_rsa.pub

```

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
apt install tree
apt install -y docker.io docker-compose
usermod -aG docker $USER
systemctl start docker
systemctl enable docker
docker --version
```
#### Création des répertoires ssh-leo et ansible-leo et des Clés SSH
```bash
mkdir -p challenge-infra/{ssh-leo,ansible-leo}
cd challenge-infra
ssh-keygen -t rsa -b 4096 -f ansible-leo/id_rsa -N ""
cp ansible-leo/id_rsa.pub ssh-leo/id_rsa.pub
chmod 600 ansible-leo/id_rsa
chmod 644 ssh-leo/id_rsa.pub
```
#### Configurations fichiers (Dockerfile ssh, Dockerfile ansible, playbook.yml, inventory.inventory.ini...)

```bash
nano /root/challenge-infra/ssh-leo/Dockerfile

```
```bash
FROM debian:12

RUN apt-get update && \
    apt-get install -y openssh-server sudo && \
    apt-get clean

RUN useradd -m -s /bin/bash leo && \
    echo 'leo:leopass' | chpasswd && \
    adduser leo sudo

RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

COPY entrypoint.sh /entrypoint.sh

RUN chmod +x /entrypoint.sh

EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]


```

```bash
nano /root/challenge-infra/ansible-leo/Dockerfile

```

```bash
FROM almalinux:9

RUN dnf install -y epel-release && \
    dnf install -y ansible openssh-clients iputils && \
    useradd -m ansible && \
    mkdir -p /home/ansible/.ssh

COPY id_rsa /home/ansible/.ssh/id_rsa
COPY ansible.cfg inventory.ini playbook.yml /home/ansible/

RUN chmod 600 /home/ansible/.ssh/id_rsa && chown -R ansible:ansible /home/ansible

USER ansible
WORKDIR /home/ansible

CMD ["tail", "-f", "/dev/null"]


```
```bash
nano /root/challenge-infra/ansible-leo/inventory.ini

```
```bash
[ssh_target]
sshserv_leo ansible_ssh_user=ansible ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
```

```bash
nano /root/challenge-infra/ansible-leo/playbook.yml
```

```bash
- name: Installer Nginx
  hosts: ssh_target
  become: yes
  tasks:
    - name: Mettre à jour les paquets
      apt:
        update_cache: yes
        upgrade: yes
      when: ansible_facts['distribution'] == "Ubuntu" or ansible_facts['distribution'] == "Debian"

    - name: Installer Nginx
      apt:
        name: nginx
        state: present
      when: ansible_facts['distribution'] == "Ubuntu" or ansible_facts['distribution'] == "Debian"

    - name: Démarrer et activer le service Nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
```bash
nano /root/challenge-infra/ansible-leo/ansible.cfg
```

```bash
#[defaults]
#inventory = ./inventory.ini
#host_key_checking = False


[defaults]
inventory = ./inventory.ini
host_key_checking = False
ask_sudo_pass = True

```

```bash
nano /root/challenge-infra/docker-compose.yml
```

```bash
version: '3.8'

services:
  ssh-leo:
    build: ./ssh-leo
    container_name: sshserv_leo
    hostname: ssh-leo
    ports:
      - "2222:22"

  ansible-leo:
    build: ./ansible-leo
    container_name: ansible_leo
    depends_on:
      - ssh-leo

```

```bash
nano /root/challenge-infra/ssh-leo/entrypoint.sh
```

```bash
#!/bin/bash
set -e
mkdir -p /run/sshd
exec /usr/sbin/sshd -D
```
```bash
chmod +x ssh-leo/entrypoint.sh
```

#### La commande hostname identifie le nom de la machine utilisée et whoami affiche l'utilisateur connecté.

```bash
hostname
........
whoami
```

![whoami](https://github.com/user-attachments/assets/6a031caa-bcce-4184-b0ef-8223b96085f2)


#### Ensuite il suffit de faire cette commande pour executer
```bash 
docker-compose up -d
docker ps 
docker-compose down si vous souhaitez l'arrêter.
```
docker images
#### Ensuite on peut se connecter au conteneur ssh en tant que leo
```bash
ssh leo@localhost -p 2222
```
## 2. Create an Ansible Docker image based on AlmaLinux-9 image

```
┌──────────────┐          ┌────────────────────┐
|  (docker)    |  build   | (docker)           |
|  AlmaLinux   |  ====>   | ansible_controller |
└──────────────┘          └────────────────────┘
```
#### Grâce au docker-compose up -d , cela nous permet de démarrer les conteneurs. Comme j'ai refait plusieurs fos le challenge je me suis permis d'optimiser ça.

#### Vérification version Ansible
```bash 
ansible --version
```
#### Il faut maintenant tester la connexion et le ping entre les deux conteneurs, j'ai eu quelques soucis de permission donc voici les commandes de modification de permission et déplacement de fichiers. Il y a quelques modifications à faire dans les conteneurs si cela ne fonctionne pas

```bash
ansible -i "sshserv_leo, ansible_host=ssh-leo ansible_user=leo ansible_ssh_private_key_file=/root/.ssh/id_rsa" -m ping
chmod 600 /root/.ssh/id_rsa
chmod 600 ~/challenge-infra/ansible-leo/id_rsa
docker exec -it ansible_leo bash
ls -l /root/.ssh/id_rsa
ssh -i /home/ansible/ansible-leo/id_rsa leo@ssh-leo
docker exec -it ansible_leo bash
docker cp ~/challenge-infra/ansible-leo/id_rsa ansible_leo:/home/ansible/ansible-leo/id_rsa
docker exec -it ansible_leo bash
docker cp ~/challenge-infra/ansible-leo/id_rsa ansible_leo:/home/ansible/.ssh/id_rsa
docker exec -it ansible_leo bash
mkdir -p /home/ansible/.ssh
docker exec -it ansible_leo bash
ansible -i inventory.ini ssh_target -m ping

```

#### Une fois que vous êtes dans le conteneur Ansible nous pouvons faire le test du ping. (Il faut vérifier si tout est bon dans les fichiers et conteneur sinon vous aurez un message d'erreur en rouge).


```bash
ansible -i /ansible/inventory.ini ssh_servers -m ping

```
![Exemple message d'errreur](https://github.com/Leo-Meneau/Leo-Meneau-Docker-Ansible-Gitlab-Challenge-Infra/blob/main/Exemple%20message%20d'erreur.png)
#### Message confirmant le ping entre les deux conteneurs (ansible vers ssh).
![Ping ansible vers ssh](https://github.com/user-attachments/assets/aea8eec7-0b9e-4470-ae98-e890654d876a)


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
## Ici c'est la partie Nginx mais dans ce second test je n'ai pas réussi à reproduire comme l'ancienne version que j'avais fait pourtant le déploiement avait fonctionné. Je laisse cette étape pour montrer que je l'ai quand même fait et réussi 1 fois.

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

Grâce à ce challenge j'ai pu mettre en place une situation dans un environnement docker, conteneur SSH et Ansible pour automatiser la configuration d’un environnement et déployer un serveur nginx via conteneur ansible vers conteneur ssh + test. Je pense avoir réussi les missions demandées, je suis fier de ce que j'ai pu faire car je connaissais seulement de nom Ansible. J'avais fait un exposé sur Ansible mais jamais utilisé auparavant. Je pense avoir atteint l'objectif, cela me motive davantage pour intégrer votre organisation en tant qu'alternant ingénieur intégrateur Système, Réseau & Sécurité. Cette expérience a été enrichissante pour moi. Avec quelques soucis niveau temps dû à mes trajets et travail (alternance, école), je n'ai pas pu tout exploiter et essayé de nouveau la partie nginx sur mon nouveau lab. Désormais je connais le principe de ce challenge, les fichiers de confnigurations, l'environnement docker car je l'ai refait plusieurs fois.

# ***5. Dictionnaire*** 

- Docker : Plateforme qui permet de créer, déployer et exécuter des applications dans des conteneurs légers et isolés.

- Nginx : Serveur web open source performant, souvent utilisé comme proxy inverse ou pour héberger des sites web.

- Ansible : Outil de gestion de configuration et d'automatisation, basé sur ssh, qui permet de déployer et configurer des machines à distance.

- YAML : Format de fichier lisible par l’humain, souvent utilisé pour décrire des configurations comme les playbooks Ansible.

- AlmaLinux9 : Système d’exploitation Linux basé sur RHEL (Red Hat Enterprise Linux), compatible entreprise, souvent utilisé pour des serveurs en production.

- docker-compose.yml : Fichier de configuration pour Docker Compose qui définit les services, réseaux et volumes nécessaires pour les conteneurs, incluant ssh-leo et ansible-leo.

- inventory.ini : Fichier d'inventaire Ansible qui contient la liste des hôtes à gérer, avec leurs paramètres de connexion ssh (utilisateur, clé privée, etc.).

- playbook.yml : Fichier YAML qui contient les instructions Ansible à exécuter sur les hôtes. Il définit des tâches comme l'installation de Nginx ou la gestion de services.


| Commandes | Explication |
|-----------|-------------|
| `mkdir -p challenge-infra/{ssh-leo,ansible-leo}` | Crée le répertoire `challenge-infra` avec les sous-répertoires `ssh-leo` et `ansible-leo`. |
| `ssh-keygen -t rsa -b 4096 -f ansible-leo/id_rsa -N ""` | Crée une clé ssh rsa de 4096 bits pour `ansible-leo`. |
| `cp ansible-leo/id_rsa.pub ssh-leo/id_rsa.pub` | Copie la clé publique ssh de `ansible-leo` vers `ssh-leo`. |
| `chmod 600 ansible-leo/id_rsa` | Définit les bonnes permissions pour la clé privée ssh de `ansible-leo`. |
| `docker-compose up -d` | Lance les conteneurs définis dans le fichier `docker-compose.yml` en mode détaché. |
| `docker ps` | Affiche la liste des conteneurs en cours d'exécution. |
| `docker exec -it ansible_leo bash` | Ouvre une session interactive Bash dans le conteneur `ansible_leo`. |
| `ansible -i "sshserv_leo, ansible_host=ssh-leo ansible_user=leo ansible_ssh_private_key_file=/root/.ssh/id_rsa" -m ping` | Vérifie la connexion sshavec Ansible vers `ssh-leo`. |
| `docker cp ~/challenge-infra/ansible-leo/id_rsa ansible_leo:/home/ansible/.ssh/id_rsa` | Copie la clé privée ssh dans le conteneur `ansible_leo` pour l'utilisateur `ansible`. |
| `docker exec -it --user root ansible_leo bash` | Ouvre une session Bash dans le conteneur `ansible_leo` en tant qu'utilisateur `root`. |
| `docker exec -it ssh-leo bash` | Ouvre une session interactive Bash dans le conteneur `ssh-leo`. |
| `ansible-playbook -i /ansible/inventory.ini /ansible/playbook.yml` | Exécute un playbook Ansible pour déployer ou configurer les hôtes définis dans l'inventaire. |




# ***6. Sources***

[Déployer Nginx sur Docker avec Ansible - BogoToBogo](https://www.bogotobogo.com/DevOps/Ansible/Ansible-Deploy-Nginx-to-Docker.php)

 [Comment se connecter en SSH à un conteneur Docker deux méthodes](https://fr.linux-console.net/?p=19925)

 #### J'ai également utilisé les diverses ressources mises à ma disposition dans le document fournit (readme du mail).

#### Léo Meneau



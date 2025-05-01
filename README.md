# Docker_Projet

# Définitions

### Dockerfile
Un **Dockerfile** est un fichier de configuration utilisé pour créer des images Docker. Il définit les étapes nécessaires à l'installation d'un environnement spécifique dans un conteneur, comme l'installation de paquets (par exemple, OpenSSH) et la configuration de services (comme SSH). Ce fichier permet d'automatiser la création et le déploiement d'un conteneur avec un environnement cohérent.

### Inventory
L'**inventory** dans Ansible est un fichier qui liste les hôtes (serveurs ou conteneurs) ciblés pour l'exécution des tâches. Il définit les informations nécessaires pour se connecter à ces hôtes (adresse IP, utilisateur, etc.), et permet à Ansible de savoir où exécuter les playbooks.

### Playbook
Un **playbook** est un fichier YAML dans lequel sont définies les actions à exécuter sur les hôtes listés dans l'inventory. Chaque playbook contient des "plays" (groupes de tâches) qui automatisent des configurations, comme l'installation ou la gestion de services, sur plusieurs machines à la fois.

### Fichiers `.yml`
Les fichiers `.yml` (YAML) sont utilisés pour la configuration dans Docker et Ansible. Dans **Docker Compose**, ils définissent les services, réseaux et volumes des conteneurs. Dans **Ansible**, ils servent à écrire des playbooks et des rôles pour automatiser les tâches sur les hôtes ciblés.

---

# Docker-Ansible Challenge: Infrastructure

## Objectif

L'objectif de cet exercice est de mettre en place un environnement de déploiement automatisé en utilisant **Docker** et **Ansible**. Ce processus inclut la création de conteneurs pour un serveur SSH et un conteneur pour exécuter Ansible, puis le déploiement de Nginx sur le serveur SSH.

L'exercice inclut :
1. **Création d'un serveur SSH** dans un conteneur Docker.
2. **Création d'un conteneur Ansible** pour exécuter des playbooks.
3. **Déploiement de Nginx** sur le serveur SSH via Ansible.

### Structure du projet

Le projet est divisé en trois parties principales :

1. **Création du serveur SSH Docker** :
   - Utilisation d'une image Debian-12 pour créer un serveur SSH dans un conteneur Docker.
   
2. **Création du conteneur Ansible Docker** :
   - Utilisation d'une image AlmaLinux-9 pour installer Ansible et exécuter des commandes ou des playbooks.
   
3. **Déploiement de Nginx avec Ansible** :
   - Création d'un playbook Ansible pour installer et configurer un serveur Nginx sur le serveur SSH.

---

## Étapes

### 1. Création du serveur SSH Docker

La première étape consiste à configurer un serveur SSH dans un conteneur Docker. Pour cela, nous utilisons une image de base `Debian-12`, installons OpenSSH et configurons un utilisateur non-root pour accéder au conteneur via SSH.

#### Dockerfile pour le serveur SSH

```dockerfile
FROM debian:12

RUN apt update && apt install -y openssh-server sudo

RUN useradd -m -d /home/test -s /bin/bash -g root -G sudo test

RUN echo 'test:test' | chpasswd

RUN mkdir /var/run/sshd

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

Explication des choix :
- **Debian-12** : Une image légère et maintenue, idéale pour personnaliser l'environnement.
- **Installation d'OpenSSH et de sudo** : Permet d'utiliser SSH pour l'administration et de créer un utilisateur ayant des privilèges sudo.
- **Création d'un utilisateur non-root** : Cela permet de sécuriser l'accès au conteneur en évitant d'utiliser le compte root.
- **Port 22 exposé** : Le serveur SSH utilise le port 22 pour accepter les connexions SSH.
- **Command démarrage SSH** : La commande `/usr/sbin/sshd -D` permet de garder le serveur SSH en fonctionnement.

### 2. Création du conteneur Ansible Docker

Le conteneur Ansible utilise une image de base **AlmaLinux-9**, sur laquelle nous installons **Ansible** ainsi que les dépendances nécessaires à son fonctionnement.

#### Dockerfile pour le conteneur Ansible

```dockerfile
FROM almalinux:9

RUN dnf update -y && \
    dnf install -y python3 python3-pip openssh-clients sudo glibc-langpack-en && \
    pip3 install --no-cache-dir ansible && \
    dnf clean all

ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

CMD ["/bin/bash"]
```

### 3. Configuration de l'infrastructure avec Docker Compose

Le fichier `docker-compose.yml` permet de lier les deux conteneurs, de gérer les réseaux et de faciliter la gestion des services.

#### Exemple de `docker-compose.yml`

```yaml
version: "3.9"

services:
  ssh_server:
    build:
      context: ./ssh-serveur
    container_name: ssh-serveur
    ports:
      - "2222:22"  # Redirection du port local 2222 vers le port 22 du conteneur SSH
    hostname: ssh-serveur
    networks:
      - my_network

  ansible-alma9:
    image: ansible-alma9
    container_name: ansible-alma9
    networks:
      - my_network
    stdin_open: true
    tty: true

networks:
  my_network:
    driver: bridge
```

- **Répartition des services** : Les services sont séparés pour une gestion plus claire, un conteneur pour le serveur SSH et un autre pour Ansible.
- **Réseau privé** : Utilisation d'un réseau Docker privé pour sécuriser la communication entre les conteneurs.

### 4. Playbook Ansible pour déployer Nginx

Une fois que le serveur SSH et le conteneur Ansible sont en place, un **playbook Ansible** est utilisé pour installer et configurer Nginx sur le serveur SSH.

#### Exemple de playbook Ansible (`nginx_playbook.yml`)

```yaml
---
- name: Install and start Nginx on web servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install required packages
      apt:
        name:
          - nginx
          - python3
          - python3-apt
        state: present
        update_cache: yes

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes
```

Le playbook :
- **Installe Nginx** sur le serveur.
- **Démarre le service Nginx** et le configure pour démarrer au démarrage du système.

---

## Sécurisation de l'infrastructure

### 1. Sécurisation de la redirection de ports

La redirection de ports est un point crucial en termes de sécurité. Dans ce projet, nous redirigeons le port **2222** du conteneur SSH vers le port **22** de l'hôte. Cette pratique présente plusieurs avantages et nécessite quelques précautions :

- **Port non standard** : En redirigeant le port 22 vers un autre port (2222), nous évitons de laisser le port SSH standard exposé directement à l'extérieur, réduisant ainsi les risques d'attaques automatisées ciblant ce port.
- **Accès contrôlé** : Le port **2222** est le seul point d'accès au conteneur SSH, ce qui limite les possibilités d'accès direct aux autres services.
- **Utilisation de clés SSH** (prévu dans une étape future) pour renforcer la sécurité, notamment en remplaçant les mots de passe par des clés publiques privées.


### 2. Sécurisation générale

En plus de la redirection de ports, voici d'autres mesures de sécurisation mises en place :

- **Utilisation d'un utilisateur non-root** pour la connexion SSH afin d'éviter les risques d'accès direct à l'utilisateur root.
- **Isolation du réseau Docker** : Nous utilisons un réseau Docker privé pour communiquer entre les conteneurs, évitant ainsi d'exposer les autres services ou ports inutiles.
- **Mises à jour de sécurité régulières** : Les images Docker sont régulièrement mises à jour pour intégrer les derniers correctifs de sécurité.

---

## Tests et Validation

Après avoir mis en place l'infrastructure Docker avec le serveur SSH et le conteneur Ansible, voici les étapes pour tester le bon fonctionnement :

1. **Vérifier la connexion SSH** : Utilise `ssh` pour te connecter au conteneur SSH en utilisant le port redirigé (ex. `ssh -p 2222 test@localhost`).
2. **Exécuter le playbook Ansible** : Lance le playbook pour installer Nginx et vérifie que le serveur est opérationnel avec la commande `curl` ou en accédant à `http://localhost:80`.

---

## Conclusion

Ce projet montre comment utiliser Docker et Ansible pour automatiser le déploiement d'une application web (Nginx) dans un environnement sécurisé et modulaire. Chaque composant (serveur SSH, conteneur Ansible, et playbook Nginx) a été conçu pour fonctionner de manière indépendante tout en garantissant la sécurité et la facilité de gestion.

---

**Liens utiles** :
- [Mettre en place un serveur SSH dans un conteneur Docker](https://dev.to/s1ntaxe770r/how-to-setup-ssh-within-a-docker-container-i5i)
- [Référence Docker Compose](https://docs.docker.com/reference/compose-file/build/)
- [Création d'un utilisateur non-root dans Docker](https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user#_creating-a-nonroot-user)
- [Introduction à Ansible](https://docs.ansible.com/ansible/latest/getting_started/index.html)
- [Intégration de Nginx avec Ansible](https://www.skynats.com/blog/how-to-integrate-nginx-with-ansible/)

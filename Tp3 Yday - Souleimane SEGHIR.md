                                    TP3 Devops – Ansible
Réalisé au préalable :
-	3 Vms (master, worker1, worker2) Ubuntu 20.04 avec connexion SSH depuis la machine hôte
-	Les Vms communiquent entre elles et accèdent à internet
-	Openssh et python 3 installé sur toutes les machines
-	Ansible installé sur une Vm hôte

Fichier de configuration Ansible:
```
[defaults]
remote_user         = soussou
inventory           = ./inventory.ini
private_key_file    = ~/.ssh/id_rsa
deprecation_warning = False
roles_path          = ./roles

host_key_checking   = False

[privilege_escalation]
become_method       = sudo
```

**Inventory**
```
Fichier inventory.ini
[masters]
master ansible_host=10.0.0.2
[workers]
worker1 ansible_host=10.0.0.3
worker2 ansible_host=10.0.0.4
```
On génère une clé ssh sur la machine hôte :
```
ssh-keygen -t rsa
```
Puis on fait une copie de la clé sur toutes les machines (master inclus)
```
ssh-copy-id soussou@10.0.0.2
ssh-copy-id soussou@10.0.0.3
ssh-copy-id soussou@10.0.0.4
```
**Role Common**
Exécuter le module ping avec la ligne de commande Ansible : 
```
soussou@controller:~/tp3# ansible all -u soussou -m ping
worker1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
worker2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
master | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
Puis dans le role common :
Mettre à jour les dépots de la machines avec le module apt.
Installer les paquets suivant permettant à apt d'être utilisés par de l'HTTPS :
> apt-transport-https
ca-certificates
curl
gnupg-agent
software-properties-common
```
soussou@controller:~/tp3# ansible-playbook roles/common/tasks/main.yml --ask-become-pass
BECOME password:

PLAY [all] ********************************************************************************************

TASK [Gathering Facts] ********************************************************************************
ok: [worker1]
ok: [worker2]
ok: [master]

TASK [Update package repository] **********************************************************************
ok: [master]
ok: [worker1]
ok: [worker2]

TASK [Upgrade packages] *******************************************************************************
ok: [master]
ok: [worker1]
ok: [worker2]

TASK [Install apt-transport-https] ********************************************************************
ok: [master]
ok: [worker1]
ok: [worker2]

TASK [Install ca-certificates] ************************************************************************
ok: [master]
ok: [worker2]
ok: [worker1]

TASK [Install curl] ***********************************************************************************
ok: [master]
ok: [worker2]
ok: [worker1]

TASK [Install gnupg-agent] ****************************************************************************
ok: [master]
ok: [worker2]
ok: [worker1]

TASK [Install software-properties-common] *************************************************************
ok: [master]
ok: [worker1]
ok: [worker2]

PLAY RECAP ********************************************************************************************
master                     : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker1                    : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker2                    : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Contenu du playbook :
```
- hosts: all
  become: yes
  tasks:
  - name: Update package repository
    apt:
      update_cache: yes
      force_apt_get: yes
      cache_valid_time: 3600

  - name: Upgrade packages
    apt:
      upgrade: dist
      force_apt_get: yes

  - name: Install apt-transport-https
    apt:
      name: apt-transport-https
      state: latest

  - name: Install ca-certificates
    apt:
      name: ca-certificates
      state: latest

  - name: Install curl
    apt:
      name: curl
      state: latest

  - name: Install gnupg-agent
    apt:
      name: gnupg-agent
      state: latest

  - name: Install software-properties-common
    apt:
      name: software-properties-common
      state: latest
```
**Role Docker**
Installer Docker et ses dépendances sur toutes les machines
Ajouter l'utilisateur ansible au groupe Docker
```
soussou@controller:~/tp3$ ansible-playbook ./roles/docker/tasks/main.yml --ask-become-pass
BECOME password:

PLAY [all] ******************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************************************************************************
ok: [worker1]
ok: [worker2]
ok: [master]

TASK [Add Docker GPG apt Key] ***********************************************************************************************************************************************************************************
ok: [worker2]
ok: [worker1]
ok: [master]

TASK [Add Docker Repository] ************************************************************************************************************************************************************************************
ok: [worker2]
ok: [master]
ok: [worker1]

TASK [Update apt and install docker-ce] *************************************************************************************************************************************************************************
ok: [worker2]
ok: [worker1]
ok: [master]

TASK [Install docker-ce-cli] ************************************************************************************************************************************************************************************
ok: [worker2]
ok: [worker1]
ok: [master]

TASK [Install containerd.io] ************************************************************************************************************************************************************************************
ok: [master]
ok: [worker2]
ok: [worker1]

TASK [Create and add the user ansible to docker group] **********************************************************************************************************************************************************
ok: [worker2]
ok: [worker1]
ok: [master]

PLAY RECAP ******************************************************************************************************************************************************************************************************
master                     : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker1                    : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker2                    : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Contenu du playbook :
```
- hosts: all
  tasks:
    - name: Add Docker GPG apt Key
      become: yes
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository
      become: yes
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
        state: present

    - name: Update apt and install docker-ce
      become: yes
      apt: update_cache=yes name=docker-ce state=latest

    - name: Install docker-ce-cli
      become: yes
      apt: update_cache=yes name=docker-ce-cli state=latest

    - name: Install containerd.io
      become: yes
      apt: update_cache=yes name=containerd.io state=latest

    - name: Create and add the user ansible to docker group
      become: yes
      user:
        name: ansible
        password: '$6$yi8CQo9XXm$pJUloGWHYRyA4B3wNP18TYcv1/QYbDW3ZNsnAqmyKSxUxluA4o29nW.ypFDFy9ZExcawZjJvZCb3lvnkuqEA5/'
        groups: docker, sudo
        state: present
        shell: /bin/bash
        createhome: yes
        home: /home/ansible
```
> password généré avec mkpass pour ne pas qu'il apparaisse en clair dans le fichier

**Rôle Kubernetes master**


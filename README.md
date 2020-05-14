# cycloid-suse
How to deploy Cycloid on SLE


Pre-Setup
============

SLE Modules
-----------
SUSEConnect -p sle-module-desktop-applications/15.1/x86_64
SUSEConnect --product sle-module-development-tools/15.1/x86_64
SUSEConnect --product PackageHub/15.1/x86_64

Installation
============

Requirements
------------

>Note: Before running ansible playbook, ensure virtualenv, pip and dependencies are satisfied on your system.

SUSE
```
zypper in python3-setuptools gcc git-core libffi-devel libopenssl-devel patterns-devel-python-devel_python3 
```

Install virtualenv
------------
```
sudo easy_install pip
sudo pip install virtualenv
```

## Récupérer ssh_id_rsa pour git via ssh sur cycloid
eval `ssh-agent -s`
ssh-add /home/cycloid/.ssh/suse_id_rsa

git clone git@github.com:cycloidio/ansible-cycloid-onprem.git
mkdir cycloid-onprem
cd cycloid-onprem
cp -r ../ansible-cycloid-onprem/playbooks/* .
virtualenv --clear .env
source .env/bin/activate

#pip install molecule ansible==2.8.* docker-py passlib bcrypt
pip install ansible==2.8.* passlib bcrypt

mkdir keys
ssh-keygen -t rsa -b 2048 -N '' -f keys/id_rsa


ansible-galaxy install -r requirements.yml --roles-path=roles -v
###
(.env) cycloid:/Cycloid/cycloid-onprem # ansible-galaxy install -r requirements.yml --roles-path=roles -v
No config file found; using defaults
- ansible-cycloid-onprem (master) is already installed, skipping.
- geerlingguy.docker (master) is already installed, skipping.
- extracting jdauphant.nginx to /Cycloid/cycloid-onprem/roles/jdauphant.nginx
- jdauphant.nginx was installed successfully
- extracting mhutter.docker-systemd-service to /Cycloid/cycloid-onprem/roles/mhutter.docker-systemd-service
- mhutter.docker-systemd-service (master) was installed successfully
###

cat >> inventory <<EOF
[cycloid]
20.20.20.10

# Meta groups to setup all in one
[cycloid-core:children]
cycloid
[cycloid-scheduler:children]
cycloid
[cycloid-redis:children]
cycloid
[cycloid-db:children]
cycloid
[cycloid-creds:children]
cycloid
[minio:children]
cycloid
[smtp-server:children]
cycloid
[cycloid-scheduler-db:children]

cycloid

EOF

##Creation environnement 
export CYCLOID_ENV=poc
##

cat >> environments/${CYCLOID_ENV}-cycloid.yml <<EOF
# Resolvable domain name from the instance and externally to use Cycloid console.
cycloid_console_dns: console.isvlab.net

# Initial Cycloid user
cycloid_initial_user:
  username: "admin"
  email: “christophe.ledorze@suse.com"
  password: “Linux1234"
  given_name: admin
  family_name: cycloid

# Pipeline engine, server configuration.
concourse_session_signing_key: "{{lookup('file', 'keys/id_rsa')}}"
concourse_host_key: "{{lookup('file', 'keys/id_rsa')}}"
concourse_authorized_worker_keys:
  - "{{lookup('file', 'keys/id_rsa.pub')}}"
EOF

##
Edit : vi /Cycloid/cycloid-onprem/roles/geerlingguy.docker/tasks/main.yml
- name: Install Docker.
  package:
          #name: "{{ docker_package }}"
          name: "docker"
          #state: "{{ docker_package_state }}"
  notify: restart docker
##
Edit : vi /Cycloid/cycloid-onprem/roles/geerlingguy.docker/defaults/main.yml
docker_compose_version: "1.25.5"
##


vi ./roles/ansible-cycloid-onprem/tasks/pre-setup.xml
---
- package:
    name:
      - python3-pyOpenSSL
      - mariadb-client
      - python3-setuptools
    state: present
##

source .env/bin/activate
export AWS_ACCESS_KEY_ID=XXXXX 
export AWS_SECRET_ACCESS_KEY=XXXXXX
aws ecr get-login --no-include-email --region eu-west-1
#eval $(aws ecr get-login --no-include-email --region eu-west-1 | sed 's|https://||')
#ssh-copy-id -i ./keys/id_rsa.pub admin@10.10.10.13

#eval `ssh-agent -s`
ssh-add /root/suse_id_rsa
ssh-add ./keys/id_rsa
export CYCLOID_ENV=poc
cp ./roles/mhutter.docker-systemd-service/vars/RedHat.yml /Cycloid/cycloid-onprem/roles/mhutter.docker-systemd-service/vars/Suse.yml
!!!!SSH KEY FORMAT !!!!!!

ansible-playbook -e env=${CYCLOID_ENV} -u admin -b -i inventory playbook.yml -c local

Verify DNS, then: 
```
export AWS_ACCESS_KEY_ID=XXXXX
export AWS_SECRET_ACCESS_KEY=XXXXX
aws ecr get-login --no-include-email --region eu-west-1
```
****************
eval $(aws ecr get-login --no-include-email --region eu-west-1 | sed 's|https://||')
docker pull vault:1.1.3
docker pull mariadb:10.2.11
docker pull mailhog/mailhog:v1.0.0
docker pull redis:5.0
docker pull postgres:9.6
docker pull concourse/concourse:5.7.2
docker pull 661913936052.dkr.ecr.eu-west-1.amazonaws.com/youdeploy-http-api:prod
****************


./roles/geerlingguy.docker/defaults/main.yml
BUG => Consider to install docker-compose manually 
docker_edition: ''
#docker_edition: 'ce'
#docker_package: "docker-{{ docker_edition }}"
docker_package: "docker"
docker_package_state: present

# Service options.
docker_service_state: started
docker_service_enabled: true
docker_restart_handler_state: restarted

# Docker Compose options.
docker_install_compose: false
docker_compose_version: "1.25.5"
docker_compose_path: /usr/local/bin/docker-compose
#####



######
cycloid:/Cycloid/ansible-cycloid-onprem # vi ./roles/geerlingguy.docker/tasks/main.yml
- name: Install Docker.
  package:
          #name: "{{ docker_package }}"
          name: "docker"
          #state: "{{ docker_package_state }}"
  notify: restart docker
#######



#########
(.env) cycloid:/Cycloid/cycloid-onprem # pip3 uninstall docker-py
WARNING: Skipping docker-py as it is not installed.
(.env) cycloid:/Cycloid/cycloid-onprem # pip3 uninstall docker
WARNING: Skipping docker as it is not installed.
(.env) cycloid:/Cycloid/cycloid-onprem # pip uninstall docker
WARNING: Skipping docker as it is not installed.
(.env) cycloid:/Cycloid/cycloid-onprem # pip uninstall docker-py
WARNING: Skipping docker-py as it is not installed.
(.env) cycloid:/Cycloid/cycloid-onprem # pip3 install docker
Collecting docker
#########



CLEANING
ansible-playbook -u admin -b -i inventory adhoc/uninstall.yml
After reboot
ansible-playbook -e env=${CYCLOID_ENV} -u admin -b -i inventory vault_unseal.yml
DEBUG
for image in `docker images | awk -F" " '{print $3}'|grep -e ^[0-9]`; do docker rmi -f $image; done

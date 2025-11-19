## Part 1 - Launch the instances

- Create 4 `Red Hat Enterprise Linux 8 (HVM). 

- ansible_control (t2.medium) (tag: Name=ansible_control)

- ansible_postgresql, ansible_nodejs, ansible_react (t2.micro) (Add 5000, 3000, 5432 to security grıops)

## Part 2 - Prepare the scene

Connect the ansible_control node
   
   - Copy the student files

   - Run the commands below to install Python3 and Ansible. 

```bash
$ sudo yum update -y
```

```bash
$ sudo yum install -y python3 
```

```bash
$ sudo yum install -y python3-pip 
```

```bash
$ pip3 install --user ansible
```

- Check Ansible's installation with the command below.

```bash
$ ansible --version
```

- Create ansible directory and change directory to this directory.

```bash
mkdir ansible
cd ansible
```

- Create `ansible.cfg` files.

```
[defaults]
host_key_checking = False
inventory=inventory_aws_ec2.yml
interpreter_python=auto_silent
private_key_file=/home/ec2-user/aduncan.pem 
remote_user=ec2-user
```

- copy pem file from local to home directory of ec2-user.

```bash
scp -i <pem-file> <pem-file> ec2-user@<public-ip of ansible_control>:/home/ec2-user
```

## Part 3 - Creating dynamic inventory

- go to AWS Management Consol and select the IAM roles:

- click the  "create role" then create a role with "AmazonEC2FullAccess"

- go to EC2 instance Dashboard, and select the control-node instance

- select actions -> security -> modify IAM role

- select the role thay you have jsut created for EC2 full access and save it.

- install "boto3"

```bash
pip3 install --user boto3
```

- Tag the postgresql, nodejs and react instances as below.

```
Name=ansible_postgresql
Name=ansible_nodejs
Name=ansible_react
```

- Tag ansible_control, ansible_postgresql, ansible_nodejs, ansible_react instances as below.

```
stack=ansible_project
```

- Tag ansible_postgresql, ansible_nodejs, ansible_react instances as below.

```
environment=development
```

- Create `inventory_aws_ec2.yml` file under the ansible directory. 

```yaml
plugin: aws_ec2
regions:
  - "us-east-1"
filters:
  tag:stack: ansible_project
keyed_groups:
  - key: tags.Name
  - key: tags.environment
compose:
  ansible_host: public_ip_address
```

```bash
$ ansible-inventory -i inventory_aws_ec2.yml --graph
```

```
@all:
  |--@_ansible_control:
  |  |--ec2-3-80-96-146.compute-1.amazonaws.com
  |--@_ansible_nodejs:
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |--@_ansible_postgresql:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |--@_ansible_react:
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |--@_development:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |--@aws_ec2:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |  |--ec2-3-80-96-146.compute-1.amazonaws.com
  |--@ungrouped:
```

- To make sure that all our hosts are reachable with dynamic inventory, we will run various ad-hoc commands that use the ping module.

```bash
$ ansible all -m ping 
```

## Part 4 - Prepare the playbook files

- Create `ansible-Project` directory under home directory and change directory to this directory.

```bash
cd ~/ansible
mkdir ansible-project
cd ansible-project
```

- Create `postgres`, `nodejs`, `react` directories.

```bash
mkdir postgres nodejs react
```

### postgres

- Copy the content of `~/student_files/todo-app-pern/database` directory to `~/ansible-project/postgres` folder.

- Change directory to `postgres` directory.

```bash
cd postgres
```

- Create a Dockerfile

```Dockerfile
FROM postgres

COPY ./init.sql /docker-entrypoint-initdb.d/

EXPOSE 5432
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as postgres playbook and name it `docker_postgre.yml`.

```yaml
---
- name: configure postgre instance
  hosts: _ansible_postgresql
  become: true
  vars_files:
    - secret.yml
  tasks:
    - name: upgrade all packages
      ansible.builtin.dnf: 
        name: '*'
        state: latest

    - name: remove Older versions of Docker if installed
      ansible.builtin.dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - podman
          - runc
        state: removed
    
    - name: Install "dnf-plugins-core"
      ansible.builtin.dnf:
        name: "dnf-plugins-core"
        state: latest

    - name: Add Docker repo
      ansible.builtin.get_url:  
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo   
  
    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install Python and pip dependencies
      ansible.builtin.dnf:
        name: 
          - python3
          - python3-pip
          - python3-devel
          - gcc
        state: present

    - name: Upgrade pip to latest version
      ansible.builtin.pip:
        name: pip
        state: latest
        executable: pip3

    - name: Install Docker SDK for Python with compatible versions
      ansible.builtin.pip:
        name:
          - docker==6.1.3
          - requests==2.31.0
          - urllib3<2.0
        state: present
        executable: pip3
        extra_args: "--force-reinstall --ignore-installed"

    - name: Add ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes
        # usermod -a -G docker ec2-user

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Reset ssh connection to pick up docker group
      ansible.builtin.meta: reset_connection

    - name: Wait for Docker daemon to be ready
      ansible.builtin.wait_for:
        path: /var/run/docker.sock
        state: present
        delay: 5
        timeout: 30

    - name: Test Docker connection
      ansible.builtin.command: docker version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout_lines

    - name: copy files to the postgresql node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/postgres/
        dest: /home/ec2-user/postgresql
    
    - name: remove cw_postgre container
      community.docker.docker_container:
        name: cw_postgre
        state: absent
        force_kill: true
    
    - name: remove clarusway/postgre image
      community.docker.docker_image:
        name: clarusway/postgre
        state: absent
    
    - name: build container image
      community.docker.docker_image:
        name: clarusway/postgre
        build:
          path: /home/ec2-user/postgresql
        source: build
        state: present
      register: image_info
    
    - name: print the image info
      ansible.builtin.debug:
        var: image_info
    
    - name: Launch postgresql docker container
      community.docker.docker_container:
        name: cw_postgre
        image: clarusway/postgre
        state: started
        ports: 
        - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
          # POSTGRES_PASSWORD: "{{password}}"
        volumes:
          - /db-data:/var/lib/postgresql/18/main
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```

- Execute it.

```bash
ansible-vault create secret.yml
```

```secret.yml
password: Pp123456789
```

```
ansible-playbook --ask-vault-pass docker_postgre.yml
```

```
ansible _ansible_postgresql -b -m shell -a "docker ps"
```

```
ansible-inventory --graph #if you get ansible host match error
```

### nodejs

- Copy the content of `~/student_files/todo-app-pern/server` directory to `~/ansible-project/nodejs` folder.


- Change directory to `~/ansible-project/nodejs` directory.

```bash
cd ~/ansible-project/nodejs
```

- Create a Dockerfile.

```Dockerfile
FROM node:14

# Create app directory
WORKDIR /usr/src/app


COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production


# copy all files into the image
COPY . .

EXPOSE 5000

CMD ["node","app.js"]
```

- Change the `~/ansible-project/nodejs/.env` file as below.

```
SERVER_PORT=5000
DB_USER=postgres
DB_PASSWORD=Pp123456789
DB_NAME=clarustodo
DB_HOST=172.31.12.133 # (private ip of postgresql instance)
DB_PORT=5432
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as nodejs playbook and name it `docker_nodejs.yml`.

```yaml
---
- name: configure nodejs instance
  hosts: _ansible_nodejs
  become: true
  tasks:
    - name: upgrade all packages
      ansible.builtin.dnf: 
        name: '*'
        state: latest
 
    - name: remove Older versions of Docker if installed
      ansible.builtin.dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - podman
          - runc
        state: removed
    
    - name: Install "dnf-plugins-core"
      ansible.builtin.dnf:
        name: "dnf-plugins-core"
        state: latest

    - name: Add Docker repo
      ansible.builtin.get_url:  
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo   
  
    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install Python and pip dependencies
      ansible.builtin.dnf:
        name: 
          - python3
          - python3-pip
          - python3-devel
          - gcc
        state: present

    - name: Upgrade pip to latest version
      ansible.builtin.pip:
        name: pip
        state: latest
        executable: pip3

    - name: Remove system-installed requests package
      ansible.builtin.dnf:
        name: python3-requests
        state: absent

    - name: Install Docker SDK for Python with compatible versions
      ansible.builtin.pip:
        name: 
          - docker==6.1.3
          - requests==2.31.0
          - urllib3<2.0
        state: present
        executable: pip3


    - name: Add ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes
        # usermod -a -G docker ec2-user

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Reset ssh connection to pick up docker group
      ansible.builtin.meta: reset_connection

    - name: Wait for Docker daemon to be ready
      ansible.builtin.wait_for:
        path: /var/run/docker.sock
        state: present
        delay: 5
        timeout: 30

    - name: Test Docker connection
      ansible.builtin.command: docker version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout_lines

    
    - name: copy files to the nodejs node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/nodejs/
        dest: /home/ec2-user/nodejs
    
 
    - name: remove cw_nodejs container
      community.docker.docker_container:
        name: cw_nodejs
        state: absent
        force_kill: true

    - name: remove clarusway/nodejs image
      community.docker.docker_image:
        name: clarusway/nodejs
        state: absent
    
    - name: build container image
      community.docker.docker_image:
        name: clarusway/nodejs
        build:
          path: /home/ec2-user/nodejs
        source: build
        state: present
      register: image_info
     
    - name: print the image_info
      ansible.builtin.debug:
        var: image_info
    
    - name: Launch nodejs docker container
      community.docker.docker_container:
        name: cw_nodejs
        image: clarusway/nodejs
        state: started
        ports:
          - "5000:5000"
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info      
```

- Execute it.

```
ansible-playbook docker_nodejs.yml
```

```
ansible _ansible_nodejs -b -m shell -a "docker ps"
```

```
ansible-inventory --graph #if you get ansible host match error
```

### react

- Copy the content of `~/student_files/todo-app-pern/client` directory to `~/ansible-project/react` folder.

- Change directory to `~/ansible-Project/react` directory.

```bash
cd ~/ansible-project/react
```

- Create a Dockerfile.

```Dockerfile
FROM node:14

# Create app directory
WORKDIR /app


COPY package*.json ./

RUN npm install

# copy all files into the image
COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

- Change the `~/ansible-project/react/.env` file as below.

```
REACT_APP_BASE_URL=http://<public ip of nodejs>:5000/
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as react playbook and name it `docker_react.yml`.

```yaml
---
- name: configure react instance
  hosts: _ansible_react
  become: true
  tasks:
    - name: upgrade all packages
      ansible.builtin.dnf: 
        name: '*'
        state: latest
 
    - name: remove Older versions of Docker if installed
    # we may need to uninstall any existing docker files from the centos repo first. 
      ansible.builtin.dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - podman
          - runc
        state: removed
    
    - name: Install "dnf-plugins-core"
      ansible.builtin.dnf:
        name: "dnf-plugins-core"
        state: latest


    - name: Add Docker repo
      ansible.builtin.get_url:  
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo   
  
    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install Python and pip dependencies
      ansible.builtin.dnf:
        name: 
          - python3
          - python3-pip
          - python3-devel
          - gcc
        state: present

    - name: Upgrade pip to latest version
      ansible.builtin.pip:
        name: pip
        state: latest
        executable: pip3

    - name: Remove system-installed requests package
      ansible.builtin.dnf:
        name: python3-requests
        state: absent


    - name: Install Docker SDK for Python with compatible versions
      ansible.builtin.pip:
        name: 
          - docker==6.1.3
          - requests==2.31.0
          - urllib3<2.0
        state: present
        executable: pip3

    - name: Add ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes
        # usermod -a -G docker ec2-user

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Reset ssh connection to pick up docker group
      ansible.builtin.meta: reset_connection

    - name: Wait for Docker daemon to be ready
      ansible.builtin.wait_for:
        path: /var/run/docker.sock
        state: present
        delay: 5
        timeout: 30

    - name: Test Docker connection
      ansible.builtin.command: docker version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout_lines

    - name: copy files to the react node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/react/
        dest: /home/ec2-user/react
    
 
    - name: remove cw_react container
      community.docker.docker_container:
        name: cw_react
        state: absent
        force_kill: true

    - name: remove clarusway/react image
      community.docker.docker_image:
        name: clarusway/react
        state: absent
    
    - name: build docker image
      community.docker.docker_image:
        name: clarusway/react
        build:
          path: /home/ec2-user/react
        source: build
        state: present
      register: image_info
     
    - name: print the image_info
      ansible.builtin.debug:
        var: image_info
    
    - name: run react docker container
      community.docker.docker_container:
        name: cw_react
        image: clarusway/react
        state: started
        ports:
          - "3000:3000"
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```

- Execute it.

```
ansible-playbook docker_react.yml
```

## Part 5 - Prepare one playbook file for all instances.

- Create a `docker_project.yml` file under `the ~/ansible` folder.

```yaml
---
- name: Install docker and configure
  hosts: _development
  become: true
  tasks:
    - name: upgrade all packages
      ansible.builtin.dnf: 
        name: '*'
        state: latest
 
    - name: remove Older versions of Docker if installed
      ansible.builtin.dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - podman
          - runc
        state: removed
    
    - name: Install "dnf-plugins-core"
      ansible.builtin.dnf:
        name: "dnf-plugins-core"
        state: latest

    - name: Add Docker repo
      ansible.builtin.get_url:  
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo   
  
    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install Python and pip dependencies
      ansible.builtin.dnf:
        name: 
          - python3
          - python3-pip
          - python3-devel
          - gcc
        state: present

    - name: Upgrade pip to latest version
      ansible.builtin.pip:
        name: pip
        state: latest
        executable: pip3

    - name: Remove system-installed requests package
      ansible.builtin.dnf:
        name: python3-requests
        state: absent

    - name: Install Docker SDK for Python with compatible versions
      ansible.builtin.pip:
        name: 
          - docker==6.1.3
          - requests==2.31.0
          - urllib3<2.0
        state: present
        executable: pip3

    - name: Add ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes
        # usermod -a -G docker ec2-user

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Reset ssh connection to pick up docker group
      ansible.builtin.meta: reset_connection

    - name: Wait for Docker daemon to be ready
      ansible.builtin.wait_for:
        path: /var/run/docker.sock
        state: present
        delay: 5
        timeout: 30

    - name: Test Docker connection
      ansible.builtin.command: docker version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout_lines

# Now, what were the different parts? PostgreSQL, NodeJS, and React. Now, we'll start with PostgreSQL and work on the different parts. We'll do it for PostgreSQL first.

- name: configure postgre instance
  hosts: _ansible_postgresql
  become: true
  vars_files:
    - secret.yml
  vars:
    container_path: /home/ec2-user/postgresql
    container_name: cw_postgre
    image_name: clarusway/postgre

  tasks:
    - name: copy files to the postgresql node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/postgres/
        dest: "{{ container_path }}"

    - name: remove "{{ container_name }}" container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true
    
    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent
    
    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present
      register: image_info
    
    - name: print the image_info
      ansible.builtin.debug:
        var: image_info

    - name: launch postgresql docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5432:5432"
        env:
          POSTGRES_PASSWORD: "{{ password }}"
        volumes:
          - /db-data:/var/lib/postgresql/18/main
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info

# Now we are going to perform the steps to configure the nodejs server.
- name: nodejs configuration
  hosts: _ansible_nodejs
  become: true
  vars:
    container_path: /home/ec2-user/nodejs
    container_name: cw_nodejs
    image_name: clarusway/nodejs

  tasks:
    - name: copy files to the nodejs node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/nodejs/
        dest: "{{ container_path }}"
    
    - name: remove "{{ container_name }}" container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}" 
        state: absent
    
    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present
      register: image_info
     
    - name: print the image_info
      ansible.builtin.debug:
        var: image_info
    
    - name: launch nodejs docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5000:5000"
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info

# Now we are configuring the React server.
- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  vars:
    container_path: /home/ec2-user/react
    container_name: cw_react
    image_name: clarusway/react

  tasks:
    - name: copy files to the react node
      ansible.builtin.copy:
        src: /home/ec2-user/ansible/ansible-project/react/
        dest: "{{ container_path }}"
 
    - name: remove "{{ container_name }}" container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent
    
    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present
      register: image_info
     
    - name: print the image_info
      ansible.builtin.debug:
        var: image_info
    
    - name: launch react docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "3000:3000"
      register: container_info

    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```

- Execute it.

```bash
ansible-playbook --ask-vault-pass  docker_project.yml
```


## Part 6 - Prepare playbook with roles solution.

- Cretae a role folder in /home/ec2-user/ansible. Then create role folders.

mkdir roles && cd roles
ansible-galaxy init docker
ansible-galaxy init postgre
ansible-galaxy init nodejs
ansible-galaxy init react

- Add the roles_path=/home/ec2-user/ansible/roles to the ansible.cfg.

- Go to the /home/ec2-user/ansible/roles/docker/tasks/main.yml and copy the following.

```yaml
    - name: upgrade all packages
      ansible.builtin.dnf: 
        name: '*'
        state: latest
 
    - name: remove Older versions of Docker if installed
      ansible.builtin.dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - podman
          - runc
        state: removed
    
    - name: Install "dnf-plugins-core"
      ansible.builtin.dnf:
        name: "dnf-plugins-core"
        state: latest

    - name: Add Docker repo
      ansible.builtin.get_url:  
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo   
  
    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install Python and pip dependencies
      ansible.builtin.dnf:
        name: 
          - python3
          - python3-pip
          - python3-devel
          - gcc
        state: present

    - name: Upgrade pip to latest version
      ansible.builtin.pip:
        name: pip
        state: latest
        executable: pip3

    - name: Remove system-installed requests package
      ansible.builtin.dnf:
        name: python3-requests
        state: absent

    - name: Install Docker SDK for Python with compatible versions
      ansible.builtin.pip:
        name: 
          - docker==6.1.3
          - requests==2.31.0
          - urllib3<2.0
        state: present
        executable: pip3

    - name: Add ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes
        # usermod -a -G docker ec2-user

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Reset ssh connection to pick up docker group
      ansible.builtin.meta: reset_connection

    - name: Wait for Docker daemon to be ready
      ansible.builtin.wait_for:
        path: /var/run/docker.sock
        state: present
        delay: 5
        timeout: 30

    - name: Test Docker connection
      ansible.builtin.command: docker version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout_lines
```


- Go to the /home/ec2-user/ansible/roles/postgre/tasks/main.yml and copy the followings.

```yaml
    - name: copy the Dockerfile
      ansible.builtin.copy:
        src: postgres/ # write only file name
        dest: "{{ container_path }}"

    - name: remove {{ container_name }} container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent

    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present

    - name: Launch postgresql docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports: 
        - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /db-data:/var/lib/postgresql/18/main
      register: container_info
    
    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```


- Copy /home/ec2-user/ansible/ansible-project/postgres folder to /home/ec2-user/ansible/roles/postgre/files.

- Copy these variables to /home/ec2-user/ansible/roles/postgre/vars/main.yml.

```
container_path: /home/ec2-user/postgresql
container_name: cw_postgre
image_name: clarusway/postgre
```

- Go to the /home/ec2-user/ansible/roles/nodejs/tasks/main.yml and copy the followings.

```yaml
    - name: copy files to the nodejs node
      copy:
        src: nodejs/
        dest: "{{ container_path }}"

    - name: remove {{ container_name }} container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent
  
    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present

    - name: Launch postgresql docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports: 
        - "5000:5000"
      register: container_info
    
    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```

- Copy /home/ec2-user/ansible/ansible-project/nodejs folder to /home/ec2-user/ansible/roles/nodejs/files.


- Copy these variables to /home/ec2-user/ansible/roles/nodejs/vars/main.yml.

```
container_path: /home/ec2-user/nodejs
container_name: cw_nodejs
image_name: clarusway/nodejs
```

- Go to the /home/ec2-user/ansible/roles/react/tasks/main.yml and copy the followings.

```yaml
    - name: copy files to the react node
      copy:
        src: react/
        dest: "{{ container_path }}"

    - name: remove {{ container_name }} container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove "{{ image_name }}" image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent
  
    - name: build container image
      community.docker.docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present

    - name: Launch postgresql docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports: 
        - "3000:3000"
      register: container_info
    
    - name: print the container info
      ansible.builtin.debug:
        var: container_info
```

- Copy /home/ec2-user/ansible/ansible-project/react folder to /home/ec2-user/ansible/roles/react/files.

- Copy these variables to /home/ec2-user/ansible/roles/react/vars/main.yml.

```
container_path: /home/ec2-user/react
container_name: cw_react
image_name: clarusway/react
```

- Update the ansible.cfg file. Add roles_path as below.

```cfg
roles_path=/home/ec2-user/ansible/roles
```

- Go to the /home/ec2-user/ansible/ and create a playbook.

```
cd /home/ec2-user/ansible/
nano play-role.yml
```

```yaml
- name: Docker install and configuration
  hosts: _development
  become: true
  roles: 
    - docker
- name: Postgre Database configuration
  hosts: _ansible_postgresql
  become: true
  roles:
    - postgre
- name: Nodejs server configuration
  hosts: _ansible_nodejs
  become: true
  roles:
    - nodejs
- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  roles:
    - react
```

- Execute the follow.

```bash
ansible-playbook play-role.yml
```

## Part 7 - Using role from ansible-galaxy.

- Search for a docker role.

```bash
ansible-galaxy search docker --platform EL | grep geerl
```

- Change directory to ansible folder.

```bash
cd /home/ec2-user/ansible
```

- install the role.

```bash
ansible-galaxy install geerlingguy.docker
```

- Create a new play-newrole.yml

```yaml
- name: Docker install and configuration
  hosts: _development
  become: true
  roles: 
    - geerlingguy.docker
- name: Postgre Database configuration
  hosts: _ansible_postgresql
  become: true
  roles:
    - postgre
- name: Nodejs server configuration
  hosts: _ansible_nodejs
  become: true
  roles:
    - nodejs
- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  roles:
    - react
```

- Execute the follow.

```bash
ansible-playbook play-newrole.yml
```
# enRoute sysadmin technical test write up

## Table of Contents
- [Objectives](#objectives)
- [Detailed Process](#detailed-process)
  - [Creating the VMs](#creating-the-vms)
    - [Downloading ISO via GUI](#downloading-ISO-via-GUI)
    - [Attempting to create the Ansible control node via CLI](#attempting-to-create-the-Ansible-control-node-via-CLI)
    - [Creating the Ansible control-node via GUI](#creating-the-Ansible-control-node-via-GUI)
    - [Installing Debian on control-node via CLI](#installing-Debian-on-control-node-via-CLI)
    - [Creating the Gatus worker node](#creating-the-Gatus-worker-node)
  - [Setting up the network](#setting-up-the-network)
    - [Blocking 141.98.11.59](#blocking-141.98.11.59)
    - [Modifying interfaces on control-node](#modifying-interfaces-on-control-node)
    - [Modifying interfaces on gatus-node](#modifying-interfaces-on-gatus-node)
    - [Verifying IP forwarding on PVE host](#verifying-IP-forwarding-on-PVE-host)
- [Prepared steps that couldn’t be tried out](#prepared-steps-that-couldn’t-be-tried-out)
  - [Setting up Ansible]()
  - [Editing the Docker role]
  - [Editing the Gatus role]
  - [Editing the Gatus playbook]
- [Possible Further steps]

## Objectives

Using a Proxmox-Debian-Compose-Ansible stack, build a simplified infrastructure under 3 hours to:
1. Deploy Gatus with an accessible https
2. Monitor enroute.mobi, chouette.enroute.mobi and ara.enroute.mobi
3. Document the process in this markdown write up

## Detailed Process

Creating and installing the VMs ended up being more problematic and time-consuming than expected. Network configuration in particular couldn’t be solved within the target 3 hours. 2 more hours later, still unsolved.

### **Creating the VMs**

#### Downloading ISO via GUI

Navigate throught:
> em-vibrant-austin > localnetwork > local > ISO Images > Download from URL

Enter:
> URL: `https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/debian-13.5.0-amd64-DVD-1.iso`
>
>File name: `debian-13.5.0-amd64-DVD-1.iso`

Making sure it got downloaded in `/var/lib/vz/template/iso`:
```console
ls /var/lib/vz/template/iso
```

#### Attempting to create the Ansible control node via CLI

```console
qm create 100 --name control-node --memory 4096 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/debian-13.5.0-amd64-DVD-1.iso,format=iso
```

> Iso not among the valid formats for scsi0

Copying the link for the QEMU VM in *qcow2* format for the 64 bits AMD/Intel architecture, from [the official Debian distro page](https://www.debian.org/distrib/):

> https://cloud.debian.org/images/cloud/trixie/latest/debian-13-nocloud-amd64.qcow2

Downloading the VM on Proxmox via CLI:
```console
wget -O /var/lib/vz/images/debian-13-nocloud-amd64.qcow2 https://cloud.debian.org/images/cloud/trixie/latest/debian-13-nocloud-amd64.qcow2, format=qcow2
```

> proxmox server doesn’t have `/dev/pve` so no local-lvm

#### Creating the Ansible control-node via GUI

Back in “ISO Images”, click the “Create VM” button and fill in form:
> In General:
>
>> Name: control-node
>
>In OS:
>
>> ISO image: debian-13.5.0-amd64-DVD-1.iso
>
>In CPU:
>
>> Cores: 2
>
>In Memory:
>
>> Memory (MiB): 4096

Leave everything else by default.

#### Installing Debian on control-node via CLI

In GUI, go to:
> em-vibrant-austin > 100 > Console
>
> Click the “Start Now” button

In the console:
> Press “Enter”
>
> Language: Type in `20`
>
> Region: Type in `19`
>
> Continent: Type in `7`
>
> Country: Type in `17`
>
> Language-Country combination: Type in `15`
>
>> ISO-qwerty
>
> Keyboard: Type in `12`
>
> Name server: Press "Enter"
>
> Hostname: Type `control-node`
>
> Domain name: Press "Enter"
>
> Root password: Press "Enter" twice
>
> Full name: Ansible User
>
> username: ansible-user
>
> Password: 
>
>> As research shows, longer readable passwords (like passphrases, made of multiple words, e.q. thisismysuperpassphrase) are more effective than shorter nonsensical passwords containing digits, uppercase letters and symbols (e.g. t1M$pwD!). Thus, NIST recommends using them in [SP 800-63B]( https://pages.nist.gov/800-63-4/sp800-63b/passwords/).
>
> Partitioning: Type in `1`
>
> Disk: Type in `1`
>
> Partitioning schema: Type in `1`
>
> Partitioning overview: Type in `12`
>
> “Write changes to discks?”: Type in `1`
>
> Scan additional media: Type in `2`
>
> “Use a network mirror?”: Type in `2`
>
> Package usage survey: Type in `2`
>
> Collection installation: Type in `12`
>
>> Server needs no desktop and no SSH: will be administrating straight through tty
>>
>> If ansible won’t work without SSH, install manually with `sudo apt update` `sudo apt install openssh-server` and start with `sudo systemctl enable --now ssh`
>
> Install GRUB: Type in `1`
>
> Drive selection: Type in `2`
>
> Press “Enter” to reboot

#### Creating the Gatus worker node

Follow all previous steps identically to create VM and install Debian, except:
> In VM creation:
>
>> VM ID: 200
>>
>> Name: gatus-node
>
> In Debian installation:
>
>> Hostname: Type `gatus-node`

***

### **Setting up the network**

#### Blocking 141.98.11.59
Reported brute-forcing Lithuanian IP, noticed in numbers in System Logs
On host:
```console
iptables -I INPUT -s 141.98.11.59 -j DROP
```

#### Modifying interfaces on control-node

```console
sudo nano /etc/network/interfaces
```
> `ip a` to check interface name, “ens18” in our case

Under “#The primary network interface”, add a static IP form the same subnet as the host:
```/etc/network/interfaces
auto ens18
iface ens18 inet static
	address <guest_IP>/24
	gateway <host_IP>
```

With nano:
> Ctrl + O > Enter > Ctrl + X to save and quit

With vi:
> In ’normal mode’, type `i` to enter ’insert mode’ and edit file as needed
>
> Press ’Esc’ to go back to ’normal mode’ and type `: wq` to save and exit vi; or type `:q` to discard changes and exit vi

> If changes were made to /etc/network/interfaces use `sudo systemctl restart networking` to apply

#### Modifying interfaces on gatus-node

Do the same as above with another IP from the same subnet:
```/etc/network/interfaces
auto ens18
iface ens18 inet static
	address <second-guest_IP>/24
	gateway <host_IP>
```

#### Verifying IP forwarding on PVE host

```console
sysctl net.ipv4.ip_forward
```

> Output is indeed `net.ipv4.ip_forward = 1` meaning forwarding is active but VMs still can’t ping gateway or DNS, though host can access the internet


## Prepared steps that couldn’t be tried out

The following steps were planed ahead via researching the documentations, but didn’t get the chance to be tested. The docker-compose deployment in particular is definitely incomplete.

### **Setting up Ansible**

#### Installing python virtualenv and sshpass pacquets on control-node

```console
sudo apt install python3.11-venv sshpass
```

> virtualenv (venv) will allow us to isolate any number of ansible installations and executions in their own virtual environment
>
> sshpass will allow ansible to ssh into the worker nodes


#### Identifying gatus node hostname

```console
vi /etc/hosts
```

> add gatus…

#### Fingerprinting SSH connexion to gatus node

```console
ssh ansible-user@gatus
```

> Type `yes` 
>
> Then type `exit` to return to control node

#### Creating a virtual environment named “ansible”

```console
virtualenv ansible
```

#### Activating the virtual environment 

```console
source ansible/bin/activate
```

#### Installing newest ansible version in the virtual environment

```console
pip install ansible
```

> check installed version with `ansible --version`
>
> check ansible installation with `ansiblebin/ansible* -l`

#### Creating the ansible inventory

```console
vi inventaire.ini
```

inventaire.ini
```inventaire.ini
[gatus]
gatus
```

#### Creating encrypted password

```console
ansible localhost -i inventaire.ini -m debug -a "msg={{'<PWD>'|password_hash('sha512','secretsalt')}}"
```
> substitute `<PWD>` with the password to encrypt
>
> Copy `<HASH>` from output `{ “msg”: “<HASH>” }`

#### Creating SSH keys

```console
ssh-keygen -t ecdsa
```

> Leave every option by default
>
>> The SSH passphrase is optional

#### Pushing the public key to all nodes

```console
ansible -i inventaire.ini -m authorized_key -a 'user=ansible-user state=present key="{{ lookup("file", "/home/ansible-user/.ssh/id_ecdsa.pub") }}"' --user ansible-user --ask-pass --become --ask-become-pass all
```

#### Adding the docker community collection

```console
ansible-galaxy collection install community.docker
```

#### Creating Ansible roles

```console
mkdir roles
cd roles
```

```console
mkdir -p docker/tasks
touch docker/tasks/main.yaml
```

```console
mkdir -p gatus/tasks
touch gatus/tasks/main.yaml
```

***

### **Editing the Docker role**

```console
vi /docker/tasks/main.yaml
```

roles/docker/tasks/main.yaml
```roles/docker/tasks/main.yaml
---
- name: Update apt packages
  become: true
  apt:
    update_cache: yes

- name: Install required packages
  ansible.builtin.apt:
    name: 	#packages unsure whether to install or not are commented
      #- apt-transport-https
      - ca-certificates
      - curl
      #- gnupg
      #- lsb-release
      #- python3-docker
    state: present
    update_cache: yes
  become: true

- name: Create APT keyrings directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"
  become: true

- name: Add Docker GPG key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/debian/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: "0644"
  become: true

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable"
    state: present
    filename: docker
  become: true

- name: Update apt packages again	#as indicated in official documentation
  become: true
  apt:
    update_cache: yes

- name: “install docker packages”
   apt:
      name:
        - docker-ce
        - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
     state: latest

- name: Ensure Docker is running
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: true
  become: true
```

> Constructed as to follow the steps to install Docker Engine [using the `apt` repostory]( https://docs.docker.com/engine/install/debian/#install-using-the-repository) from the [Official Docker documentation](docs.docker.com)

***

### **Editing the Gatus role**

```console
vi /docker/tasks/main.yaml
```

roles/docker/tasks/main.yaml
```roles/docker/tasks/main.yaml
---
- name: Deploy Gatus via Docker Compose
  hosts: gatus
  become: true

  tasks:
    - name: Create application directory
      ansible.builtin.file:
        path: /opt/gatus
        state: directory
        mode: "0755"

    - name: Copy docker compose.yaml to the server
      ansible.builtin.copy:
        content: |
      services:
        gatus:
          image: twinproduction/gatus:latest
#or build: context: ./gatus (rep containing custom dockerfile) ?
          ports:
           - 8080:8080	#or 443:8080?
          volumes:
           - ./config:/config		#really?
#also copy /config/config.yaml?

    -name: docker compose build	# if necessary

    - name: docker compose up -d	#backgroud mod
```

> docker compose ps(containers), top(process), stat(resources), logs(-f for real time)
>
> docker compose stop (to pause)
>
>docker compose down (to remove services; -v to also remove created volumes)

#### From the Gatus documentation

```console
docker run -p 8080:8080 --mount type=bind,source="$(pwd)"/config.yaml,target=/config/config.yaml --name gatus ghcr.io/twin/gatus:stable
```

> https://github.com/TwiN/gatus/blob/master/.examples/docker-compose/compose.yaml

compose.yaml
```compose.yaml
services:
  gatus:
    image: twinproduction/gatus:latest
    ports:
      - 8080:8080
    volumes:
      - ./config:/config
	# If running multiple services, add:
     #networks:
       #- metrics

#networks:
  #metrics:
    #driver: bridge
```

> https://github.com/TwiN/gatus/blob/master/.examples/docker-compose/config/config.yaml

/config/config.yaml
```config/config.yaml
endpoints:
  - name: website                 # Name of your endpoint, can be anything
    url: "https://enroute.mobi"
    interval: 5m                  # Duration to wait between every status check (default: 60s, other example: 30s)
    conditions:
      - "[STATUS] == 200"         # Status must be 200
	# Example of extra conditions:
      - "[BODY].status == UP"     # The json path "$.status" must be equal to UP
      - "[RESPONSE_TIME] < 300"   # Response time must be under 300ms

    - name: chouetteSaas
     url: " https://chouette.enroute.mobi"
     interval: 5m        
     conditions:
      - "[STATUS] == 200"         

    - name: araSaas
     url: " https:// ara.enroute.mobi"
     interval: 5m        
     conditions:
      - "[STATUS] == 200"         
```

### **Editing the gatus playbook**

```console
cd ..
vi gatus-playbook.yaml
```

./gatus-playbook.yaml
```./gatus-playbook.yaml
---
- name: Spinning up Gatus
  hosts: docker
  roles:
    - role: “docker”
    - role: “gatus”
```

Run the playbook with:
```console
ansible-playbook -i inventaire.ini --user user-ansible --become --ask-become-pass gatus-playbook.yaml
```

## Possible Further steps

### Adding postgres

https://github.com/TwiN/gatus/blob/master/.examples/docker-compose-postgres-storage/compose.yaml

### Integrating Grafana-pronetheus

https://github.com/TwiN/gatus/blob/master/.examples/docker-compose-grafana-prometheus/compose.yaml
```compose.yaml
prometheus:
    container_name: prometheus
    image: prom/prometheus:v3.5.0
    restart: always
    command: --config.file=/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - metrics

  grafana:
    container_name: grafana
    image: grafana/grafana:12.1.0
    restart: always
    environment:
      GF_SECURITY_ADMIN_PASSWORD: secret
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/grafana.ini/:/etc/grafana/grafana.ini:ro
      - ./grafana/provisioning/:/etc/grafana/provisioning/:ro
    networks:
      - metrics

networks:
  metrics:
    driver: bridge
```

### Configuring gatus alerts

Possibly to:
- datadog
- email addresses
- opsgenie
- slack
- splunk
- telegram

### Deploying via Terraform and Kubernetes rather than compose

https://github.com/TwiN/terraform-kubernetes-gatus

### Adding the monitored services to the infrastructure handler actions to automatically reboot them when detected unhealthy 

### Clustering proxmox hosts for high availability

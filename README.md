This article explains how to set up a Docker environment to use Ansible for managing containers via SSH. The setup includes two Docker containers: one running Ansible as an SSH client and another running an Ubuntu-based SSH server. The goal is to SSH into the Ansible container and, from there, SSH into the Ubuntu container to execute Ansible commands.

## Project Structure

The project is organized as follows:

```bash
.
├── ansible-image
│   └── ubuntu
│       └── Dockerfile # Dockerfile for ansible
├── docker-compose.yaml # docker compose file to run ansible and ubuntu server
├── README.md
└── ssh-image
    └── ubuntu
        └── Dockerfile # Dockerfile for ubuntu-server
```

## Step 1: Setting Up the Docker Environment


### Ansible Container Dockerfile

The `Dockerfile` for the Ansible container installs necessary packages (`openssh-server`, `openssh-client`, `ansible`, etc.) and configures SSH to allow root login with password and public key authentication.

```dockerfile
FROM ubuntu

RUN apt-get update && \
    apt-get install -y openssh-server openssh-client ansible sudo curl gnupg wget && \
    apt-get clean

RUN mkdir /var/run/sshd

RUN echo "root:root" | chpasswd

RUN echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
    sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'AuthorizedKeysFile .ssh/authorized_keys' >> /etc/ssh/sshd_config

RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh

CMD ["/usr/sbin/sshd", "-D"]
```

### Ubuntu SSH Server Dockerfile

The `ssh-server/Dockerfile` is similar, setting up an Ubuntu container with an SSH server for Ansible to manage.

```dockerfile
FROM ubuntu

RUN apt-get update && \
    apt-get install -y openssh-server openssh-client ansible sudo curl gnupg wget && \
    apt-get clean

RUN mkdir /var/run/sshd

RUN echo "root:root" | chpasswd

RUN echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
    sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'AuthorizedKeysFile .ssh/authorized_keys' >> /etc/ssh/sshd_config

RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh

CMD ["/usr/sbin/sshd", "-D"]
```
### Docker Compose Configuration

The `docker-compose.yaml` file defines two services: `ansible` (Ansible client) and `ubuntu-ssh-server` (Ubuntu SSH server). Both containers are connected to a custom bridge network (`ansible-net`) for communication.

```yaml
services:
  ansible-ssh:
    build:
      context: ./ansible-image/ubuntu
      dockerfile: Dockerfile
    container_name: ansible
    hostname: ansible
    ports:
      - "2222:22"
    volumes:
      - ansible-data:/opt/ansible-data
    networks:
      - ansible-net
    restart: unless-stopped
    depends_on:
      - ubuntu-ssh-server

  ubuntu-ssh-server:
    build:
      context: ./ssh-image/ubuntu
      dockerfile: Dockerfile
    container_name: ubuntu-ssh-server
    hostname: ubuntu-1
    ports:
      - "2223:22"
    volumes:
      - ssh-data:/opt/ssh-data
    networks:
      - ansible-net
    restart: unless-stopped

volumes:
  ansible-data:
  ssh-data:

networks:
  ansible-net:
    driver: bridge
```


## Step 2: Configuring SSH Access

First, run the Docker containers using the following command

```bash
docker compose up --build
```

To enable SSH from the Ansible container to the Ubuntu container, follow these steps:

1. **Generate SSH Keys on the Host**:
   On the host machine, generate an SSH key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
   ```

2. **Copy the Public Key to the Ansible Container**:
   Copy the public key to the Ansible container and append it to the `authorized_keys` file:
   ```bash
   docker cp ~/.ssh/id_rsa.pub ansible:/root/.ssh/temp_key.pub
   docker exec ansible bash -c "cat /root/.ssh/temp_key.pub >> /root/.ssh/authorized_keys && rm /root/.ssh/temp_key.pub && chmod 600 /root/.ssh/authorized_keys && chown root:root /root/.ssh/authorized_keys"
   ```

3. **Access the Ansible Container**:
   Find the IP address of the Ansible container:
   ```bash
   docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ansible
   ```
   SSH into the Ansible container (e.g., `ssh root@<ansible-container-ip>`).

4. **Generate SSH Keys Inside the Ansible Container**:
   Inside the Ansible container, generate a new SSH key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
   ```

5. **Copy the Ansible Container’s Public Key to the Ubuntu Container**:
   Find the IP address of the Ubuntu container:
   ```bash
   docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ubuntu-ssh-server
   ```
   Copy the Ansible container’s public key to the Ubuntu container (e.g., IP `172.18.0.2`):
   ```bash
   ssh-copy-id root@172.18.0.2
   ```
   Enter the password (`root`) when prompted.

6. **Test SSH Access**:
   From the Ansible container, test SSH access to the Ubuntu container:
   ```bash
   ssh root@172.18.0.2
   ```

## Step 3: Running Ansible Commands

With SSH configured, you can now run Ansible commands from the Ansible container to manage the Ubuntu container. For example, create an Ansible inventory file and playbook:

Create an inventory file`inventory.yaml`
```yaml
all:
  hosts:
    ubuntu-ssh-server:
      ansible_host: 172.18.0.2
      ansible_user: root
```

Create a playbok `playbook.yaml`
```yaml
- name: Test Ansible Playbook
  hosts: ubuntu-ssh-server
  tasks:
    - name: Ensure a test file exists
      file:
        path: /opt/ssh-data/test.txt
        state: touch
```

Run the playbook from the Ansible container:
```bash
ansible-playbook -i inventory.yaml playbook.yaml
```

## Conclusion

You can modify the `Dockerfile` and `docker-compose.yaml` to run Ansible and SSH Servers according to your needs.
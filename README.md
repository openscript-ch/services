# Services

This is an umbrella ☂️ project for the services we use as our infrastructure. All services are living inside Docker containers.

- [Development operations](https://github.com/openscript-ch/devops-services/)
- [Identity and Access Management](https://github.com/openscript-ch/lms-services)
- [Learning management system](https://github.com/openscript-ch/lms-services/)

## Host set up

A host is set up as follows. The cloud init script does the same.

### Environment

These steps describe how to set up the system environment on Ubuntu 20.04 LTS on ARM or x86_64:

1. Install updates

   ```bash
   apt update && apt upgrade
   ```

1. Install Docker dependencies

   ```bash
   apt install apt-transport-https ca-certificates curl gnupg lsb-release
   ```

1. Import Dockers GPG key

   ```bash
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   ```

1. Add Dockers apt repository
   - On x86_64

     ```bash
     echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
     ```

   - On ARM

     ```bash
     echo "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
     ```

1. Install Docker

   ```bash
   apt update && apt install docker-ce docker-ce-cli containerd.io
   ```

1. Create directory for Docker cli plugins

   ```bash
   mkdir -p /usr/local/lib/docker/cli-plugins
   ```

1. Download `docker-compose` executable
   - On x86_64

     ```bash
     curl -SL https://github.com/docker/compose/releases/download/v2.4.1/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
     ```

   - On ARM

     ```bash
     curl -SL https://github.com/docker/compose/releases/download/v2.4.1/docker-compose-linux-aarch64 -o /usr/local/lib/docker/cli-plugins/docker-compose
     ```

1. Give executable permission to Docker Compose

   ```bash
   chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
   ```

1. Validate the installation of Docker (>= `20.10.14`)

   ```bash
   docker -v
   ```

1. Validate the installation of Docker Compose (>= `2.4.1`)

   ```bash
   docker-compose -v
   ```

The following sources were used:

- [Documentation: Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Documentation: Docker Compose V2](https://docs.docker.com/compose/cli-command/#installing-compose-v2)

### Application system

1. Clone this repository into `/srv/lms-services`

   ```bash
   git clone https://github.com/openscript-ch/lms-services /srv/lms-services
   ```

1. Make sure that the DNS is routing all subdomains to the host where the individual services run on.

   ```txt
   *.lms 10800 IN CNAME lms.example.com.`
   `lms 10800 IN A 999.999.999.999`
   `lms 10800 IN AAAA xxxx:xxxx:xxxx:xxxx::1`
   ```

1. Replace all `example.com` with the domain where the application runs on.

   ```bash
   sed -i "s/example.com/example.ch/g" docker-compose.yml`
   ```

1. Change contact email for SSL certificates in `traefik.yml`
1. Configure the environment variables

### Deployment user

1. Create `deploy` user with restricted shell (rbash)

   ```bash
   useradd --create-home --shell /bin/rbash deploy
   ```

1. Add authorized ssh key
   1. Create `.ssh` directory

      ```bash
      mkdir /home/deploy/.ssh
      ```

   1. Add public key to `authorized_keys`

      ```bash
      echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCeW1LxwN6nlJre3SkQgpe6e5t1NmjUe7apCSGJKG5xAkhIjSiNv+xt0q5IepTD7mfe6MAkTWx+z8Q0cBJD0SW04Ag2e1+GwUo6lotCavOKHFXkYYHq3FakV/JkXT3yuzROVqGWMZ0D8F0dZTsok66Ptg2ccXUU4y5em+ubxVo2/hzsU6yGFFQl5FMj444JqOqyGdFnB7vU0dbf7XSVbb4j+hoVYO+Y9ompgWOjjPFsXPkqamSgQDUe0x5vMkbNBnr9uGdB6nFZ5f55HIQUs9Qvd6jETzv6MWYuF0E0oHrh1Vy3Z27u8y2eob+o9H7CXUTw/UF8rH3wMadaP4vdCdBOk1Q82eMpZM8BdiiQoX30HU7KJgc/oW+nb9Aa6k3jV1YwnTj6s36nbOTuwuWZ1eu4ru1B6HtRpTGp+p9oL3DUhqs+VGtKU5mu5zCXRtn/xtHqJx2J7+dGKUVlYuHnHHS/f+JckJRbEgN/sK/BHlfgrRiQ+SLa9GhJm3rm2/2NBCM= deploy@openscript" >> /home/deploy/.ssh/authorized_keys
      ```

   1. Set ownership

      ```bash
      chown -R deploy:deploy /home/deploy/.ssh
      ```

   1. Set permission for ssh directory

      ```bash
      chmod 700 /home/deploy/.ssh
      ```

   1. Set permission for `authorized_keys`

      ```bash
      chmod 600 /home/deploy/.ssh/authorized_keys
      ```

1. Allow to update Docker images
   1. Create a `scripts` directory

      ```bash
      mkdir /home/deploy/scripts/
      ```

   1. Add the following script to `/home/deploy/scripts/update-docker.sh`

      ```bash
      #/bin/bash

      cd /srv/lms-services
      docker compose -f docker-compose.yml pull
      docker compose -f docker-compose.yml up -d --remove-orphans
      docker image prune -a -f
      ```

   1. Turn `update-docker.sh` into an executable

      ```bash
      chmod +x /home/deploy/scripts/update-docker.sh
      ```

   1. Create a symbolic link to `sudo`

      ```bash
      ln -s /usr/bin/sudo /home/deploy/scripts/sudo
      ```

   1. Allow deploy user to run `update-docker.sh` with `sudo` by adding to `/etc/sudoers`

      ```txt
      deploy ALL=NOPASSWD: /home/deploy/scripts/update-docker.sh
      ```

   1. Add `scripts` to the deploy users `PATH` by adding the following to `.bashrc`

      ```bash
      readonly PATH=$HOME/scripts
      export PATH
      ```

   1. Change SSH SFTP to `internal-sftp` in `/etc/ssh/sshd_config` by setting `Subsystem	sftp	internal-sftp` and reload with `systemctl reload sshd.service`
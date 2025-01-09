
>**Note:** This was for a specific use case, but these are a more consolidated and straightforward instruction set for deploying a TAK Hardened Server in a Docker rootless configuration. 

# Rootless TAK Server Deployment
## Prepping the box
1. ```sudo apt-get update -y && sudo apt-get upgrade -y```
2. ```sudo apt-get install -y uidmap ```
## Installing the Docker engine
#### Add Docker official GPG key
1. ```sudo apt-get install ca-certificates curl```
2. ```sudo install -m 0755 -d /etc/apt/keyrings```
3. ```sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc```
4. ```sudo chmod a+r /etc/apt/keyrings/docker.asc```
### Add the Docker repo
1.
    ``echo \``
   ``"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \``
   ``$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \``
``sudo tee /etc/apt/sources.list.d/docker.list > /dev/null``
2. ```sudo apt-get update```
### Install Docker packages
1. ```sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin```
### Verify the Docker installation
1. ```sudo docker run hello-world```
## Prep rootless user
### Establish non-root user
1. ```sudo useradd -m -G <dockergroup> <username>```
>**Example:** docker dockeruser
### Verify proper uid/gid
1. ```sudo su <username of docker user>```
2. ```whoami```
3. ```grep ^$(whoami): /etc/subuid```
4. ```grep ^$(whoami): /etc/subgid```
### Ensure the user uid/gids have at least 65,536 sub uid/gids. If needed you can edit the respective file
1. ```vim /etc/subuid```
2. ```vim /etc/subgid```
>**Note:** ```format is <username>:<uid or gid>:<count>```
### Install rootless support packages
1. ```sudo apt-get install -y dbus-user-session```
2. ```sudo apt-get install -y systemd-container```
3. ```sudo apt-get install -y docker-ce-rootless-extras```
### Ensure proper user lingering
1. ```ls /var/lib/systemd/linger```
> **Note:** If the user you created is here. Skip to the next section. If not, continue.
2. ```loginctl enable-linger <username>```
3. ```ls /var/lib/systemd/linger```
### Ensure proper systemd config
1. ```systemctl --user```
> **Note:** If the output has no errors, you can skip to the next section. Else, continue down.
2. ``` Under the username you created add the following to the .bashrc```
``export XDG_RUNTIME_DIR=/run/user/$UID``
3. ```source .bashrc```
## Install Docker rootless
### Disable Docker on other accounts
1. ```sudo systemctl disable --now docker.service docker.socket```
2. ```sudo rm /var/run/docker.sock```
### Under the new non-root user
1. ```dockerd-rootless-setuptool.sh install```
2.  ``` Under the username you created add the following to the .bashrc```
``export PATH=/usr/bin:$PATH``
``export DOCKER_HOST=unix:///run/user/<user id>/docker.sock``
3. ```source .bashrc```
### Verify Docker is running properly
1. ```systemctl --user start docker```
2. ```systemctl --user enable docker```
> **Note:** Under the non-root user
3. ```docker run hello-world```
> **Note:** This should only work under the new rootless user. It should not run on other users if properly installed.
## Hardened TAK Server
### Acquire the zip containing the proper TAK server files from IronBank
```takserver-docker-hardened-<current stable release>.zip```
> **Note:** TAK has tar files as well, zip was just more direct for this deployment
### SCP the zip file to the server
1. ```scp takserver-docker-hardened-<current stable release>.zip <dockeruser>@<IP>:/home/<dockeruser>```
> **Note:** The user doesn't matter ultimately, however, consider ownership of the file. It may need to be changed.
### Once file is on the server
 1. Ensure you are under the correct directory
  ```cd \takserver-docker-hardened-'version'\```
  >**Note:** There should be ``\tak`` and ``\docker`` under the main dir.
### Log into IronBank/Harbor
1.  ```docker login registry1.dso.mil -u <username for IronBank>```
> **Note:** The password is the CLI secret found through the web browser
>**Note:** Before building, edit the ``cert-metadata.sh``. The default password for the CA is ``atakatak`` and should be changed. The file is under ``/tak/certs/``
### Install CA for TAK server
1. ``docker build -t ca-setup-hardened \``
``--build-arg ARG_CA_NAME=<CA_NAME> \``
``--build-arg ARG_STATE=<ST> \``
``--build-arg ARG_CITY=<CITY> \``
``--build-arg ARG_ORGANIZATIONAL_UNIT=<UNIT> \``
``-f docker/Dockerfile.ca .``
2. ```docker run --name ca-setup-hardened -it -d ca-setup-hardened```
3. ``docker cp ca-setup-hardened:/tak/certs/files files``
``[ -d tak/certs/files ] || mkdir tak/certs/files \``
``&& docker cp ca-setup-hardened:/tak/certs/files/takserver.jks tak/certs/files/ \``
``&& docker cp ca-setup-hardened:/tak/certs/files/truststore-root.jks tak/certs/files/ \``
``&& docker cp ca-setup-hardened:/tak/certs/files/fed-truststore.jks tak/certs/files/ \``
``&& docker cp ca-setup-hardened:/tak/certs/files/admin.pem tak/certs/files/ \``
``&& docker cp ca-setup-hardened:/tak/certs/files/config-takserver.cfg tak/certs/files/``
### Installing the TAK DB
> **Note:** Ensure you edit the CoreConfig file under /tak and update the <connection-url> tag with the hardened TAK Database container name and the DB password.
> **Note:** Also, change the keystore and truststore passwords
>
> **Example:**``<connection url="jdbc:postgresql://tak-database-hardened-<version>:5432/cot" username="martiuser" password="<changeme>" />``
>**Note:** Ensure you are in the root TAK directory of the unzipped folder
2. ```docker network create takserver-net-hardened-"$(cat tak/version.txt)"```
3. ```docker network inspect takserver-net-hardened-"$(cat tak/version.txt)"```
4. Take the subnet information and edit the **/tak/db-utils/pg_hba.conf** and ensure it is included.
>**Note:** Again, ensure you are in the root directory of the unzipped folder
5. ``docker build -t tak-database-hardened:"$(cat tak/version.txt)" -f docker/Dockerfile.hardened-takserver-db .``
6. ``docker run \``
``--name tak-database-hardened-"$(cat tak/version.txt)" \``
``--network takserver-net-hardened-"$(cat tak/version.txt)" \``
``--network-alias tak-database \``
``-d tak-database-hardened:"$(cat tak/version.txt)" \``
``-p 5432:5432``
### Install the TAK server
1. ```docker build -t takserver-hardened:"$(cat tak/version.txt)" -f docker/Dockerfile.hardened-takserver .```
2. ``docker run \``
``--name takserver-hardened-"$(cat tak/version.txt)" \``
``--network takserver-net-hardened-"$(cat tak/version.txt)" \``
``-p 8089:8089 -p 8443:8443 -p 8444:8444 -p 8446:8446 \``
``-t -d \``
``takserver-hardened:"$(cat tak/version.txt)"``
### Gather Certs and push admin certs
1. ``docker exec -it ca-setup-hardened bash -c "openssl x509 -noout -fingerprint -md5 -inform pem -in files/admin.pem | grep -oP 'MD5 Fingerprint=\K.*'"``
2. ``docker exec -it takserver-hardened-"$(cat tak/version.txt)" bash -c 'java -jar /opt/tak/utils/UserManager.jar usermod -A -f <admin fingerprint> admin'``

### Connecting to the TAK Server
1. First, you’ll need to pull off the host a few certs, namely, the .p12 files for the admin user you just created and the ``trust store-root.p12``. (The truststore-root will be needed to connect clients i.e., WinTAK, ATAK, etc)
2. Once you have them on your local system, how you import them depends on your specific browser but generally involves uploading them under your security settings.
3. Navigate to ``https://<IP Address>:8443/`` for the dashboard. You’ll see the normal security warnings for using a self-signed cert. Ensure you finish setting set up the server under ``https://<IP Address>:8443/setup``.

### Helpful commands
1. ``docker exec -u root -it <id> sh``
2. ``./makeCert.sh client admin``

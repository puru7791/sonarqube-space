# sonarqube-space
This Repo contains Sonarqube installation and integrated with Azure pipelines

## SonarQube Server installation linux(Ubuntu)
### Prerequisites
* SonarQube server requires at least 2GB of RAM to run efficiently and 1 vCPU cores.
### Platform notes [Official Doc](https://docs.sonarsource.com/sonarqube-server/10.4/requirements/prerequisites-and-overview/#platform-notes)
#### Linux 
> [!IMPORTANT] 
> If you're running on Linux, you must ensure that: 
* ```vm.max_map_count``` is greater than or equal to 524288
* ```fs.file-max``` is greater than or equal to 131072
* the user running SonarQube can open at least 131072 file descriptors
* the user running SonarQube can open at least 8192 threads

You can see the values with the following commands:
```
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```
You can set them dynamically for the current session by running the following commands as ```root```:
```
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```
To set these values more permanently, you must update either ```/etc/sysctl.d/99-sonarqube.conf``` (or ```/etc/sysctl.conf``` as you wish) to reflect these values.

If you are using systemd to start ```SonarQube```, you must specify those limits inside your unit file in the section ```[Service]``` :
```
[Service]
...
LimitNOFILE=131072
LimitNPROC=8192
...
```
### 1. Install JDK
* The SonarQube server requires Java version 17
### 2. Install and Configure PostgreSQL
* Automated repository configuration:
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh 
```
* To manually configure the Apt repository, follow these steps: Look out [Postgress Official Docs](https://www.postgresql.org/download/linux/ubuntu/) for installation guide
```
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
. /etc/os-release
sudo sh -c "echo 'deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $VERSION_CODENAME-pgdg main' > /etc/apt/sources.list.d/pgdg.list"
sudo apt update
sudo apt -y install postgresql
```
* Enable the postgress to start services automatically when the server rebooted
```
$ sudo systemctl enable postgresql
```
* Follow below steps 
* Create an empty schema and a ```sonarqube``` user. Grant this sonarqube user permissions to create, update, and delete objects for this schema.
```
## Change the default PostgreSQL password.
$ sudo passwd postgres

## Switch to the postgres user.
$ su - postgres

## Create a user for sonarqube.
$ createuser sonarqube

## Log in to PostgreSQL
$ psql

## Set a password for the sonarqube user, makesure to set strong passwd
$ ALTER USER sonarqube WITH ENCRYPTED password '<Strong-passwd>';

## Create a sonarqube database and set the owner to sonarqube.
CREATE DATABASE sonarqube OWNER sonarqube;

## Grant all the privileges on the sonarqube database to the sonarqube user
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonarqube;

## Exit PostgreSQL.`\q`

## Return to the non-root sudo user

```
### 3. Installing the SonarQube server from the ZIP file

* Download the SonarQube distribution files from [SonarQube official](https://www.sonarsource.com/products/sonarqube/downloads/)
```
$ sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-<VERSION_NUMBER>.zip
```
* Unzip the downloaded file. Install Unzip utility if dont't have in the distribution ```$ sudo apt-get install zip -y```
* Move the unzipped files to ```/opt/sonarqube``` directory 
### 4. Add SonarQube Group and Use
* SonarQube cannot be run as ```root``` on Unix-based systems, so created a new account dedicated to the purpose of running SonarQube
```
## Create a sonar group.
$ sudo groupadd sonarqube

## Create a sonarqube user and user and set /opt/sonarqube as the home directory.
$ sudo useradd -d /opt/sonarqube -g sonarqube sonarqube

```
* Grant the sonarqube user access to the ```/opt/sonarqube``` directory.
```
$ sudo chown sonarqube:sonarqube /opt/sonarqube -R
```
### 5. Configure SonarQube
* Edit ```<sonarqubeHome>/conf/sonar.properties``` to configure the database settings. 
* Find the following lines and Just uncomment and configure the template you need and comment out the lines dedicated to H2
```
Example for PostgreSQL
sonar.jdbc.username=sonarqube
sonar.jdbc.password=mypassword # 
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```
### 6. Setup Systemd service
#### Running SonarQube manually on Linux:
* Start or stop the instance
```
Start:
$SONARQUBE_HOME/bin/linux-x86-64/sonar.sh start

Graceful shutdown:
$SONARQUBE_HOME/bin/linux-x86-64/sonar.sh stop

Hard stop:
$SONARQUBE_HOME/bin/linux-x86-64/sonar.sh force-stop
```
#### Running SonarQube as a service on Linux with SystemD
* Create the file ```/etc/systemd/system/sonarqube.service``` based on the following:
* You can get the correct pathes to the executables using the ```which``` command as in ```which nohup``` and ```which java```
```
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=simple
User=sonarqube
Group=sonarqube
PermissionsStartOnly=true
ExecStart=/bin/nohup /opt/java/bin/java -Xms32m -Xmx32m -Djava.net.preferIPv4Stack=true -jar /opt/sonarqube/lib/sonar-application-<YOUR SONARQUBE VERSION>.jar
StandardOutput=syslog
LimitNOFILE=131072
LimitNPROC=8192
TimeoutStartSec=5
Restart=always
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```
> [!NOTE]
> Because the sonar-application jar name ends with the version of SonarQube, you will need to adjust the ```ExecStart``` command accordingly on install and at each upgrade.
> All SonarQube directories should be owned by the ```sonarqube``` user.
> If you have multiple Java versions, you will need to modify the ```java``` path in the ```ExecStart``` command. This also means ```SONAR_JAVA_PATH``` will not work with SonarQube as a service.
> If you wish to do any modification of a service file, make sure to run ```sudo systemctl daemon-reload``` after that.

* Once your sonarqube.service file is created and properly configured, run:
```
sudo systemctl enable sonarqube.service
sudo systemctl start sonarqube.service
```

* You can now browse SonarQube at ```http://<your-server-publicIP>:9000```(the default system administrator credentials are ```admin```/```admin```)

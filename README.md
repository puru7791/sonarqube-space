# sonarqube-space
This Repo contains Sonarqube installation and integrated with Azure pipelines

## SonarQube Server installation linux(Ubuntu)
### Prerequisites
* SonarQube server requires at least 2GB of RAM to run efficiently and 1 vCPU cores.
### 1. Install JDK
* The SonarQube server requires Java version 17
### 2. Install and Configure PostgreSQL
* Automated repository configuration:
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh 
```
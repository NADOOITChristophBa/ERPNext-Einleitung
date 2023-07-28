<center>
 <img src="ERPNext-servidor-cloud.jpg">
</center>

# So installieren Sie ERPNext in Ubuntu 20.04

## A Step by Step Guide

Die Installation von ERPNext kann nervig sein, besonders wenn Sie gerade erst anfangen. In diesem Artikel werde ich Schritt für Schritt vorgehen, um unser neu installiertes Betriebssystem Ubuntu 20.04 zu konfigurieren, um eine Umgebung einzurichten und ERPNext zu installieren.

## Voraussetzungen

### Software Anforderungen

* Updated Ubuntu 20.04
* A user with sudo privileges
* Python 3.6+
* Node.js 12
* Redis 5
* MariaDB 10.3.x / Postgres 9.5.x
* yarn 1.12+
* pip 20+
* wkhtmltopdf (version 0.12.5 with patched qt)
* cron
* NGINX

### Hardware Anforderungen

* 4GB RAM
* 40GB Hard Disk

In unseren ersten Schritten stellen wir sicher, dass unser System auf dem neuesten Stand ist, indem wir die folgenden Befehle ausführen:

```bash
sudo apt update
sudo apt -y upgrade
```

Es wird empfohlen, Ihr System bei jedem Upgrade neu zu starten:

```bash
sudo reboot
```

#### Schritt 1: Installieren Sie Python Tools & wkhtmltopdf

```bash
sudo apt -y install vim libffi-dev python3-pip python3-dev python3-testresources libssl-dev wkhtmltopdf python3.10-venv
```

#### Schritt 2: Installieren Sie Curl, Redis und Node.js

```bash
sudo apt install curl

sudo curl --silent --location https://deb.nodesource.com/setup_14.x | sudo bash -

curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --dearmor | sudo tee /usr/share/keyrings/yarnkey.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/yarnkey.gpg] https://dl.yarnpkg.com/debian stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

sudo apt-get update && sudo apt-get install yarn

sudo apt -y install gcc g++ make nodejs redis-server
```

#### Schritt 3: Installieren Sie den Nginx-Webserver und den MariaDB-Datenbankserver

```bash
sudo apt -y install nginx
sudo apt install mariadb-server
```

Authentifizierungs-Plugin ändern.

```bash
sudo mysql -u root
```

```sql
USE mysql;
UPDATE user SET plugin='mysql_native_password' WHERE User='root';
UPDATE user SET authentication_string=password('your_password') WHERE user='root';
FLUSH PRIVILEGES;
EXIT;
```

Sollte während der Installation folgender Fehler auftreten:

ERROR 1396 (HY000): Operation ALTER USER failed for 'root'@'localhost'

Dann handelt es sich um einen Fehler, der durch eine Änderung in der Nutzerverwaltung von MariaDB Version 10.4 und höher verursacht wird. Hierbei ist die mysql.user Tabelle nun eine Ansicht (View), und die tatsächlichen Benutzerdaten sind in der global_priv Tabelle innerhalb der mysql Datenbank gespeichert.

Um diesen Fehler zu beheben, folge bitte diesen Schritten:

Starte das MariaDB Interface mit dem Befehl:

```bash
sudo mysql -u root
```

Im MariaDB Interface, führe folgende Befehle aus:

```sql
USE mysql;
UPDATE mysql.global_priv SET priv=json_set(priv, '$.authentication_string', PASSWORD('Ihr_Neues_Passwort'), '$.plugin', 'mysql_native_password') WHERE User='root' AND Host='localhost';
FLUSH PRIVILEGES;
EXIT;
```

Ersetze 'Ihr_Neues_Passwort' mit dem neuen Passwort für den Root-Benutzer. Dieser Befehl aktualisiert sowohl die Authentifizierungsmethode als auch das Passwort für den Root-Benutzer in der mysql.global_priv Tabelle direkt.

Stellen Sie sicher, dass Sie die folgenden Einstellungen für mysqld und den mysql-Client wie angegeben haben. Ich habe eine Datei in einem [Github-Repo](https://github.com/SafdariAlireza/ERPNext_mariadb_conf) abgelegt, damit Sie die gesamte Datei kopieren und ersetzen können.

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb
```

#### Schritt 4: Installieren Sie Bench und ERPNext

Eine Bench ist ein Tool zum Installieren und Verwalten von ERPNext auf Ihrem Ubuntu-System. Wir erstellen einen Benutzer, der das ERPNext-System ausführt, und konfigurieren dann das System.

```bash
sudo useradd -m -s /bin/bash erpnext
sudo passwd erpnext
sudo usermod -aG sudo erpnext
```

Jetzt ist es an der Zeit, Ihren PATH zu aktualisieren.

```bash
sudo su - erpnext
tee -a ~/.bashrc<<EOF
PATH=\$PATH:~/.local/bin/
EOF
source ~/.bashrc
```

Als nächstes müssen Sie ein Verzeichnis für das ERPNext-Setup erstellen und erpnext-Benutzern Lese- und Schreibberechtigungen für das Verzeichnis erteilen:

```bash
sudo mkdir /opt/bench
sudo chown -R erpnext /opt/bench
```

Wechseln Sie als Nächstes zum erpnext-Benutzer und installieren Sie die Anwendung:

```bash
cd /opt/bench
```

Frappe-bench und Git installieren

```bash

sudo apt install git

sudo pip3 install frappe-bench
```

Der nächste Schritt besteht darin, das Bench-Verzeichnis mit installiertem Frappe-Framework zu initialisieren. Stellen Sie sicher, dass Sie sich noch im Verzeichnis /opt/bench befinden:

```bash
bench init --frappe-branch version-14 erpnext
```

Erstellen Sie eine neue Frappe-Site.

```bash
cd erpnext
bench new-site erp.testSite.com 
```

#### Schritt 5: Holen Sie sich die ERPNext-Anwendung von GitHub

Laden Sie die ERPNext aus dem Frappe-Github-Repo herunter. Wir werden Version 13 verwenden. Sie können jede Version verwenden, die Sie möchten.

```bash
bench get-app --branch version-13 erpnext
```

#### Schritt 6: Installieren Sie die ERPNext-App auf unserer Website

```bash
bench --site erp.testSite.com install-app erpnext
```

#### Schritt 7: Starten Sie ERPNext und schließen Sie die Installation ab

```bash
bench use erp.testSite.com
bench start
```

Navigieren Sie zur IP-Adresse Ihrer Installation und der Portnummer, die nach dem Ausführen auf dem Terminal angezeigt wird. Verwenden Sie im Fall einer lokalen Instanz 127.0.0.1:8000

Um vom Dev-Mode zu Production-Mode zu wechseln, führen Sie folgende Befehl aus:

```bash
sudo bench setup production $USER
sudo supervisorctl restart all
```

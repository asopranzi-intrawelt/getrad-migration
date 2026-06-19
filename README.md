# Stack getrad - Documentazione operativa

Documento di riferimento per l'amministrazione dello stack containerizzato getrad (gestionale Intrawelt) ospitato sulla VM810 (Ubuntu 24.04 LTS, IP 192.168.20.90).

## Architettura

Lo stack gira interamente come Docker compose sulla VM810. Due container:

- `getrad-db` — MariaDB 10.3.39 (immagine `mariadb:10.3`)
  - Datadir bind mount: `/srv/getrad-stack/data/mariadb`
  - Config aggiuntive: `/srv/getrad-stack/build/db-conf/`
  - Porta interna 3306, NON esposta sull'host

- `getrad-app` — Tomcat 9 + OpenJDK 8 (immagine custom basata su `tomcat:9-jdk8`)
  - Webapp getrad bind mount: `/srv/getrad-stack/extracted/getrad`
  - Solr home bind mount: `/srv/getrad-stack/extracted/solr`
  - Build context: `/srv/getrad-stack/build/app/`
  - Porta 8080 esposta su 192.168.20.90:8080

Rete: bridge Docker `getrad-stack_getrad-net`. Da `app` il DB si raggiunge come hostname `db:3306`.

URL applicativa: `http://192.168.20.90:8080/getrad/`
URL login: `http://192.168.20.90:8080/getrad/jsp/Login.jsp`

## Avvio, arresto, restart dello stack

```bash
cd /srv/getrad-stack

# Avvio (parte automaticamente al boot via restart: unless-stopped)
docker compose up -d

# Arresto pulito
docker compose down

# Restart solo app (senza toccare il DB)
docker compose restart app

# Stato
docker compose ps

# Log live
docker compose logs -f app
docker compose logs -f db
```

Lo stack si avvia automaticamente al boot della VM810. Healthcheck attivi: db è considerato pronto quando `mysqladmin ping` risponde, app quando `/getrad/jsp/Login.jsp` ritorna HTTP 200.

## Backup

### Schedule automatico

Cron in `/etc/cron.d/getrad-backup`:

- Settimanale: ogni lunedì 02:00, retention 30 giorni
- Mensile: primo del mese 03:00, retention 190 giorni

Script: `/usr/local/sbin/getrad-backup.sh`
Output: `/srv/getrad-stack/backups/`

### Backup manuale on-demand

```bash
# Settimanale (sostituisce quello del lunedì se eseguito oggi)
sudo /usr/local/sbin/getrad-backup.sh weekly

# Mensile
sudo /usr/local/sbin/getrad-backup.sh monthly
```

Ogni backup produce 3 file zstd-compressi più un SHA256SUMS:

- `getrad-db-{mode}-YYYY-MM-DD.sql.zst` — dump SQL del DB
- `getrad-conf-{mode}-YYYY-MM-DD.tar.zst` — config applicativa (WEB-INF/conf)
- `getrad-stack-config-{mode}-YYYY-MM-DD.tar.zst` — config infrastrutturale (compose, Dockerfile)

### Verifica integrità

```bash
cd /srv/getrad-stack/backups
sha256sum -c SHA256SUMS-weekly-YYYY-MM-DD
```

### Log dei backup

Tracciamento eventi: `/srv/getrad-stack/backups/backup.log`

## Restore

### Restore completo del DB da backup

```bash
# 1. Stop app per liberare connessioni JDBC
docker compose stop app

# 2. Decomprimi e applica il dump
zstd -d -c /srv/getrad-stack/backups/getrad-db-weekly-YYYY-MM-DD.sql.zst \
  | docker exec -i getrad-db mysql -u root getrad

# 3. Restart app
docker compose start app
```

### Restore di una singola tabella

```bash
zstd -d -c /srv/getrad-stack/backups/getrad-db-weekly-YYYY-MM-DD.sql.zst \
  | grep -v '^DROP TABLE' \
  | sed -n '/-- Table structure for table `nome_tabella`/,/-- Table structure for table /p' \
  | docker exec -i getrad-db mysql -u root getrad
```

### Restore della configurazione applicativa

```bash
# Backup current (paranoia)
sudo tar -czf /tmp/conf-pre-restore.tgz -C /srv/getrad-stack/extracted/getrad/WEB-INF conf

# Restore
zstd -d -c /srv/getrad-stack/backups/getrad-conf-weekly-YYYY-MM-DD.tar.zst \
  | sudo tar --numeric-owner -xf - -C /srv/getrad-stack/extracted/getrad/WEB-INF/

# Restart app per ricaricare la nuova config
docker compose restart app
```

### Disaster recovery completo da zero

In caso la VM810 sia da ricreare:

1. Installa Ubuntu 24.04 LTS Server minimal
2. Installa Docker engine e compose plugin (vedi sezione Setup iniziale più sotto)
3. Crea utente intrawelt e aggiungilo al gruppo docker
4. Ricostruisci la struttura `/srv/getrad-stack/`:
   `mkdir -p /srv/getrad-stack/{build/app,build/db-conf,data/mariadb,extracted,backups}`
5. Recupera l'ultimo backup-stack-config disponibile e estrai (contiene compose, Dockerfile, config db)
6. Recupera l'ultimo backup-db disponibile, decomprimi, importa nel container db appena creato
7. Recupera l'albero `/usr/skn/getrad` e `/usr/skn/solr` originali (artefatti estratti dalla VM809; conservati su backup vzdump Proxmox o backup separato)
8. `docker compose up -d`

## Modifiche di configurazione comuni

### Cambio password utente DB getrad

```bash
docker exec -i getrad-db mysql -u root <<SQL
ALTER USER 'getrad'@'%' IDENTIFIED BY 'NUOVA_PASSWORD';
ALTER USER 'getrad'@'localhost' IDENTIFIED BY 'NUOVA_PASSWORD';
FLUSH PRIVILEGES;
SQL

# Aggiorna anche getrad.properties dell'app
sudo sed -i 's|util.ConnectionManager.DbPassword=.*|util.ConnectionManager.DbPassword=NUOVA_PASSWORD|' \
  /srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties

docker compose restart app
```

### Modifica file di configurazione applicativa

I file `getrad.properties` e altri sotto `WEB-INF/conf/` vivono nel bind mount. Modifiche dirette al file sono visibili dal container. L'app NON ricarica config a runtime: serve restart.

```bash
# Backup pre-modifica
cp /srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties{,.bak.$(date +%F)}

# Modifica
nano /srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties

# Restart
docker compose restart app
```

### Aggiunta di un nuovo schema/utente DB

```bash
docker exec -i getrad-db mysql -u root <<SQL
CREATE DATABASE nuovo_schema;
CREATE USER 'nuovo_user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nuovo_schema.* TO 'nuovo_user'@'%';
FLUSH PRIVILEGES;
SQL
```

## Troubleshooting comune

### App non risponde su 8080

```bash
docker compose ps  # entrambi (healthy)?
curl -sI http://127.0.0.1:8080/getrad/  # risponde?
docker compose logs --tail=50 app  # errori recenti?
```

Se app non parte: probabile problema di connettività verso db. Verifica:

```bash
docker exec getrad-app sh -c 'getent hosts db && nc -zv db 3306'
```

### DB non parte

```bash
docker compose logs --tail=50 db
# Cerca: "InnoDB" errors, "Plugin" errors, "Aborting startup"
```

Se "MariaDB upgrade required": il datadir è di una versione minor diversa dall'immagine. Pinna l'immagine al tag corrente (`mariadb:10.3.39` invece di `mariadb:10.3`) nel docker-compose.yml.

### Tomcat ci mette eternità ad avviarsi

Lo startup dovrebbe essere ~10 secondi grazie a `catalina-skip.properties` (jarsToSkip). Se ci mette molto, verifica che il file sia presente nell'immagine costruita:

```bash
docker exec getrad-app grep jarsToSkip /usr/local/tomcat/conf/catalina.properties | head -3
```

Se manca, ricostruisci:

```bash
cd /srv/getrad-stack
docker compose build --no-cache app
docker compose up -d --force-recreate app
```

### Errore Connection refused da Solr DataImporter

Il file `data-config.xml` di Solr deve avere il dataSource che punta a `db` non a `localhost`:

```bash
grep 'name="prod"' /srv/getrad-stack/extracted/solr/conf/data-config.xml
# Atteso: url="jdbc:mysql://db/getrad"
```

### App lenta su pagine specifiche

Il logger MAIL dell'app deve essere disabilitato in getrad.properties (riga 2):

```
log4j.logger.ApplicationLogger=INFO, APP
```

NON `INFO, APP, MAIL`. Senza questa modifica l'app prova a inviare mail SMTP a smtp.office365.com che non è raggiungibile dal container, generando timeout visibili come "Please wait" multipli secondi.

### Spazio disco esaurito

Posti dove si accumula spazio:

- `/srv/getrad-stack/extracted/getrad/WEB-INF/logs/` — gestiti da logrotate
- `/srv/getrad-stack/backups/` — retention 30/190 giorni
- `/var/lib/docker/` — immagini e container vecchi: `docker system prune -af`

## Manutenzione periodica

### Settimanale

- Verifica esecuzione backup: `tail /srv/getrad-stack/backups/backup.log`
- Verifica integrità stack: `docker compose ps` (entrambi healthy)
- Verifica spazio: `df -h /srv`

### Mensile

- Verifica unattended-upgrades: `cat /var/log/unattended-upgrades/unattended-upgrades.log | tail -20`
- Verifica backup mensile presente: `ls -lh /srv/getrad-stack/backups/getrad-db-monthly-*.sql.zst`
- Cleanup immagini Docker non usate: `docker system prune -af` (chiede conferma)

### Ricostruzione immagine app dopo modifica Dockerfile

```bash
cd /srv/getrad-stack
docker compose build --no-cache app
docker compose up -d --force-recreate app
```

### Aggiornamento sistema operativo (apt upgrade)

Mensile o quando esce un security patch significativo:

```bash
sudo apt update
sudo apt upgrade --without-new-pkgs -y
# Se richiede reboot: pianifica fuori orario
ls /var/run/reboot-required && echo "REBOOT NEEDED"
```

## Riferimenti

### File chiave

- Compose: `/srv/getrad-stack/docker-compose.yml`
- Dockerfile app: `/srv/getrad-stack/build/app/Dockerfile`
- Catalina jarsToSkip: `/srv/getrad-stack/build/app/catalina-skip.properties`
- Context fragment getrad: `/srv/getrad-stack/build/app/getrad.xml`
- Plugin auth socket DB: `/srv/getrad-stack/build/db-conf/load-unix-socket.cnf`
- Config app: `/srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties`
- Config Solr: `/srv/getrad-stack/extracted/solr/conf/data-config.xml`
- Script backup: `/usr/local/sbin/getrad-backup.sh`
- Cron backup: `/etc/cron.d/getrad-backup`
- Logrotate: `/etc/logrotate.d/getrad`

### Credenziali (sviluppo, da rotare in produzione)

- DB user: `getrad` / password: `getradpwd`
- DB schema: `getrad`
- DB root: socket auth (no password), accesso solo via `docker exec`

### Origine artefatti

Lo stack è stato costruito a maggio 2026 estraendo gli artefatti dalla VM809 (Ubuntu 20.04 LTS, layout Tomcat 9 deb). La VM809 resta come archivio storico spento; va mantenuta finché la fase 5 (patching applicativo) non è conclusa.

### Versioni runtime fissate

- Ubuntu host: 24.04 LTS (noble)
- Docker engine: 29.4.x
- MariaDB: 10.3.39 (image tag `mariadb:10.3`)
- Tomcat: 9.x (image tag `tomcat:9-jdk8`)
- OpenJDK: 8u (dentro tomcat:9-jdk8)
- mysql-connector-java: 3.0.11 (dentro WEB-INF/lib di getrad)
- Solr: 3.5.0 (solr.war di gennaio 2012)

---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - README.md
  - HANDOFF_getrad.md
  - GETRAD_TOMCAT_MIGRATION.md
last-verified-commit: 4f686bf
---

# Stack applicativo

## Stack e runtime

Applicazione *getrad*: gestionale per agenzia di traduzioni (clienti, fornitori, preventivi,
ordini, fatture, provvigioni, riepiloghi). Codice JSP/Servlet Java con tag library custom
`skn:*` (es. `skn:bottone`, `skn:lista`). Origine: Tomcat 6 su Ubuntu 10.04 (circa 2010),
migrata a Tomcat 9 + Java 8 su Ubuntu 20.04, poi containerizzata su Ubuntu 24.04.

Runtime fissati:

- Java: OpenJDK 8u (dentro l'immagine `tomcat:9-jdk8`)
- Tomcat: 9.x (image tag `tomcat:9-jdk8`)
- MariaDB: 10.3.39 (image tag `mariadb:10.3`)
- Solr: 3.5.0 (`solr.war` del gennaio 2012, webapp dentro lo stesso Tomcat di getrad)
- JDBC driver: mysql-connector-java 3.0.11 (in `WEB-INF/lib`, versione legacy che richiede MariaDB ≤10.x)
- Docker engine: 29.4.x
- Docker Compose: v5.1.3
- Host OS: Ubuntu 24.04 LTS (noble), VM810 su Proxmox (192.168.20.90)

## Alternative deliberatamente escluse

Tomcat bare-metal su Ubuntu 24.04: scartato perché Ubuntu 24.04 non pacchettizza più OpenJDK 8;
mantenere Java 8 come requisito dell'app avrebbe richiesto gestione manuale del JDK. Docker
risolve l'isolamento del runtime senza toccare il sistema host.

Upgrade MariaDB a 10.4+ o MySQL 8: scartato perché il JDBC driver 3.0.11 non è compatibile con
le API di autenticazione introdotte nelle versioni successive. Il driver è embedded nell'app e
non si aggiorna senza accesso al sorgente.

Upgrade Solr a versione recente: scartato perché richiederebbe migrazione dello schema degli
indici e modifica delle API di interrogazione nell'app. Solr 3.5 funziona ed è usato solo
internamente.

Migrazione da JSP a framework moderno: fuori scope. L'obiettivo è la continuità operativa,
non il refactoring applicativo.

## Flussi di codice e ruolo architetturale dei file

Il codice applicativo è interamente in `/srv/getrad-stack/extracted/getrad/` su VM810 (bind mount
sul container `getrad-app` in `/usr/local/tomcat/webapps/getrad/`). Struttura rilevante:

```
WEB-INF/
  conf/getrad.properties    # configurazione DB, mailer, path interni (gitignored, contiene segreti)
  lib/                      # JAR di libreria, incluso mysql-connector-java 3.0.11
  logs/                     # Application.log (rotazione log4j), Accesses.csv, ConnectionManager.log
jsp/                        # sorgenti JSP organizzati per modulo
  _include/                 # template globali (templateBottom*.jspf)
  clienti/
  fornitori/
  preventivi/
  ordini/
  fatture/
  traduttori/
  provvigioni/
  riepiloghi/
  lead/
  societa/
  vipa/
  search/
```

Compilazione JSP: Jasper (il compilatore JSP di Tomcat 9) compila i `.jsp`/`.jspf` on-demand al
primo accesso e mette in cache le servlet generate in
`/usr/local/tomcat/work/Catalina/localhost/getrad/` dentro il container. Per forzare la
ricompilazione cancellare quella cartella e riavviare il container app.

Solr: `solr.war` deployato come webapp standard Tomcat in `webapps/solr/`. La home Solr è in
`/srv/getrad-stack/extracted/solr/` (bind mount in `/usr/local/tomcat/solr/`). Connessione al
DB per il DataImporter: `jdbc:mysql://db/getrad` (hostname docker interno).

## Riferimenti a snippet

- Dockerfile app: `/srv/getrad-stack/build/app/Dockerfile`
- Docker compose: `/srv/getrad-stack/docker-compose.yml`
- Config app: `/srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties`
- Config Solr DataImporter: `/srv/getrad-stack/extracted/solr/conf/data-config.xml`
- Script backup: `/usr/local/sbin/getrad-backup.sh`

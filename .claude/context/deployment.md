---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - README.md
last-verified-commit: 4f686bf
---

# Deployment

## Livelli

Un solo livello: produzione. Nessun staging. L'istanza gira su VM810 (IP 192.168.20.90),
accessibile solo dalla LAN aziendale sulla porta 8080. Nessun HTTPS: l'accesso è interno e
non è mai stato configurato un reverse proxy o un tunnel TLS.

URL produzione: `http://192.168.20.90:8080/getrad/`
URL login: `http://192.168.20.90:8080/getrad/jsp/Login.jsp`
URL Solr: `http://192.168.20.90:8080/solr/`

## Comandi

Tutti i comandi si eseguono su VM810 come utente `intrawelt` dalla directory
`/srv/getrad-stack/`:

```bash
# Avvio (dopo boot la macchina parte automaticamente via restart: unless-stopped)
docker compose up -d

# Arresto
docker compose down

# Restart solo app (senza toccare DB)
docker compose restart app

# Stato + healthcheck
docker compose ps

# Log in tempo reale
docker compose logs -f app
docker compose logs -f db

# Ricostruzione immagine app (dopo modifica Dockerfile o catalina-skip.properties)
docker compose build --no-cache app
docker compose up -d --force-recreate app

# Forzare ricompilazione Jasper dopo modifica JSP (non richiede ricostruzione immagine)
docker exec getrad-app rm -rf /usr/local/tomcat/work/Catalina/localhost/getrad
docker compose restart app
```

Backup ad-hoc prima di interventi rischiosi (intero albero webapp + DB):

```bash
TAG=pre-intervento-$(date +%Y-%m-%d)
BACKUP_DIR=/srv/getrad-stack/backups
docker exec getrad-db mysqldump --single-transaction --routines --triggers --events getrad \
  | zstd -T0 -3 -o ${BACKUP_DIR}/getrad-db-${TAG}.sql.zst
sudo tar --numeric-owner -cf - -C /srv/getrad-stack/extracted getrad \
  | zstd -T0 -3 -o ${BACKUP_DIR}/getrad-fullapp-${TAG}.tar.zst
cd ${BACKUP_DIR} && sha256sum getrad-*-${TAG}* | sudo tee SHA256SUMS-${TAG}
```

## Variabili d'ambiente e segreti

Nessuna variabile d'ambiente nel compose. Tutti i segreti vivono nel file di configurazione
dell'app, letto all'avvio e non ricaricato a runtime:

`/srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties`

Contiene: URL JDBC con password DB, credenziali SMTP Office365, eventuali token applicativi.
Il file è gitignored; non va mai versionato. Il riferimento in repo è
`getrad.properties.example` (segreti sostituiti con `__SET_ME__`).

Segreto con priorità di rotazione: la password SMTP Office365 presente nel file è riutilizzata
su altri sistemi e va ruotata in produzione, indipendentemente dallo stato del repository.

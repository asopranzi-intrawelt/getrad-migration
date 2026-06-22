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

Due livelli, entrambi sullo stesso host VM810 (IP 192.168.20.90) e raggiungibili dalla LAN
aziendale, distinti per progetto compose e porta pubblicata. Nessun HTTPS su nessuno dei due:
l'accesso è interno e non è mai stato configurato un reverse proxy o un tunnel TLS. La
produzione è il progetto compose `getrad-stack` radicato in `/srv/getrad-stack/`, sulla porta
8080, per l'uso normale degli utenti. Il test è il progetto compose `getrad-test` radicato in
`/srv/getrad-stack/test/`, sulla porta 8090, per verificare le modifiche prima di promuoverle.
L'ambiente di test è additivo e isolato: rete, datadir DB, copia del codice e dell'indice Solr
propri, allegati `files/` montati dalla produzione in sola lettura, mailer disattivato. Vedi
ADR-007, ADR-008 e ADR-009 in `memory/decisions.md`.

URL produzione: `http://192.168.20.90:8080/getrad/`
URL login produzione: `http://192.168.20.90:8080/getrad/jsp/Login.jsp`
URL Solr produzione: `http://192.168.20.90:8080/solr/`
URL test: `http://192.168.20.90:8090/getrad/`
URL login test: `http://192.168.20.90:8090/getrad/jsp/Login.jsp`

Nota: UFW risulta `inattivo` sull'host, quindi entrambe le porte sono raggiungibili sulla LAN
senza regole esplicite. Il binding dei container è su `0.0.0.0`. Il context fragment Tomcat
espone l'app con docBase `/usr/skn/getrad` (bind mount), non sotto `webapps/`.

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

I comandi dell'ambiente di test sono gli stessi, riferiti al compose di test con `-f`:

```bash
# Avvio / arresto / stato del solo ambiente di test
docker compose -f /srv/getrad-stack/test/docker-compose.yml up -d
docker compose -f /srv/getrad-stack/test/docker-compose.yml down
docker compose -f /srv/getrad-stack/test/docker-compose.yml ps

# Ricompilazione Jasper su test dopo modifica JSP (anche: test/recompile-test.sh)
docker exec getrad-test-app rm -rf /usr/local/tomcat/work/Catalina/localhost/getrad
docker compose -f /srv/getrad-stack/test/docker-compose.yml restart app

# Riseminare il DB di test da un dump di produzione (ripartenza da dati freschi)
docker compose -f /srv/getrad-stack/test/docker-compose.yml down
sudo rm -rf /srv/getrad-stack/test/data/mariadb && mkdir -p /srv/getrad-stack/test/data/mariadb
docker compose -f /srv/getrad-stack/test/docker-compose.yml up -d db
# attendere lo stato healthy, poi:
zstd -dc /srv/getrad-stack/backups/getrad-db-<TAG>.sql.zst \
  | docker exec -i getrad-test-db mysql -u root -pgetradpwd getrad
docker compose -f /srv/getrad-stack/test/docker-compose.yml up -d app
```

Promozione di file validati da test a produzione, con backup automatico della versione di
produzione corrente e ricompilazione Jasper su prod:

```bash
sudo /srv/getrad-stack/test/promote-to-prod.sh jsp/fornitori/insupdfornitore.jsp
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

## Firewall e allowlist di rete

Dal 2026-06-19 l'accesso al gestionale è ristretto ai soli IP statici autorizzati della LAN. Il
filtro è applicato nella catena iptables `DOCKER-USER` e non con UFW, perché Docker pubblica le
porte manipolando direttamente iptables e bypassa UFW (vedi ADR-010). Il filtro distingue
produzione e test sulla porta di destinazione originale prima del DNAT, tramite il modulo conntrack,
dato che entrambi i container ascoltano internamente sulla 8080.

La produzione, sia sulla porta 80 (reverse proxy con nome amichevole) sia sulla 8080 (accesso
diretto a Tomcat), è consentita a sei postazioni: 192.168.10.80 (PC-SONIA), 192.168.10.81
(PC-FABIO), 192.168.10.206 (PC-ELISA), 192.168.10.73 (PC-ALESSIO, Sopranzi), e dal 2026-06-22 anche
192.168.10.75 (PC-ALESSIA-NAS, Alessia Nasini) e 192.168.10.76 (PC-ALESSANDRO, Alessandro
Potalivo). Il test sulla porta 8090 è consentito alla sola 192.168.10.73. Ogni altra sorgente
verso 80, 8080 e 8090 viene scartata. La porta SSH dell'host non è interessata, perché non è una
porta pubblicata da container e non transita per `DOCKER-USER`.

Le regole vivono nello script idempotente `/srv/getrad-stack/firewall/getrad-firewall.sh`, applicato
al boot dopo Docker dall'unità `getrad-firewall.service` (abilitata). Lo stato iptables precedente è
salvato in `/srv/getrad-stack/firewall/iptables-backup-2026-06-19.rules` per rollback. Per
modificare gli IP si edita lo script e si rilancia con `systemctl restart getrad-firewall.service`.
La verifica empirica del blocco va fatta da una postazione non autorizzata, controllando che
`http://192.168.20.90:8080/getrad/` non risponda.

## Nome amichevole egetrad e reverse proxy

Dal 2026-06-19 gli utenti raggiungono la produzione con un nome amichevole sulla porta 80, invece
di digitare IP e porta 8080. Un reverse proxy nginx, progetto compose separato e additivo in
`/srv/getrad-stack/proxy/` (container `getrad-proxy`, agganciato come rete esterna alla rete della
produzione `getrad-stack_getrad-net`), ascolta sulla porta 80 e proxa verso `getrad-app:8080`. La
radice `/` e la rotta `/egetrad-login` reindirizzano (302) alla pagina di login
`/getrad/jsp/Login.jsp`; tutto il resto, incluso `/getrad/` e `/solr/`, è proxato in modo
trasparente. La configurazione è in `/srv/getrad-stack/proxy/nginx.conf`. Il compose di produzione
non è stato toccato, quindi il proxy si rimuove con `docker compose -f /srv/getrad-stack/proxy/docker-compose.yml down`.

Lato client la risoluzione del nome avviene su ogni PC Windows autorizzato aggiungendo al file
`C:\Windows\System32\drivers\etc\hosts` tre voci che mappano `egetrad`, `egetrad-login` ed
`egetrad.intrawelt.com` su 192.168.20.90. Non esiste un DNS di LAN: `egetrad.intrawelt.com` su DNS
pubblico risolve a un IP esterno non collegato a questo server, quindi mapparlo nel file hosts
verso la LAN serve a un doppio scopo, far funzionare i link legacy interni dell'app e impedire che
quelle postazioni raggiungano per quel nome il sito pubblico attuale.

I link assoluti che l'applicazione genera verso `http://egetrad.intrawelt.com:8080/...` (conferme,
preventivo rapido, alcune aree) sono cosi' coperti: risolvono al server di LAN sulla 8080, che e'
gia' nell'allowlist. La navigazione principale usa percorsi assoluti `/getrad/...` relativi all'host
e funziona sulla porta 80 attraverso il proxy. Il `server_name` di nginx include anche
`egetrad.intrawelt.com` oltre a `egetrad` ed `egetrad-login`.

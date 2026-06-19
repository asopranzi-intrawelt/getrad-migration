# Handoff progetto getrad

Documento di passaggio di consegne autoportante. Una nuova istanza Claude Code può continuare
il lavoro da qui senza ricostruire il contesto da altri documenti. Aggiornato il 2026-06-19.

## Sommario esecutivo

Migrazione del gestionale legacy getrad (agenzia traduzioni Intrawelt) da Ubuntu 10.04 →
Ubuntu 20.04 → Ubuntu 24.04. La fase 4 estesa (containerizzazione su Ubuntu 24.04, che non
supporta più OpenJDK 8 bare-metal) è chiusa: l'app gira stabilmente su Docker compose sulla
VM810. La fase 5 (patching meccanico dei file JSP per compatibilità Tomcat 9) è in corso: 11
file fixati, 8 file con errori noti pending, ~24 file mai toccati. Prossimo passo immediato:
`fornitori/insupdfornitore.jsp`.

## Architettura attuale

Host fisico: nodo Proxmox `pve` (192.168.20.11), storage LVM-thin `SERVIZI` (1 TiB, 376 GiB
liberi).

VM810 Ubuntu 24.04 LTS minimal:

- IP: 192.168.20.90 (statico)
- Disco: 250 GiB LVM-thin, 110 GiB usati
- Utente: `intrawelt` (gruppo `docker`)
- Stack di runtime: Docker engine 29.4.x + Compose plugin v5.1.3 (repo ufficiale
  download.docker.com)
- Firewall: UFW attivo, permessi solo 22/tcp (SSH) e 8080/tcp (app)
- Patching automatico: unattended-upgrades, security only
- NetworkManager mantenuto (decisione consapevole, non sostituito con systemd-networkd)

Stack applicativo (`/srv/getrad-stack/`):

```
/srv/getrad-stack/
├── docker-compose.yml              # 2 servizi: db + app
├── README.md                       # documentazione operativa dello stack
├── build/
│   ├── app/                        # Dockerfile + solr.war + getrad.xml + catalina-skip.properties
│   └── db-conf/                    # load-unix-socket.cnf (plugin auth_socket)
├── data/
│   └── mariadb/                    # datadir MariaDB 10.3.39 (UID:GID 999:999)
├── extracted/
│   ├── getrad/                     # webapp getrad (bind mount → /usr/local/tomcat/webapps/getrad)
│   └── solr/                       # home Solr 3.5 + solr.war (bind mount → /usr/local/tomcat/solr)
└── backups/                        # backup schedulati + backup ad-hoc + log
```

Container in esecuzione:

- `getrad-db` — mariadb:10.3 (3306 interna, NON esposta sull'host)
- `getrad-app` — tomcat:9-jdk8 custom (8080 esposta su 0.0.0.0:8080)
- Rete bridge: `getrad-stack_getrad-net`, risoluzione DNS interna (`db` raggiungibile come
  `db:3306`)

URL principale: `http://192.168.20.90:8080/getrad/jsp/Login.jsp`

## Credenziali

| Risorsa | User | Password / Metodo |
|---|---|---|
| SSH VM810 | `intrawelt` | password locale (chiedere all'utente) |
| SSH host Proxmox | `root` | password root@pam web UI |
| DB MariaDB user app | `getrad` | `getradpwd` |
| DB MariaDB root | `root` | socket auth (no password, solo via `docker exec`) |
| Login app getrad | utente IT | password aziendale (chiedere all'utente) |

## Cosa è stato fatto (cronologia condensata)

### Patching JSP Tomcat 6 → Tomcat 9 (febbraio 2026, VM809 Ubuntu 20.04)

Jasper di Tomcat 9 è più rigoroso di Tomcat 6: qualsiasi attributo HTML con delimitatori
doppi apici il cui valore contenga codice Java con doppi apici interni produce HTTP 500
"Unable to compile class for JSP". Stesso problema per char literal Java multi-carattere
(`'id'` invece di `"id"`). La regola di correzione è: apici singoli esterni all'attributo
HTML, doppi apici interni al Java.

Eseguito da Tommaso Vezeni + Alessio Sopranzi via Teams (11-18 febbraio 2026) su VM809
(Ubuntu 20.04). Risultato: 11 file fixati, stato trasferito su VM810 nel tarball
`usr-skn-getrad`.

### Estrazione artefatti da VM809 e costruzione stack Docker (maggio 2026)

VM809 con app working è stata spenta, montato il suo disco read-only dall'host Proxmox,
estratti 5 tarball: mysql-datadir, usr-skn-getrad, usr-skn-solr, tomcat9-conf, etc-mysql
+ SHA256SUMS. Trasferiti via scp a VM810 (~12 GiB in ~3 minuti). VM809 successivamente
rimossa da Proxmox (`qm destroy 809 --purge`); safety net vzdump del 4-5 maggio conservato
su NAS aziendale.

### Bonifica VM810

Sistema era installato come Ubuntu Desktop. Rimossi: mariadb-server 10.11, openjdk-8/21,
Tomcat9 residuo manuale, GNOME complete, snap stack, cups, apache2, openvpn, firefox,
thunderbird, cloud-init. Disabilitati servizi non necessari. Target multi-user.target.

### Setup ambiente

Docker CE + Compose plugin da repo ufficiale; `intrawelt` aggiunto a gruppo `docker`; SSH
attivato; pacchetti operativi installati (vim, htop, curl, rsync, dnsutils, zstd,
qemu-guest-agent); pin apt per snapd con Priority -10.

### Costruzione stack Docker

- `docker-compose.yml`: 2 servizi con healthcheck e restart policy
- Dockerfile app: base `tomcat:9-jdk8`, COPY solr.war, COPY context fragment `getrad.xml`,
  COPY `catalina-skip.properties` (jarsToSkip esaustiva, riduce startup da 147s a 9.5s)
- Container db: bind mount del datadir + bind mount di `load-unix-socket.cnf` (richiesto
  perché il plugin unix_socket non è caricato di default nell'immagine MariaDB upstream)

### Patch applicate ai file di config dell'app

- `getrad.properties`: `jdbc:mysql://127.0.0.1:3306/getrad` → `jdbc:mysql://db:3306/getrad`
- `data-config.xml` di Solr: stessa modifica `localhost` → `db`
- `getrad.properties` riga 2: rimosso `MAIL` da `log4j.logger.ApplicationLogger=INFO, APP,
  MAIL` (il SMTPAppender tentava smtp.office365.com non raggiungibile dal container,
  generando timeout di 10-30 secondi su ogni pagina)
- Creato `getrad@'%'` nel datadir (esisteva solo `getrad@'localhost'`, insufficient per
  connessioni dalla rete bridge Docker)

### Operabilità installata

- Backup schedulato: `/usr/local/sbin/getrad-backup.sh weekly|monthly` da
  `/etc/cron.d/getrad-backup` (lunedì 02:00 weekly retention 30gg, primo del mese 03:00
  monthly retention 190gg), output zstd-compressi in `/srv/getrad-stack/backups/` con SHA256
- Logrotate `/etc/logrotate.d/getrad` per log applicativi in WEB-INF/logs/
- Firewall UFW: deny default, allow 22 e 8080
- Patching automatico: unattended-upgrades security-only
- Journal cap 200M

## Stato attuale verificato

- `docker compose ps`: entrambi i container `(healthy)`
- Login app funzionante via browser
- Tomcat startup: ~10 secondi a freddo
- Backup test eseguito con successo
- Spazio: 110 GiB usati su 250 GiB del disco VM810

## Lavori in sospeso

### Fase 5 — patching meccanico file JSP (PRIORITÀ ALTA)

Tutti i file JSP si trovano in `/srv/getrad-stack/extracted/getrad/jsp/` su VM810.

**File già fixati** (trasferiti da VM809 nello stato corretto, backup sorgente
`backup_usr_skn_getrad_jsp_18022026_1440`):

| File (relativo a `jsp/`) | Righe modificate |
|---|---|
| `clienti/index.jsp` | 171, 253, 318 |
| `fornitori/index.jsp` | 65, 119 |
| `clienti/insupdcliente.jsp` | 1236-1238, 1326, 1357, 1408-1409, 1414, 1618, 1645, 1680, 1741-1742, 1757, 1810-1828 (selettivi) |
| `clienti/contatti.jspf` | 369, 371, 374, 375 |
| `clienti/commerciali.jspf` | 33 |
| `clienti/pm_rif.jspf` | 22 |
| `clienti/cli_gruppo.jspf` | 26, 59 |
| `_include/templateBottomBlank.jspf` | 1 |
| `_include/templateBottom.jspf` | 4 |
| `preventivi/index.jsp` | 314, 319, 350, 355, 387-388, 539, 592, 600 |
| `preventivi/insupdpreventivo.jsp` | 1314-1315, 1337, 1367, 1381, 1386, 1408-1409, 1414, 1618-1828 (selettivi) |

**File PENDING — errori noti, da fixare in ordine**:

| File | Errore noto | Rotta di test |
|---|---|---|
| `fornitori/insupdfornitore.jsp` | righe 369, 370, 713, 714; `one.get('id')` a riga 672 | `/jsp/fornitori/insupdfornitore.jsp?id=13577496633860` |
| `preventivi/_combinazione.jsp` | HTTP 500; log da riprodurre via `docker compose logs` | `/jsp/preventivi/_combinazione.jsp` |
| `ordini/index.jsp` | line 211: `multiLang.getLabel(id_lingua, "label.cerca")` con apici errati | `/jsp/ordini/index.jsp` |
| `fatture/index.jsp` | via `fatture/index_tr.jspf` line 78 → `fatture/fatture_tr.jspf` line 479 | `/jsp/fatture/index.jsp` |
| `traduttori/banca.jsp` | mai toccato | — |
| `traduttori/emailtrad.jsp` | mai toccato | — |
| `provvigioni/index.jsp` | via `provvigioni/provvigioni_env.jspf` line 20 | `/jsp/provvigioni/index.jsp` |
| `riepiloghi/index.jsp` | via `riepiloghi/index_tr.jspf` line 86 | `/jsp/riepiloghi/index.jsp` |

**File candidati — mai toccati, errori latenti probabili** (da scansione grep su VM809):

```
getrad/js/ordini.jsp          clienti/area/charts.jsp
clienti/area/index.jsp        clienti/area/invoices.jsp
clienti/area/orders.jsp       clienti/area/quotes.jsp
clienti/area/include/_allegaForm.jsp
fatture/fattura.jsp           lead/associateLead.jsp
lead/getDatiAzienda.jsp       lead/insupdlead.jsp
lead/storico.jsp              search/index.jsp
societa/insupdsocieta.jsp     traduttori/cercaTrad.jsp
traduttori/comb_ling_int.jsp  traduttori/comb_ling.jsp
traduttori/home_trad.jsp      traduttori/professionali_int.jsp
traduttori/professionali.jsp  vipa/amministra.jsp
vipa/index.jsp                test/clienti_settori.jsp
ordini/20140603/insupdordine_env.jsp
```

Non modificare le sottocartelle `20140603/`, `20120523/`, `20210129/` (versioni storiche
archiviate) a meno che non producano errori su rotte attive.

**Tipi di errore da correggere** (casi A-E):

```
Caso A — attributo HTML con espressione JSP:
  PRIMA: label="<%= multiLang.getLabel(id_lingua, "label.salva") %>"
  DOPO:  label='<%= multiLang.getLabel(id_lingua, "label.salva") %>'

Caso B — jsp:param con valore JSP:
  PRIMA: <jsp:param name="id" value="<%= one.get("id") %>" />
  DOPO:  <jsp:param name="id" value='<%= one.get("id") %>' />

Caso C — direttiva include:
  PRIMA: <%@ include file="contatti.jspf" %>
  DOPO:  <%@ include file='contatti.jspf' %>

Caso D — char literal Java multi-carattere:
  PRIMA: one.get('id')         DOPO: one.get("id")

Caso E — confronto con char literal:
  PRIMA: if(one.get("fl_sms") == 'S')
  DOPO:  if("S".equals(one.get("fl_sms")))
```

**Prompt Windsurf** per patching automatico (genera `FILE_fixed.jsp` da confrontare prima
di applicare):

```
Esegui una scansione completa del file JSP aperto e correggi in modo deterministico tutti
gli errori che possono impedire la traduzione JSP → servlet → compilazione Java da parte
di Jasper su Tomcat 9. Identifica ed elimina ogni literal Java racchiuso in apici singoli
che contiene più di un carattere all'interno di scriptlet, expression tag o declaration tag.
Converti sempre i literal sbagliati da 'xxx' a "xxx". Mantieni chiari i livelli di quoting:
apici singoli esterni per HTML/XML, doppi apici interni per Java. Rileva e correggi anche
tutti i casi di conflitto tra apici HTML e apici Java all'interno degli attributi JSP,
assicurando che il contenuto Java sia sempre racchiuso tra doppi apici e che l'attributo
esterno utilizzi apici singoli. Verifica che non compaiano apici tipografici non ASCII e
sostituiscili con apici standard 0x22 o 0x27. Controlla la chiusura corretta di tutti i
tag <% %>, <%= %> e <%! %>. Correggi i literal char non validi e rimuovi ogni dichiarazione
Java sintatticamente non valida introdotta da errori precedenti. Assicurati che il codice
risultante sia compilabile dal compilatore Java usato da Jasper (Java 8+). Applica tutte le
modifiche in modo puntuale, senza introdurre refactoring logico o funzionale, intervenendo
esclusivamente sugli errori sintattici che impediscono la compilazione.
```

### Hardening Tomcat secondo OWASP (dopo Fase 5)

Riferimento da consultare: https://wiki.owasp.org/index.php/Securing_tomcat [non verificato
in questa sessione]. Punti attesi: rimozione webapp manager/host-manager, disabilitazione
directory listing, header HTTP di sicurezza, audit permessi filesystem container.

### Scorporo Solr in container separato (OPZIONALE, non urgente)

Vantaggi: isolamento failure, lifecycle indipendente. Svantaggi: container in più, modifica
URL Solr nell'app. Sconsigliato per un gestionale a consultazione statica con poco traffico.

### Test pratico disaster recovery

Mai testato un restore from-scratch. Da pianificare: host vergine Ubuntu 24.04, restore di
backup-stack-config + backup-db + alberi getrad/solr. Procedure in `README.md`.

## Errori operativi noti — da NON ripetere

1. **Conflitto IP fra VM809 e VM810** quando entrambe avevano .90. Causa cascata di problemi
   (SSH tagliato, web UI Proxmox temporaneamente inaccessibile). Risoluzione: spegnere la VM
   source prima di assegnare l'IP alla target. VM809 ora rimossa, non più rischio.

2. **Tomcat residuo manuale** sull'host VM810 (PID 1023 utente `tomcat`) sopravvissuto al
   primo round di pulizia perché senza unit systemd. Occupava la porta 8080 al primo
   `docker compose up`. Verificare sempre `ss -tlnp | grep :8080` prima di tirare su il
   container app.

3. **Plugin unix_socket non caricato di default** in `mariadb:10.3` upstream (mentre era
   presente nella 10.3 dei repo Ubuntu di VM809). Sintomo: `ERROR 1524 (HY000): Plugin
   'unix_socket' is not loaded`. Soluzione: bind mount di `load-unix-socket.cnf` in
   `/etc/mysql/conf.d/`.

4. **mysqldump con utente getrad** falliva su `SHOW CREATE FUNCTION` per insufficient
   privileges. Lo script di backup usa socket auth di root nel container, senza `-u getrad`.

5. **`docker compose up` con port already in use** lascia il container senza port mapping
   anche dopo la risoluzione. Serve `--force-recreate`.

## Note operative per Claude Code

L'utente lavora via SSH da Windows 11 verso la VM810 (`ssh intrawelt@192.168.20.90`).
Per modificare file dell'app: editare direttamente via vim sul bind mount oppure scp
locale → edit → scp di ritorno. Le modifiche ai bind mount sono visibili dal container
ma l'app non ricarica config a runtime: serve `docker compose restart app`. Le modifiche
al Dockerfile richiedono `docker compose build app && docker compose up -d --force-recreate app`.

Per il debug dei log applicativi:

- Log Tomcat container: `docker compose logs -f app`
- Log app applicativi: `/srv/getrad-stack/extracted/getrad/WEB-INF/logs/Application.log`
- Log DB: `docker compose logs -f db`
- Log host: `sudo journalctl -f`

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

## File di riferimento essenziali

| Path | Contenuto |
|---|---|
| `/srv/getrad-stack/README.md` | Documentazione operativa completa (backup, restore, troubleshooting) |
| `/srv/getrad-stack/docker-compose.yml` | Compose stack |
| `/srv/getrad-stack/build/app/Dockerfile` | Build immagine app |
| `/srv/getrad-stack/build/app/catalina-skip.properties` | jarsToSkip Tomcat |
| `/srv/getrad-stack/build/db-conf/load-unix-socket.cnf` | Plugin auth_socket MariaDB |
| `/srv/getrad-stack/extracted/getrad/WEB-INF/conf/getrad.properties` | Config app (toccare con backup) |
| `/srv/getrad-stack/extracted/getrad/jsp/` | Sorgenti JSP (target del patching Fase 5) |
| `/srv/getrad-stack/extracted/solr/conf/data-config.xml` | Config Solr DataImporter |
| `/usr/local/sbin/getrad-backup.sh` | Script backup |
| `/etc/cron.d/getrad-backup` | Schedule cron backup |
| `/etc/logrotate.d/getrad` | Logrotate config |

## Versioni runtime fissate

- Ubuntu host VM810: 24.04 LTS (noble), kernel 6.8.0-106-generic
- Docker engine: 29.4.x, Docker Compose: v5.1.3
- MariaDB: 10.3.39 (image `mariadb:10.3`)
- Tomcat: 9.x (image `tomcat:9-jdk8`)
- OpenJDK: 8 (dentro tomcat:9-jdk8)
- mysql-connector-java: 3.0.11 (dentro `WEB-INF/lib` di getrad)
- Solr: 3.5.0 (`solr.war` di gennaio 2012)

## Promemoria di sicurezza indipendente dal git

La password SMTP Office365 presente in `getrad.properties` è viva e riutilizzata su altri
sistemi: va ruotata in produzione, indipendentemente dallo stato del repository.

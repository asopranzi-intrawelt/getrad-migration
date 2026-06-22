# Work-log

> Append-only, in ordine cronologico inverso (la voce più recente in alto).

## 2026-06-22 - Estensione allowlist di produzione a due nuove postazioni LAN (e correzione IP)

Commit: fuori repo (script firewall su VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/firewall/getrad-firewall.sh` (backup
`getrad-firewall.sh.bak-2026-06-22`), schede `.claude/context/deployment.md`, `.claude/memory/`
Motivo: rendere la produzione accessibile in LAN anche a due postazioni nuove, PC-ALESSIA-NAS
(192.168.10.75, Alessia Nasini) e PC-ALESSANDRO (Alessandro Potalivo).

Esecuzione: backup datato dello script firewall, poi aggiunta dei due IP a `PROD_IPS` con commento
del nome postazione e aggiornamento del commento di intestazione. `TEST_IPS` invariato (sola
192.168.10.73). Lo script idempotente e' stato rieseguito: la catena `DOCKER-USER` elenca le RETURN
per i nuovi IP su porta 80 e 8080, i DROP finali restano in coda, il test 8090 resta riservato
all'amministrazione. La persistenza e' garantita perche' `getrad-firewall.service` (enabled, active)
ha `ExecStart` su quello stesso script e lo riapplica dopo Docker al boot. Stato finale
dell'allowlist di produzione, sei postazioni: .80 PC-SONIA, .81 PC-FABIO, .206 PC-ELISA, .73
PC-ALESSIO (Sopranzi), .75 PC-ALESSIA-NAS, .76 PC-ALESSANDRO.

Diagnosi di un mancato accesso, lezione operativa. Per PC-ALESSANDRO l'IP annotato era sbagliato
(prima 192.168.10.208, poi indicato come 192.168.0.208), e con quell'IP in allowlist l'accesso
falliva con "Impossibile raggiungere il sito". La causa non era il file hosts ne' il proxy del
browser: il nome `egetrad-login` risolveva bene e il ping passava, ma nessun pacchetto TCP arrivava
al server con quell'IP. Con regole di LOG temporanee davanti ai DROP e con il campo `SourceAddress`
di `Test-NetConnection` si e' visto che quel PC esce in rete come 192.168.10.76, non .208. Corretto
l'allowlist (rimosso il .208 mai usato, autorizzato il .76), `TcpTestSucceeded` e' passato a True.
PC-ALESSIA-NAS (.75) risultava gia' True. Lezione: l'IP da autorizzare si legge empiricamente dal
`SourceAddress` della macchina, non dai valori annotati, che qui erano imprecisi.

Mappatura hosts gia' eseguita dall'utente sui PC, con i tre nomi `egetrad`, `egetrad-login` ed
`egetrad.intrawelt.com` mappati su 192.168.20.90, coerente con il catch-all del proxy e con i link
assoluti legacy verso `egetrad.intrawelt.com:8080`. Su PC-ALESSANDRO resta da separare nel file
hosts la riga di `egetrad`, fusa per errore con quella di `195.96.193.36 intrawelt.com` (quindi
`egetrad` senza suffisso punta a un IP pubblico sbagliato); non blocca l'uso perche' gli utenti
usano `egetrad-login`.

Lavoro di rete soltanto: nessun JSP promosso, produzione applicativa intatta. La pulizia DB e la
protezione degli accessi indesiderati (incluso il restringimento per password dei soli account
ammessi) sono rimandate a una fase successiva, su indicazione dell'utente.

## 2026-06-19 - Promozione hardening in produzione (header di sicurezza e mascheramento versione)

Commit: fuori repo (produzione su VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/extracted/getrad/WEB-INF/web.xml`, `/srv/getrad-stack/conf/server.xml`
(nuovo), `/srv/getrad-stack/docker-compose.yml`, `backups/`, schede `.claude/`
Motivo: dopo la verifica completa su test (lato server e a video), promozione in produzione degli
interventi di hardening.

Esecuzione: backup di `web.xml` e del compose di produzione in `backups/` con tag
`prod-pre-hardening-2026-06-19`. Diff confermato che il `web.xml` di test aggiunge solo il blocco
filtro. Promozione `web.xml` con cp da test a prod e owner uniforme intrawelt. Per il mascheramento
versione, copiato il `server.xml` indurito in `/srv/getrad-stack/conf/server.xml` e aggiunto il
relativo bind-mount in sola lettura al compose di produzione (prima modifica al compose di prod,
backuppato). Ricreato il container app di produzione.

Verifica su produzione: app healthy; header `X-Frame-Options: SAMEORIGIN` e `X-Content-Type-Options:
nosniff` presenti su 8080 e anche attraverso il proxy sulla porta 80; versione Tomcat mascherata
nella 404; login 200 su 8080 e via nome amichevole. Hardening di base ora allineato tra test e
produzione. Restano opzionali, da decidere se necessari: HTTPS sulla LAN e `showReport=false`.

## 2026-06-19 - Hardening su test: header di sicurezza e mascheramento versione Tomcat

Commit: fuori repo (albero test e conf test su VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/test/extracted/getrad/WEB-INF/web.xml`,
`/srv/getrad-stack/test/conf/server.xml` (nuovo), `/srv/getrad-stack/test/docker-compose.yml`,
schede `.claude/`
Motivo: avvio della Fase 7 sugli interventi fattibili senza sorgente, su test come da indicazione.

Header di sicurezza: aggiunto al `web.xml` dell'app il filtro nativo Tomcat
`HttpHeaderSecurityFilter`, posizionato prima dei servlet per rispettare la DTD 2.3. Produce
`X-Frame-Options: SAMEORIGIN` (l'app usa frame propri, no DENY) e `X-Content-Type-Options: nosniff`.
HSTS disabilitato perche' non c'e' HTTPS. Verificato su 8090: header presenti, login 200, app
healthy.

Mascheramento versione: aggiunto un `ErrorReportValve` con `showServerInfo="false"` al `server.xml`,
montato in sola lettura nel solo container di test. Mantenuto `showReport` di default (gli errori
restano leggibili in test). Verificato: la pagina 404 mostra solo "HTTP Status 404", non piu'
"Apache Tomcat/9.0.117".

Debito MD5: confermato non affrontabile senza sorgente Java (assente sul server, logica di login
nei jar compilati). Mitigazione effettiva: l'allowlist di rete della Fase 6 limita il login a 4 IP.

Promozione futura in produzione: il `web.xml` si promuove via rsync come i JSP (sta in `WEB-INF/`);
il `server.xml` invece e' montato solo sul test, quindi per la produzione andra' montato anche li'
o cotto nel Dockerfile, come passo a parte. Per ora tutto resta su test.

Verifica degli header (2026-06-19): completa e positiva. Lato server: SAMEORIGIN sicuro perche' gli
iframe dell'app (FatturaPA, allegati, liberatoria, registrazione traduttori) usano `path_base` =
`common.base=/getrad`, percorso relativo, quindi same-origin; nosniff sicuro perche' gli asset
statici sono serviti con Content-Type corretti (text/javascript, image/jpeg, text/html, text/css).
Riscontro a video confermato dall'utente su test: navigazione, allegati, anteprima FatturaPA e
liberatoria si vedono correttamente, nessun riquadro vuoto ne' anomalia. Header pronti per la
promozione in produzione insieme al resto dell'hardening.

## 2026-06-19 - Promozione dei fix JSP in produzione (allineamento prod a test)

Commit: fuori repo (host VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/extracted/getrad/jsp/` (103 file), `backups/`, schede `.claude/`
Motivo: la produzione era ancora rotta sui moduli sistemati solo in test (ordini, fatture,
provvigioni, riepiloghi, traduttori, lead e altri davano 500), mentre i tre utenti di produzione
iniziavano ad accedere via nome amichevole. Decisione condivisa: allineare produzione a test ora,
proseguendo la verifica funzionale in test.

Razionale di rischio: i 103 file promossi erano esattamente quelli oggi rotti in produzione; le
pagine che gia' funzionavano non sono tra questi, perche' con il difetto di quoting darebbero 500.
Quindi la promozione poteva solo trasformare un 500 in pagina compilabile, senza regredire pagine
funzionanti. Correzione deterministica, 7 rotte gia' verificate a runtime.

Esecuzione: backup dell'albero JSP di produzione in `backups/getrad-jsp-prod-pre-promote-2026-06-19.tar.zst`
con SHA256; copia per contenuto (rsync --checksum, solo `jsp/`, niente configurazione quindi mailer
di prod invariato) di 103 file da test a prod; ripristino proprieta' uniforme intrawelt:intrawelt;
ricompilazione Jasper di produzione. Verifica: app prod healthy, rotte chiave 302, censimento su
8080 con 234 pagine che compilano, zero bug di quoting, stesse 24 pagine altro di test. Produzione
e test allineati. Mailer di produzione non toccato. Le quattro pagine non di quoting restano 500
come in test.

## 2026-06-19 - Nome amichevole egetrad e reverse proxy su porta 80

Commit: fuori repo (host VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/proxy/` (nginx.conf, docker-compose.yml), `getrad-firewall.sh`,
schede `.claude/`
Motivo: consentire agli utenti di produzione di raggiungere il gestionale con un nome amichevole
e una rotta di login facile, su PC Windows 11.

Introdotto un reverse proxy nginx sulla porta 80 (container `getrad-proxy`, progetto compose
separato agganciato alla rete di produzione come esterna, compose di produzione intatto). Radice e
`/egetrad-login` reindirizzano al login, il resto e' proxato verso `getrad-app:8080`. Verificato in
locale: ingresso e proxy rispondono 200, redirect corretti. Firewall esteso: porta 80 ammessa agli
stessi quattro IP della produzione, 8090 resta solo all'amministrazione; script firewall aggiornato
e riapplicato. Scelta in ADR-011.

Lato client serve aggiungere al file hosts dei PC autorizzati la mappatura di `egetrad` ed
`egetrad-login` su 192.168.20.90 (nessun DNS interno gestibile). Caveat legacy annotato: l'app
genera alcune URL assolute verso `egetrad.intrawelt.com:8080`, da gestire a parte se daranno
problemi sulla LAN. Produzione applicativa intatta, nessun JSP promosso.

## 2026-06-19 - Allowlist di rete (Fase 6) e audit degli account di login

Commit: fuori repo (host VM810) + schede `.claude/`
File toccati: `/srv/getrad-stack/firewall/` (script e backup iptables),
`/etc/systemd/system/getrad-firewall.service`, `_notes/` (export non versionato), schede
`.claude/context/` e `.claude/memory/`
Motivo: avvio della Fase 6 della roadmap (restrizione accesso ai soli IP statici LAN) e richiesta
di audit degli account che accedono al gestionale.

Allowlist di rete: applicata in `DOCKER-USER` (non UFW, che Docker bypassa per le porte pubblicate)
filtrando sulla porta di destinazione originale via conntrack. Produzione 8080 consentita a
192.168.10.80, .81, .206, .73; test 8090 alla sola .73. Script idempotente
`/srv/getrad-stack/firewall/getrad-firewall.sh`, reso persistente da `getrad-firewall.service`
(After=docker, abilitato). Backup iptables pre-modifica salvato. Verificato: regole presenti e
idempotenti, sopravvivono al restart di Docker, accesso locale host ancora 200, SSH non coinvolto.
Manca la verifica empirica del blocco da una postazione non autorizzata. Scelta motivata in ADR-010.

Audit account di login (tabella `an_utenti`): circa 8.200 account loggabili, di cui 8.129
Traduttori e 76 Contatti (self-service esterni) e una decina di account interni di staff. Tutte le
password sono hash MD5 a 32 cifre esadecimali, non testo in chiaro, quindi non recuperabili in
chiaro. Finding di sicurezza per la Fase 7: MD5 probabilmente non salato, debole; alcune password
mai cambiate da anni (un operatore fermo al 2007). Export completo username e hash in
`_notes/utenti_loggabili_2026-06-19.csv`, non versionato (le schede non contengono credenziali).

## 2026-06-19 - Censimento completo e patching massivo del quoting JSP su test

Commit: fuori repo (modifiche su albero test in `/srv/getrad-stack/test/`) + schede `.claude/`
File toccati: ~90 file `.jsp`/`.jspf` sotto `/srv/getrad-stack/test/extracted/getrad/jsp/`,
`.claude/context/`, `.claude/memory/`
Motivo: l'utente ha confermato che le sette rotte pending rendono bene nel browser e ha chiesto
un giro completo per confermare in via definitiva che l'intera applicazione e' funzionante.

Metodo: censimento deterministico di tutte le 253 pagine `.jsp` su test (porta 8090) tramite
richiesta HTTP e classificazione dell'esito. Una pagina che redirige al login (302) ha superato
la compilazione Jasper; una rotta rotta resta 500 anche senza login. Prima passata: 163 compilano,
72 con il bug di quoting Caso A/B, 18 altri errori. Le sette rotte gia' note erano tra le 72.

Correzione massiva: script Perl deterministico che converte gli attributi di soli tag `skn:*` e
azioni `jsp:*` la cui valore e' uno scriptlet `="<%= ... %>"` con doppi apici interni, portandoli
ad apici singoli esterni. Vincoli di sicurezza appresi e applicati: si interviene solo su righe con
tag `skn:`/`jsp:` (Jasper non applica la regola agli attributi HTML di template, confermato
empiricamente), solo su scriptlet singoli ancorati alla virgoletta di chiusura (niente spanning tra
attributi), solo se l'espressione non contiene apici singoli. Elaborazione a byte perche' i file
sono Latin-1, non UTF-8. Snapshot dell'albero jsp prima della modifica. Un primo tentativo con
regex non ancorata aveva corrotto alcune righe `<img src="<%=...%>/x.png">` ed e' stato annullato
da snapshot prima di qualsiasi compilazione.

Due casi a quoting misto (`traduttori/_comb_ling.jspf`, `_comb_ling_int.jspf`: espressione
`"javascript:settori('"+n_comb+"');"` con doppi e singoli apici) corretti a mano usando l'escape
unicode Java `'` per i singoli apici, mantenendo apici singoli esterni.

Esito finale (sweep ripetuto): 229 delle 253 pagine compilano, zero bug di quoting residui. La
migrazione del quoting JSP per Tomcat 9 e' completa su test. Le 24 pagine ancora 500 non sono bug
di quoting: sono in larga parte frammenti inclusi (falliscono solo se aperti da soli, funzionano
tramite la pagina genitore che compila), piu' la pagina gestore errori `error.jsp` e una pagina di
test, piu' un piccolo gruppo di pagine reali con un problema diverso e preesistente (variabili o
campi non risolti, non quoting): `comb_ling_complex.jsp` (`isCont`, `isMaintenanceMode`),
`revisioni.jsp` (`ric_rev_int`, `prefs`), `preventivo0.jsp` e `preventivo1.jsp`
(`GlobalDef.PRE_TEMPLATE`). Questi ultimi richiedono il parere di dominio sull'effettivo utilizzo.

Investigazione dei quattro casi: `GlobalDef.PRE_TEMPLATE` non esiste nel codice compilato in
`getrad.jar`, quindi `preventivo0/1.jsp` non sono correggibili a livello JSP senza il sorgente;
`comb_ling_complex.jsp` include `templateTop.jspf` senza definire `isCont`/`isMaintenanceMode`
(che le pagine complete ottengono includendo `_include/login.jspf`). Nessuno e' un fix rapido e
l'utilizzo non e' confermato, quindi sono lasciati fuori dal patching. Creato
`CAMMINATA_VERIFICA.md` alla radice del repository: piano di prova manuale completo, per modulo,
da usare quando si schedula la verifica nel browser.

Tutte le modifiche sono su test. Produzione intatta. Nulla promosso.

## 2026-06-19 - Backup pre-patching e predisposizione ambiente di test sulla LAN

Commit: fuori repo (infrastruttura su VM810) + schede `.claude/` aggiornate (commit manuale
dell'utente)
File toccati: `/srv/getrad-stack/backups/` (nuovo backup pre-patching), `/srv/getrad-stack/test/`
(nuovo progetto compose, codice, Solr, datadir, script), `.claude/memory/` e `.claude/context/`
Motivo: ripresa dal punto di `index.md` (backup pre-patching, poi `fornitori/insupdfornitore.jsp`)
con il requisito aggiuntivo di servire sulla LAN sia un ambiente di test per verificare le
modifiche sia quello di produzione per l'uso normale, dallo stesso host.

Backup pre-patching prodotto e verificato in `/srv/getrad-stack/backups/`:
`getrad-db-pre-patching-2026-06-19.sql.zst` (dump completo, 19 routine, trailer "Dump completed",
prodotto come `root@localhost` via socket perché l'utente `getrad` non ha privilegi su
`mysql.proc`) e `getrad-code-pre-patching-2026-06-19.tar.zst` (albero codice 284 MiB, esclusi i
quindici gibibyte di `files/` e il `tmp/` rigenerabile). Checksum in `SHA256SUMS-pre-patching-2026-06-19`,
riverificati OK.

Ambiente di test (progetto compose `getrad-test`, vedi ADR-007/008/009): codice copiato in
`/srv/getrad-stack/test/extracted/getrad/` (293M), Solr copiato per intero (9.5G), allegati
`files/` montati da produzione in sola lettura, datadir DB proprio inizializzato fresco e
seminato dal dump pre-patching (86 tabelle e 19 routine, identico a prod). App di test sulla
porta 8090 riusando l'immagine `getrad-stack-app:latest`. Mailer applicativo disattivato nella
copia di test. Aggiunti `test/promote-to-prod.sh` e `test/recompile-test.sh`.

Verifiche al termine: entrambi gli ambienti healthy (prod 8080, test 8090), `Login.jsp` su test
HTTP 200, isolamento DB confermato (alias `db` risolve al solo `getrad-test-db`, app di test su
sola rete `getrad-test_getrad-net`), rotta `fornitori/insupdfornitore.jsp` HTTP 500 sia su test
sia su prod (test riproduce fedelmente il bug noto), `files/` in sola lettura. Produzione non
toccata. Disco passato da 111G a 121G usati su 250G.

Discrepanze rilevate rispetto alla documentazione esistente, da correggere nelle schede: il
docBase del context fragment è `/usr/skn/getrad` (non `/usr/local/tomcat/webapps/getrad`); UFW
risulta `inattivo` sull'host nonostante il work-log lo desse configurato.

## 2026-06-19 - Sincronizzazione iniziale delle schede di contesto

Commit: 4f686bf
File toccati: `.claude/context/` (tutte le schede), `.claude/memory/` (tutte le schede),
`HANDOFF_getrad.md`
Motivo: prima popolazione delle schede scaffold con i dati effettivi del progetto, estratti
da `README.md`, `HANDOFF_getrad.md` (sessione precedente) e `GETRAD_TOMCAT_MIGRATION.md`.
Stato del patching JSP (Fase 5) mappato in `current-work.md`. Roadmap aggiornata in
`roadmap.md`. HANDOFF_getrad.md aggiornato con lo stato completo del patching JSP e i
path corretti per VM810.

## 2026-06-19 - Bootstrap repository git e struttura progetto (Windows)

Commit: 4f686bf - "Initial commit: documentazione migrazione getrad + adozione standard di
progetto"
File toccati: `CLAUDE.md`, `.gitignore`, `.claude/`, `README.md`, `GETRAD_TOMCAT_MIGRATION.md`,
`getrad.properties.example`
Motivo: primo push del repository `getrad-migration` su GitHub. Identità
`asopranzi@intrawelt.com`, remote `git@github-corp:asopranzi-intrawelt/getrad-migration.git`,
`core.sshCommand` fissato su OpenSSH di sistema (Windows). `getrad.properties` confermato
gitignored (segreti non esposti, leak_segreti=0).

## 2026-05 - Containerizzazione stack su VM810 (Ubuntu 24.04 LTS)

Commit: fuori repo (modifiche dirette su VM810, nessun git)
File toccati: `/srv/getrad-stack/` su VM810
Motivo: lo stack getrad era su VM809 (Ubuntu 20.04) con Tomcat/Java/MariaDB installati
tramite apt. Ubuntu 24.04 non pacchettizza più OpenJDK 8. Soluzione: containerizzare su
VM810 con Docker compose, freezing delle versioni runtime nell'immagine.

Passi eseguiti:
- Estrazione tarball da VM809 (montata read-only da Proxmox): mysql-datadir, usr-skn-getrad,
  usr-skn-solr, tomcat9-conf, etc-mysql + SHA256SUMS (~12 GiB)
- Trasferimento via scp da host Proxmox a VM810 direttamente
- Bonifica VM810 (era installata come Desktop Ubuntu): rimossi GNOME complete, snap stack,
  mariadb-server 10.11, openjdk-8/21, tomcat9 manuale residuo, cups, apache2, firefox,
  thunderbird, cloud-init. Disabilitati servizi non necessari. Target multi-user.target
- Installazione Docker CE + Compose plugin da repo ufficiale download.docker.com
- Costruzione docker-compose.yml + Dockerfile app (tomcat:9-jdk8, COPY solr.war, COPY
  context fragment getrad.xml, COPY catalina-skip.properties - startup da 147s a 9.5s)
- Patch configurazione app: JDBC URL `127.0.0.1:3306` → `db:3306`, Solr `localhost` → `db`
- Rimosso MAIL appender da log4j: eliminati timeout multipli-secondi per connessione
  smtp.office365.com non raggiungibile dal container
- Creato utente `getrad@'%'` nel DB (esisteva solo `getrad@'localhost'`, non sufficiente
  per connessioni via Docker bridge)
- Bind mount `load-unix-socket.cnf` per plugin unix_socket MariaDB (non caricato di default
  nell'immagine upstream, mentre era presente nell'Ubuntu 20.04 pacchetto)
- Operabilità: script backup + cron (`weekly` lunedì 02:00, `monthly` 1° del mese 03:00),
  logrotate, UFW, unattended-upgrades security-only, journal cap 200M
- Documentazione: `README.md` su VM810, `HANDOFF_getrad.md` nel repository

Stato verificato al termine: entrambi container (healthy), login app funzionante, startup
Tomcat ~10s, backup test OK, spazio 110 GiB su 250 GiB.

VM809 rimossa da Proxmox (`qm destroy 809 --purge`). Safety net vzdump del 4-5 maggio 2026
conservato su NAS aziendale.

## 2026-02-11 al 2026-02-18 - Patching JSP Tomcat 6 → Tomcat 9 (VM809, Ubuntu 20.04)

Commit: fuori repo (modifiche dirette su VM809)
File toccati: `/usr/skn/getrad/jsp/` su VM809 (ora in `/srv/getrad-stack/extracted/getrad/jsp/`)
Motivo: Jasper di Tomcat 9 è più rigoroso di Tomcat 6 sul quoting degli attributi JSP.
Qualsiasi attributo HTML con delimitatori doppi apici che contenga codice Java con doppi
apici interni produce HTTP 500. Stessa incompatibilità per char literal Java multi-carattere
(`'id'` invece di `"id"`).

Passi eseguiti (Tommaso Vezeni + Alessio Sopranzi via Teams, 11-18 febbraio 2026):
- Identificazione root cause (errore Jasper "Attribute value [...] is quoted with [\"\"] which
  must be escaped")
- Sviluppo prompt Windsurf per patching automatico (genera FILE_fixed.jsp)
- Patching iterativo file per file: modifica manuale o via Windsurf → confronto diff →
  copia sul server → riavvio Tomcat → verifica rotta
- Backup stabili su Desktop VM809 (`backup_usr_skn_getrad_jsp_*`)

Risultato: 11 file fixati (clienti, fornitori base, template globali, preventivi). Circa 8
file con errori noti pending, ~24 file mai toccati. Ultimo backup stabile:
`backup_usr_skn_getrad_jsp_18022026_1440`. Questo stato è quello trasferito su VM810.

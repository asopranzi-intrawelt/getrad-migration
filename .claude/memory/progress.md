# Work-log

> Append-only, in ordine cronologico inverso (la voce più recente in alto).

## 2026-06-19 — Sincronizzazione iniziale delle schede di contesto

Commit: 4f686bf
File toccati: `.claude/context/` (tutte le schede), `.claude/memory/` (tutte le schede),
`HANDOFF_getrad.md`
Motivo: prima popolazione delle schede scaffold con i dati effettivi del progetto, estratti
da `README.md`, `HANDOFF_getrad.md` (sessione precedente) e `GETRAD_TOMCAT_MIGRATION.md`.
Stato del patching JSP (Fase 5) mappato in `current-work.md`. Roadmap aggiornata in
`roadmap.md`. HANDOFF_getrad.md aggiornato con lo stato completo del patching JSP e i
path corretti per VM810.

## 2026-06-19 — Bootstrap repository git e struttura progetto (Windows)

Commit: 4f686bf — "Initial commit: documentazione migrazione getrad + adozione standard di
progetto"
File toccati: `CLAUDE.md`, `.gitignore`, `.claude/`, `README.md`, `GETRAD_TOMCAT_MIGRATION.md`,
`getrad.properties.example`
Motivo: primo push del repository `getrad-migration` su GitHub. Identità
`asopranzi@intrawelt.com`, remote `git@github-corp:asopranzi-intrawelt/getrad-migration.git`,
`core.sshCommand` fissato su OpenSSH di sistema (Windows). `getrad.properties` confermato
gitignored (segreti non esposti, leak_segreti=0).

## 2026-05 — Containerizzazione stack su VM810 (Ubuntu 24.04 LTS)

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
  context fragment getrad.xml, COPY catalina-skip.properties — startup da 147s a 9.5s)
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

## 2026-02-11 al 2026-02-18 — Patching JSP Tomcat 6 → Tomcat 9 (VM809, Ubuntu 20.04)

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

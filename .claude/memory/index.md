# Snapshot di sincronizzazione

> Da leggere per primo a inizio sessione. Fotografa lo stato del progetto al commit di
> riferimento e mappa ogni scheda al suo stato di verifica. È la fonte di verità su cosa è
> fatto, non le spunte del diario.

## Stato

```
Branch attivo:          main
Commit di riferimento:  bc84938
Data snapshot:          2026-06-22
```

## Stato di verifica delle schede

| Scheda | last-verified | Stato |
|---|---|---|
| STACK.md | 4f686bf | aggiornata |
| design-and-security.md | 4f686bf | aggiornata |
| deployment.md | 4f686bf | aggiornata |
| dev-testing.md | 4f686bf | aggiornata |
| current-work.md | 4f686bf | aggiornata |
| roadmap.md | 4f686bf | aggiornata |

## Punto di ripresa

Stato al 2026-06-19, fine giornata. Fatto: backup pre-patching verificato in
`/srv/getrad-stack/backups/`; ambiente di test attivo e isolato sulla LAN (progetto `getrad-test`,
porta 8090; vedi ADR-007/008/009); migrazione quoting JSP completa su test (229/253 pagine
compilano, zero bug di quoting residui, sweep deterministico); piano di camminata funzionale in
`CAMMINATA_VERIFICA.md`; allowlist di accesso LAN applicata e persistente via systemd
(`getrad-firewall.service`, regole in `DOCKER-USER`, vedi ADR-010 e `deployment.md`); nome
amichevole `egetrad`/`egetrad-login` su porta 80 tramite reverse proxy nginx in
`/srv/getrad-stack/proxy/` (ADR-011), con la 80 nell'allowlist. Migrazione quoting JSP
promossa in produzione il 2026-06-19: produzione allineata a test (234 pagine compilano, zero bug
di quoting), backup pre-promozione in `backups/getrad-jsp-prod-pre-promote-2026-06-19.tar.zst`.
Mailer di produzione non ancora toccato. Hardening di base (header di sicurezza
`X-Frame-Options`/`X-Content-Type-Options` e mascheramento versione Tomcat) applicato e verificato
su test e poi promosso in produzione il 2026-06-19, vedi ADR-013 e `roadmap.md`.

Tutto lo stato operativo vive su disco su VM810 ed e' ripristinabile: albero di test in
`/srv/getrad-stack/test/`, script firewall in `/srv/getrad-stack/firewall/` con backup iptables
pre-modifica, export utenti loggabili (con hash MD5) in `_notes/` non versionato.

Il 2026-06-22 l'allowlist di produzione e' stata estesa a sei postazioni LAN, aggiungendo
PC-ALESSIA-NAS (192.168.10.75, Alessia Nasini) e PC-ALESSANDRO (192.168.10.76, Alessandro Potalivo)
su porta 80 e 8080; accesso verificato dal vivo (`TcpTestSucceeded: True`) da entrambe. Set completo:
.80 PC-SONIA, .81 PC-FABIO, .206 PC-ELISA, .73 PC-ALESSIO (Sopranzi), .75 PC-ALESSIA-NAS, .76
PC-ALESSANDRO. L'IP di PC-ALESSANDRO era stato annotato male (.208): l'IP reale .76 e' stato letto
dal `SourceAddress` di `Test-NetConnection`. Mappatura hosts gia' fatta lato client; su
PC-ALESSANDRO la riga di `egetrad` nel file hosts, prima fusa con `195.96.193.36 intrawelt.com`, e'
stata separata e ora risolve a 192.168.20.90.

Sempre il 2026-06-22 e' stato fatto il blocco accessi al gestionale: il login e' ora ristretto a sei
account interni (`smartellini`, `fguidali`, `emonterubb`, `anasini`, `apotalivo`, `getrad`),
disattivando `fl_attivo` su tutti gli altri 10.140 account loggabili senza cancellare alcuna
anagrafica. Reversibile, backup `backups/an_utenti-prod-pre-lockdown-2026-06-22.sql`, enforcement di
`fl_attivo` verificato su test e in produzione. Password dell'admin `getrad` reimpostata a un valore
noto all'utente, non versionata. Dettagli e razionale nel work-log.

Il 2026-06-23 disattivato il mailer applicativo in produzione (mail.host e log4j MAIL su 127.0.0.1,
auth off, credenziali svuotate): eliminato il timeout SMTP che rallentava il login di ~20s, e
rimossa la password Office365 viva dalla config attiva (backup pre-modifica da bonificare).
Reimpostata anche la password di `smartellini`. Password mai versionate.

Il 2026-06-23 cifrati i backup a chiave pubblica GPG: chiave dedicata RSA4096, solo la pubblica sul
server (keyring `/etc/getrad-backup-gpg`, fingerprint 34688023949F4C34E51F45752C064A4CB038334D), la
privata custodita fuori dal server e cancellata da qui. `getrad-backup.sh` ora produce `*.zst.gpg`,
giro cifra/ripristina verificato, i 21 backup storici in chiaro sono stati cifrati a posteriori e i
vecchi getrad.properties con segreti eliminati. Restore aggiornato nel README (serve `gpg --decrypt`
e la chiave privata). Restano in chiaro solo gli artefatti di rollback temporanei da far scadere.

Prossimi passi: schedulare la camminata di verifica funzionale nel browser seguendo
`CAMMINATA_VERIFICA.md`, che l'utente esegue in test (ogni problema runtime si corregge in test e
si ripromuove con `test/promote-to-prod.sh`); proseguire la roadmap di sicurezza (Fase 7 hardening
legacy, incluso il debito sull'hashing MD5 delle password). Quattro pagine
restano non risolte per problemi non di quoting (vedi `current-work.md`), in attesa di conferma sul
loro utilizzo.

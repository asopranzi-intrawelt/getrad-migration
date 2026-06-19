# Snapshot di sincronizzazione

> Da leggere per primo a inizio sessione. Fotografa lo stato del progetto al commit di
> riferimento e mappa ogni scheda al suo stato di verifica. È la fonte di verità su cosa è
> fatto, non le spunte del diario.

## Stato

```
Branch attivo:          main
Commit di riferimento:  11b13e8
Data snapshot:          2026-06-19
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
Mailer di produzione non ancora toccato.

Tutto lo stato operativo vive su disco su VM810 ed e' ripristinabile: albero di test in
`/srv/getrad-stack/test/`, script firewall in `/srv/getrad-stack/firewall/` con backup iptables
pre-modifica, export utenti loggabili (con hash MD5) in `_notes/` non versionato.

Prossimi passi: schedulare la camminata di verifica funzionale nel browser seguendo
`CAMMINATA_VERIFICA.md`, che l'utente esegue in test (ogni problema runtime si corregge in test e
si ripromuove con `test/promote-to-prod.sh`); proseguire la roadmap di sicurezza (Fase 7 hardening
legacy, incluso il debito sull'hashing MD5 delle password); completare la mappatura hosts sui PC di
Sonia ed Elisa. Quattro pagine restano non risolte per problemi non di quoting (vedi
`current-work.md`), in attesa di conferma sul loro utilizzo.

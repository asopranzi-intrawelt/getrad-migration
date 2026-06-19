---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - GETRAD_TOMCAT_MIGRATION.md
  - HANDOFF_getrad.md
last-verified-commit: 4f686bf
stato: in corso
---

# Lavoro in corso

## Feature: Fase 5 - patching meccanico rotte JSP (compatibilità Tomcat 9)

Cosa fa: corregge i file JSP scritti per Tomcat 6/Jasper 2 che su Tomcat 9/Jasper 9 producono
HTTP 500 "Unable to compile class for JSP". Le incompatibilità sono sintattiche: attributi HTML
con delimitatori doppi apici il cui valore contiene codice Java con doppi apici interni, e char
literal Java multi-carattere (`'id'` invece di `"id"`). La regola è: apici singoli esterni
all'attributo HTML, doppi apici interni al Java.

Dal 2026-06-19 il patching si esegue sull'ambiente di test e si promuove in produzione (ADR-007).
I JSP da modificare e verificare stanno in `/srv/getrad-stack/test/extracted/getrad/jsp/` (test,
porta 8090); la produzione resta in `/srv/getrad-stack/extracted/getrad/jsp/` (porta 8080) e si
aggiorna solo via `test/promote-to-prod.sh` dopo che la rotta è corretta su test. Il backup
pre-patching del 2026-06-19 è in `/srv/getrad-stack/backups/` (`getrad-db-pre-patching-2026-06-19.sql.zst`
e `getrad-code-pre-patching-2026-06-19.tar.zst`, checksum verificati).

### File già fixati

Transferiti da VM809 (Ubuntu 20.04) nello stato corretto - riferimento: backup
`backup_usr_skn_getrad_jsp_18022026_1440` del 2026-02-18 ore 14:38.

| File (relativo a `jsp/`) | Righe modificate |
|---|---|
| `clienti/index.jsp` | 171, 253, 318 |
| `fornitori/index.jsp` | 65, 119 |
| `clienti/insupdcliente.jsp` | 1236-1238, 1326, 1357, 1408-1409, 1414, 1618, 1645, 1680, 1741-1742, 1757, 1810, 1815, 1817, 1819, 1828 |
| `clienti/contatti.jspf` | 369, 371, 374, 375 |
| `clienti/commerciali.jspf` | 33 |
| `clienti/pm_rif.jspf` | 22 |
| `clienti/cli_gruppo.jspf` | 26, 59 |
| `_include/templateBottomBlank.jspf` | 1 |
| `_include/templateBottom.jspf` | 4 |
| `preventivi/index.jsp` | 314, 319, 350, 355, 387-388, 539, 592, 600 |
| `preventivi/insupdpreventivo.jsp` | 1314-1315, 1337, 1367, 1381, 1386, 1408-1409, 1414, 1618, 1645, 1680, 1741-1742, 1757, 1810, 1815, 1817, 1819, 1828 |

### File PENDING - stato dopo patching su test del 2026-06-19

Tutti i file sotto sono stati corretti sull'albero di test e ora compilano (Jasper OK, rotta
302 verso `/getrad/Login`). Restano da verificare nel browser da utente loggato e poi da
promuovere in produzione con `test/promote-to-prod.sh`. Le correzioni sono dello stesso tipo:
attributo di tag custom `skn:*` o azione `jsp:*` con apici doppi esterni e apici doppi interni
nell'espressione, riportato ad apici singoli esterni (Caso A/B). Gli attributi HTML di template
puro non sono toccati perché Jasper non vi applica la regola di quoting.

| File e catena di include corretti | Rotta di test (porta 8090) | Stato |
|---|---|---|
| `fornitori/insupdfornitore.jsp` (riga 672) | `/jsp/fornitori/insupdfornitore.jsp?id=13577496633860` | compila, da verificare |
| `ordini/index.jsp` (211, 216, 318, 496) | `/jsp/ordini/index.jsp` | compila, da verificare |
| `fatture/index.jsp` via `fatture_tr.jspf`, `modifica_tr.jspf`, `modifica_tr.jsp`, `modifica_cl.jspf`, `index_cl.jspf`, `fatture_cl.jspf` | `/jsp/fatture/index.jsp` | compila, da verificare |
| `traduttori/banca.jsp` (bottoni salva/elimina/close e `jsp:param` riga 905) | `/jsp/traduttori/banca.jsp` | compila, da verificare |
| `traduttori/emailtrad.jsp` (328, 329) | `/jsp/traduttori/emailtrad.jsp` | compila, da verificare |
| `provvigioni/index.jsp` (116) e `provvigioni_env.jspf` (20) | `/jsp/provvigioni/index.jsp` | compila, da verificare |
| `riepiloghi/index.jsp` via `index_tr.jspf` (86) e `index_cl.jspf` (128) | `/jsp/riepiloghi/index.jsp` | compila, da verificare |
| `preventivi/_combinazione.jsp` | frammento incluso da `insupdpreventivo.jsp` | non si patcha da solo: gli errori standalone sono variabili non dichiarate fornite dal genitore; si verifica aprendo la pagina di un preventivo |

Le sette rotte qui sopra sono state verificate nel browser da utente loggato il 2026-06-19 e
rendono correttamente.

### Censimento completo e patching massivo del 2026-06-19

Per confermare in via definitiva l'intera applicazione si è fatto un censimento deterministico di
tutte le 253 pagine `.jsp` su test, richiedendo ogni rotta e classificando l'esito. Una correzione
massiva con script Perl ha poi sistemato in un colpo tutti gli attributi di tag `skn:*` e azioni
`jsp:*` affetti dal Caso A/B, con i vincoli di sicurezza descritti in `progress.md`. Due casi a
quoting misto sono stati corretti a mano con l'escape unicode Java.

Esito definitivo: 229 delle 253 pagine compilano, zero bug di quoting residui. La migrazione del
quoting JSP per Tomcat 9 è completa su test e, dal 2026-06-19, promossa in produzione: i 103 file
corretti sono stati allineati da test a prod via rsync per contenuto, con backup pre-promozione in
`backups/getrad-jsp-prod-pre-promote-2026-06-19.tar.zst`. Censimento su produzione (8080): 234
pagine compilano, zero quoting, stesse 24 altro. Produzione e test allineati.

Le 24 pagine che restano 500 NON sono bug di quoting e si dividono così. La maggior parte sono
frammenti inclusi che falliscono solo se aperti da soli e funzionano tramite la pagina genitore che
compila (per esempio `preventivi/_combinazione.jsp`, `ordini/altro.jsp`, `ordini/dettaglio.jsp`,
`ordini/interpretariato.jsp` e le varianti `all_*`, `fatture/_tr_riba_multiplo.jsp`,
`ordini/_tabella_qta_cliente.jsp`, `lead/top_barra_cm.jsp`, `traduttori/ordini_pm.jsp`,
`ordini_trad.jsp`, `combinazione.jsp`, `profilo.jsp`, `competenze.jsp`, `traduttore.jsp`,
`nota.jsp`), più la pagina gestore errori `error.jsp` e la pagina `test/test.jsp`. Un piccolo
gruppo è invece costituito da pagine reali con un problema diverso e preesistente, non di quoting,
da decidere con il parere di dominio sull'effettivo utilizzo: `traduttori/comb_ling_complex.jsp`
(variabili `isCont` e `isMaintenanceMode` non definite prima dell'include del template),
`traduttori/revisioni.jsp` (`ric_rev_int`, `prefs` non risolti), `preventivi/preventivo0.jsp` e
`preventivo1.jsp` (`GlobalDef.PRE_TEMPLATE` non risolto come campo).

Le quattro pagine con problemi non di quoting sono state investigate il 2026-06-19 e lasciate fuori
dal patching perché non sono fix rapidi e il loro effettivo utilizzo non è confermato.
`preventivi/preventivo0.jsp` e `preventivo1.jsp` riferiscono `GlobalDef.PRE_TEMPLATE`, campo che non
esiste piu' nel codice compilato in `WEB-INF/lib/getrad.jar`: non correggibili a livello JSP senza
il sorgente Java. `traduttori/comb_ling_complex.jsp` diverge dal contratto dei template includendo
`templateTop.jspf` senza definire `isCont` e `isMaintenanceMode` come fanno le pagine complete che
includono `_include/login.jspf`; `traduttori/revisioni.jsp` ha variabili non risolte. Tutte e
quattro vanno affrontate solo dopo conferma che siano in uso, con eventuale accesso al sorgente.

Il patching del quoting JSP è concluso e promosso in produzione. Resta da fare: la camminata di
verifica funzionale nel browser, pianificata in `CAMMINATA_VERIFICA.md`, che l'utente eseguirà in
test; ogni eventuale problema runtime emerso si corregge su test e si ripromuove con
`test/promote-to-prod.sh`. Prosegue inoltre la roadmap di sicurezza (Fase 7 hardening legacy,
incluso il debito sull'hashing MD5). Fase 6 (allowlist IP) e nome amichevole egetrad già fatti.

### File candidati - mai toccati, errori latenti probabili

Identificati da scansione grep ricorsiva su VM809, non ancora testati su VM810:

```
getrad/js/ordini.jsp
clienti/area/charts.jsp
clienti/area/include/_allegaForm.jsp
clienti/area/index.jsp
clienti/area/invoices.jsp
clienti/area/orders.jsp
clienti/area/quotes.jsp
fatture/fattura.jsp
lead/associateLead.jsp
lead/getDatiAzienda.jsp
lead/insupdlead.jsp
lead/storico.jsp
ordini/20140603/insupdordine_env.jsp
search/index.jsp
societa/insupdsocieta.jsp
test/clienti_settori.jsp
traduttori/cercaTrad.jsp
traduttori/comb_ling_int.jsp
traduttori/comb_ling.jsp
traduttori/home_trad.jsp
traduttori/professionali_int.jsp
traduttori/professionali.jsp
vipa/amministra.jsp
vipa/index.jsp
```

Non modificare file nelle sottocartelle `20140603/`, `20120523/`, `20210129/` (versioni
storiche archiviate) a meno che non producano errori su rotte attive.

### Definition of done

- [ ] Tutti i moduli principali (clienti, fornitori, preventivi, ordini, fatture, traduttori, provvigioni, riepiloghi) raggiungibili senza HTTP 500
- [ ] Backup stabile post-patch in `/srv/getrad-stack/backups/` con SHA256
- [ ] `README.md` su VM810 aggiornato con la data dell'ultimo patching

### Domande aperte

Il log `HTTP500_combinazione_log.txt` era su Desktop di VM809 (ora distrutta). Riprodurre
accedendo alla rotta su VM810 e leggendo `docker compose logs --tail=200 app`.

## Riconciliazione

Ultima verifica: 2026-06-19 al commit 4f686bf.

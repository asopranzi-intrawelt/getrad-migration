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

## Feature: Fase 5 — patching meccanico rotte JSP (compatibilità Tomcat 9)

Cosa fa: corregge i file JSP scritti per Tomcat 6/Jasper 2 che su Tomcat 9/Jasper 9 producono
HTTP 500 "Unable to compile class for JSP". Le incompatibilità sono sintattiche: attributi HTML
con delimitatori doppi apici il cui valore contiene codice Java con doppi apici interni, e char
literal Java multi-carattere (`'id'` invece di `"id"`). La regola è: apici singoli esterni
all'attributo HTML, doppi apici interni al Java.

Tutti i file JSP si trovano in `/srv/getrad-stack/extracted/getrad/jsp/` su VM810.
La base dell'app è `/srv/getrad-stack/extracted/getrad/`.

### File già fixati

Transferiti da VM809 (Ubuntu 20.04) nello stato corretto — riferimento: backup
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

### File PENDING — errori noti, priorità alta

| File | Errore noto | Rotta di test |
|---|---|---|
| `fornitori/insupdfornitore.jsp` | righe 369, 370, 713, 714; `one.get('id')` a riga 672 | `/jsp/fornitori/insupdfornitore.jsp?id=13577496633860` |
| `preventivi/_combinazione.jsp` | HTTP 500, log sorgente in `HTTP500_combinazione_log.txt` (era su VM809, da ricreare su VM810) | `/jsp/preventivi/_combinazione.jsp` |
| `ordini/index.jsp` | line 211 col 56: apici su `multiLang.getLabel(id_lingua, "label.cerca")` | `/jsp/ordini/index.jsp` |
| `fatture/index.jsp` | via `fatture/index_tr.jspf` line 78 → `fatture/fatture_tr.jspf` line 479 | `/jsp/fatture/index.jsp` |
| `traduttori/banca.jsp` | mai toccato | — |
| `traduttori/emailtrad.jsp` | mai toccato | — |
| `provvigioni/index.jsp` | line 108 → via `provvigioni/provvigioni_env.jspf` line 20 | `/jsp/provvigioni/index.jsp` |
| `riepiloghi/index.jsp` | line 54 → via `riepiloghi/index_tr.jspf` line 86 | `/jsp/riepiloghi/index.jsp` |

### File candidati — mai toccati, errori latenti probabili

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

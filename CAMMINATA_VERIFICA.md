# Camminata di verifica funzionale getrad

> Piano di prova manuale per confermare che l'intera applicazione getrad sia funzionante dopo la
> migrazione dei JSP a Tomcat 9. Si esegue nel browser, da utente loggato, contro l'ambiente di
> test. Non e' una suite automatica: e' una lista di flussi reali da percorrere e spuntare. La
> colonna Esito si compila durante la camminata.

## Scopo e metodo

La migrazione del quoting JSP e' completa su test: delle 253 pagine dell'applicazione 229
compilano e non resta alcun errore di quoting. Compilare senza errori pero' non garantisce il
comportamento a runtime, perche' la compilazione Jasper verifica solo la sintassi, non la logica,
i dati e il rendering. Questa camminata serve a colmare quel divario percorrendo a mano i flussi
d'uso reali e osservando il risultato a video.

Si lavora sull'ambiente di test alla porta 8090, raggiungibile in LAN, partendo dal login
`http://192.168.20.90:8090/getrad/jsp/Login.jsp`. L'ambiente di test e' isolato dalla produzione
e seminato con dati reali di produzione, quindi le ricerche e gli elenchi mostrano dati veri ma
ogni modifica resta confinata al test. Il mailer e' disattivato sul test, quindi le azioni che
invierebbero una mail non producono invii reali. Durante la camminata non si promuove nulla in
produzione.

Per ogni flusso si annota se la pagina si apre senza errori, se i dati si vedono correttamente, se
le azioni di salvataggio e modifica vanno a buon fine, e se la stampa o l'esportazione produce il
documento atteso. In caso di errore HTTP 500 o di anomalia visiva si cattura uno screenshot e si
riporta la rotta esatta, cosi' da poter leggere il log Tomcat di test con
`docker compose -f /srv/getrad-stack/test/docker-compose.yml logs --tail=200 app`.

## Pagine note non raggiungibili da sole

Alcune pagine restano in errore se aperte da sole ma non sono difetti da segnalare durante la
camminata. Sono frammenti inclusi da altre pagine: funzionano quando si percorre il flusso che le
include, non aprendole con l'URL diretto. Esempi sono `preventivi/_combinazione.jsp`, i dettagli
ordine `ordini/altro.jsp`, `dettaglio.jsp`, `interpretariato.jsp` e le varianti `all_*`, i pannelli
a tab dei traduttori, vari frammenti `_*.jsp` e `*_env.jsp`. Vanno verificati attraverso la pagina
genitore, non isolatamente.

Restano inoltre quattro pagine reali con un problema preesistente non legato al quoting, lasciate
fuori dal patching in attesa di sapere se sono ancora in uso: `traduttori/comb_ling_complex.jsp`
e `traduttori/revisioni.jsp` usano variabili non definite secondo il contratto dei template, e
`preventivi/preventivo0.jsp` e `preventivo1.jsp` riferiscono il campo `GlobalDef.PRE_TEMPLATE` che
non esiste piu' nel codice compilato. Se durante la camminata un flusso normale porta a una di
queste pagine, va segnalato: significherebbe che sono ancora in uso e vanno affrontate con il
sorgente Java.

## Flussi trasversali

Questi controlli valgono per tutta l'applicazione e si verificano una volta sola.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Login | `jsp/Login.jsp` | accesso con credenziali valide, redirect alla home | |
| Credenziali errate | `jsp/Login.jsp` | messaggio di errore corretto, nessun accesso | |
| Logout | menu utente | uscita pulita e ritorno al login | |
| Cambio lingua | selettore lingua interfaccia | le etichette cambiano, nessun errore sulle pagine | |
| Ricerca globale Solr | `jsp/search/index.jsp` | i risultati si popolano dall'indice Solr di test | |
| Allegati | apertura di un allegato esistente da una scheda | il documento si apre (i file sono montati in sola lettura) | |
| Stampa o esportazione PDF | da preventivo, ordine, fattura | il PDF si genera e si scarica | |

## Clienti

Modulo ad alto utilizzo. La scheda cliente include molti sottoframmenti (contatti, commerciali,
gruppo, project manager di riferimento) che si esercitano aprendo e modificando la scheda.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco clienti | `jsp/clienti/index.jsp` | lista, filtri, ordinamento, ricerca per ragione sociale | |
| Apertura scheda cliente | da elenco, click su un cliente | la scheda carica anagrafica e tutte le tab | |
| Modifica e salvataggio cliente | scheda cliente, modifica un campo, salva | salvataggio senza errori, dato aggiornato | |
| Nuovo cliente | pulsante nuovo | form vuoto, salvataggio nuovo record | |
| Contatti cliente | tab contatti nella scheda | aggiunta, modifica, eliminazione contatto | |
| Commerciali e gruppo | tab relative nella scheda | i frammenti si popolano e si salvano | |
| Area cliente | `jsp/clienti/area/index.jsp` | accesso area, ordini, preventivi, fatture lato cliente | |

## Fornitori

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco fornitori | `jsp/fornitori/index.jsp` | lista, filtri, ricerca | |
| Scheda fornitore | `jsp/fornitori/insupdfornitore.jsp` | apertura, certificati versamenti inclusi, salvataggio | |
| Nuovo fornitore | pulsante nuovo | creazione e salvataggio | |

## Preventivi

La pagina di modifica preventivo include il frammento combinazioni `_combinazione.jsp`, che e' il
modo corretto di verificarlo.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco preventivi | `jsp/preventivi/index.jsp` | lista, filtri, stati | |
| Apertura preventivo | da elenco | la scheda carica, le combinazioni linguistiche si vedono | |
| Modifica combinazioni | dentro la scheda preventivo | aggiunta e modifica combinazione servizio e lingue | |
| Nuovo preventivo | pulsante nuovo | creazione, salvataggio | |
| Preventivo da lead | `jsp/preventivi/insupdpreventivolead.jsp` | flusso preventivo collegato a lead | |
| Stampa preventivo | dalla scheda | generazione PDF corretta | |

## Ordini

La scheda ordine include i frammenti dettaglio, altro e interpretariato, che si esercitano qui.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco ordini | `jsp/ordini/index.jsp` | lista, filtri, ricerca | |
| Apertura e modifica ordine | `jsp/ordini/insupdordine.jsp` | tab dettaglio, altro, interpretariato, allegati, salvataggio | |
| Nuovo ordine | pulsante nuovo o da preventivo | creazione e salvataggio | |
| Revisioni ordine | dalla scheda ordine | la gestione revisioni si apre e funziona | |

## Fatture

Il modulo si dirama in fatture cliente, fornitore e PA. Le pagine si differenziano per tipo utente
e includono lunghe catene di frammenti.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco fatture cliente | `jsp/fatture/index.jsp` | lista, filtri per stato, ricerca per cliente | |
| Elenco fatture fornitore | `jsp/fatture/index_forn.jsp` | lista lato fornitore | |
| Modifica fattura | dalla lista, apertura fattura | dati, righe, salvataggio | |
| Accorpamento e avanzamento | azioni dalla lista | i flussi di accorpa e avanzamento si aprono | |
| Fattura PA | flusso fattura elettronica | la generazione e la popup PA funzionano | |
| Stampa fattura | dalla scheda | PDF corretto | |

## Pagamenti

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco pagamenti | `jsp/pagamenti/index.jsp` | lista, filtri | |
| Pagamento parziale | azione dalla lista | il flusso parziale cliente e fornitore funziona | |
| Storia pagamenti | dalla scheda | la storia si apre | |
| Note di credito | flusso relativo | apertura e salvataggio | |

## Provvigioni e riepiloghi

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco provvigioni | `jsp/provvigioni/index.jsp` | lista, creazione eccezione, generazione pagamento | |
| Riepiloghi | `jsp/riepiloghi/index.jsp` | generazione riepilogo, varianti traduttore e cliente | |

## Traduttori

Modulo ampio con interfaccia a tab. La pagina principale e' `autenticazione.jsp` per i traduttori e
le home per PM e traduttore. I pannelli profilo, competenze, combinazione, traduttore sono tab
incluse e si verificano navigando, non da URL diretto.

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco traduttori | `jsp/traduttori/index.jsp` | lista, ricerca, filtri | |
| Home traduttore | `jsp/traduttori/home_trad.jsp` | dashboard traduttore | |
| Home PM | `jsp/traduttori/home_pm.jsp` | dashboard project manager | |
| Scheda e tab traduttore | da elenco, apertura scheda | anagrafica, profilo, competenze, combinazioni linguistiche, banca, settori | |
| Combinazioni linguistiche | tab combinazioni | apertura editor combinazioni, salvataggio | |
| Email traduttore | `jsp/traduttori/emailtrad.jsp` | la pagina rende, nessun invio reale (mailer off) | |

## Lead

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Elenco e dashboard lead | `jsp/lead/index.jsp` | lista, dashboard, ricerca | |
| Scheda lead | `jsp/lead/insupdlead.jsp` | anagrafica, contatti, commerciali, salvataggio | |
| Attivita lead | `jsp/lead/insupdattivita.jsp` | creazione e modifica attivita | |
| Note e storico | dalle schede lead | note e storico attivita si aprono | |
| Associazione e conversione | flussi di associazione lead e accettazione preventivo | funzionano end to end | |

## Altri moduli

| Flusso | Come arrivarci | Cosa verificare | Esito |
|---|---|---|---|
| Reclami | `jsp/reclami/index.jsp`, `insupdreclamo.jsp` | elenco, apertura, salvataggio | |
| Societa | `jsp/societa/insupdsocieta.jsp` | scheda societa, conti correnti, salvataggio | |
| Utenti | `jsp/utenti/index.jsp` | elenco utenti, creazione e modifica utente | |
| Vipa | `jsp/vipa/index.jsp`, `insupdvipa.jsp` | elenco e scheda | |
| Listino | `jsp/listino/index.jsp`, `insupdlistino.jsp` | elenco e scheda listino | |
| Servizi | `jsp/servizi/index.jsp`, `insupdservizio.jsp` | elenco e scheda servizio | |
| Numerazioni | `jsp/numerazioni/index.jsp` | gestione numerazioni | |
| Transcodifiche | `jsp/transcod/index.jsp` | elenco e nuovo tipo | |
| Tabelle base | `jsp/base/index.jsp`, `insupdtabelle.jsp` | gestione tabelle di base | |
| Progetti | `jsp/progetti/home_progetti.jsp` | dashboard progetti | |
| Tracking | `jsp/tracking/index.jsp` | pagina tracking | |
| Allegati | `jsp/allegati/allega.jsp` | apertura form allegati da una scheda | |

## Esito e tracciamento

Al termine della camminata, i flussi con esito negativo si raccolgono come elenco di rotte da
correggere, ciascuna con la rotta esatta e lo screenshot. Le correzioni si applicano sull'albero di
test, si riverificano sulla stessa rotta, e quando l'intero piano e' verde si procede alla
promozione in produzione con `test/promote-to-prod.sh`. L'esito complessivo della camminata va
registrato in `.claude/memory/progress.md` come voce di work-log, con la data e il riferimento allo
stato di test verificato.

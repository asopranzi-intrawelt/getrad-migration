# Registro delle decisioni architetturali

> Convenzione ADR-lite, append-only. Ogni decisione architetturale non ovvia entra come voce
> numerata con data, stato, contesto, decisione, motivazione e conseguenze. Una decisione non si
> cancella e non si riscrive: quando viene superata, si aggiunge una nuova voce che dichiara di
> superare la precedente e ne cita il numero. Le inferenze non confermate si marcano come da
> verificare e si promuovono a decisione solo quando una fonte le conferma.

## ADR-001 — Adozione del sistema di progetto portabile

Data: 2026-06-19
Stato: accettata
Contesto: il progetto necessita di uno stato interamente recuperabile da un clone e di
documentazione che resti allineata al codice senza rilettura integrale a ogni sessione.
Decisione: adottare il sistema descritto in `.claude/PROJECT-SYSTEM.md`, con motore di
riconciliazione ancorato ai commit e doppio livello documentale tracciato/ignorato.
Motivazione: persistenza strutturale su disco indipendente dalla sessione di chat, e controllo
umano sul versionamento.
Conseguenze: ogni passo significativo aggiorna schede, `last-verified-commit`, snapshot e
work-log; commit e push restano manuali.

## ADR-002 — Docker compose come runtime per lo stack getrad su Ubuntu 24.04

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: Ubuntu 24.04 LTS non include più OpenJDK 8 nei repository ufficiali. L'applicazione
getrad richiede Java 8 per la compilazione JSP (compatibilità Jasper + JDBC driver 3.0.11).
Installare Java 8 manualmente richiederebbe la gestione di un repository non standard.
Decisione: containerizzare con Docker compose, usando `tomcat:9-jdk8` come immagine base per
il container app e `mariadb:10.3` per il DB. I tag sono pinned alla versione minore.
Motivazione: Docker isola il runtime dal sistema host, elimina il conflitto di dipendenza con
il JDK, e fissa le versioni in modo dichiarativo nel Dockerfile.
Conseguenze: ogni modifica al Dockerfile richiede `docker compose build --no-cache app`. Le
modifiche ai file JSP (bind mount) richiedono solo restart app.

## ADR-003 — MariaDB 10.3.x pinned (no upgrade)

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: il JDBC driver mysql-connector-java 3.0.11 è embedded in `WEB-INF/lib` e non è
aggiornabile senza accesso al sorgente dell'app. Non è compatibile con il protocollo di
autenticazione di MariaDB 10.4+ né con MySQL 8.
Decisione: usare l'immagine `mariadb:10.3` (effettivamente 10.3.39).
Motivazione: la compatibilità del driver impone il vincolo di versione. Un upgrade richiederebbe
la sostituzione del driver, che richiede il sorgente dell'app.
Conseguenze: il DB non riceve upgrade di versione. Security patch solo a livello di OS host e
Docker engine.

## ADR-004 — log4j MAIL appender disabilitato

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: la riga 2 di `getrad.properties` originale era `log4j.logger.ApplicationLogger=INFO,
APP, MAIL`. Il SMTPAppender tentava di connettersi a `smtp.office365.com:25` ad ogni log event.
Dal container Docker, smtp.office365.com non è raggiungibile (timeout TCP): ogni pagina
generava un ritardo di 10-30 secondi.
Decisione: rimuovere `MAIL` dall'appender: `log4j.logger.ApplicationLogger=INFO, APP`.
Motivazione: eliminazione diretta del timeout senza impatto funzionale sul gestionale.
Conseguenze: nessun alert mail automatico dall'app. Se serve, reintrodurre con relay SMTP
raggiungibile dal container.

## ADR-005 — NetworkManager mantenuto su VM810 (non sostituito con systemd-networkd)

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: durante la bonifica di VM810 si è valutata la sostituzione di NetworkManager con
systemd-networkd più leggero.
Decisione: mantenere NetworkManager.
Motivazione: il guadagno in peso è minimo rispetto al rischio di perdere la connettività di
rete durante la configurazione su una VM senza console diretta comoda.
Conseguenze: nessuna rilevante.

## ADR-006 — Solr 3.5 mantenuto nella stessa webapp Tomcat (non isolato in container proprio)

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: Solr 3.5.0 gira come webapp dentro lo stesso Tomcat di getrad. Alternativa: container
Tomcat dedicato per Solr.
Decisione: mantenere Solr dentro il container `getrad-app`.
Motivazione: il gestionale ha traffico basso e consultazione prevalentemente statica.
L'isolamento aggiunge un container, modifica l'URL Solr nell'app, e aumenta la complessità
operativa senza benefici apprezzabili.
Conseguenze: un crash di Solr porta giù anche l'app getrad. Accettabile per il profilo d'uso.

# Registro delle decisioni architetturali

> Convenzione ADR-lite, append-only. Ogni decisione architetturale non ovvia entra come voce
> numerata con data, stato, contesto, decisione, motivazione e conseguenze. Una decisione non si
> cancella e non si riscrive: quando viene superata, si aggiunge una nuova voce che dichiara di
> superare la precedente e ne cita il numero. Le inferenze non confermate si marcano come da
> verificare e si promuovono a decisione solo quando una fonte le conferma.

## ADR-001 - Adozione del sistema di progetto portabile

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

## ADR-002 - Docker compose come runtime per lo stack getrad su Ubuntu 24.04

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

## ADR-003 - MariaDB 10.3.x pinned (no upgrade)

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

## ADR-004 - log4j MAIL appender disabilitato

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

## ADR-005 - NetworkManager mantenuto su VM810 (non sostituito con systemd-networkd)

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: durante la bonifica di VM810 si è valutata la sostituzione di NetworkManager con
systemd-networkd più leggero.
Decisione: mantenere NetworkManager.
Motivazione: il guadagno in peso è minimo rispetto al rischio di perdere la connettività di
rete durante la configurazione su una VM senza console diretta comoda.
Conseguenze: nessuna rilevante.

## ADR-006 - Solr 3.5 mantenuto nella stessa webapp Tomcat (non isolato in container proprio)

Data: 2026-05 (data esatta non verificata)
Stato: accettata
Contesto: Solr 3.5.0 gira come webapp dentro lo stesso Tomcat di getrad. Alternativa: container
Tomcat dedicato per Solr.
Decisione: mantenere Solr dentro il container `getrad-app`.
Motivazione: il gestionale ha traffico basso e consultazione prevalentemente statica.
L'isolamento aggiunge un container, modifica l'URL Solr nell'app, e aumenta la complessità
operativa senza benefici apprezzabili.
Conseguenze: un crash di Solr porta giù anche l'app getrad. Accettabile per il profilo d'uso.

## ADR-007 - Ambiente di test affiancato alla produzione sullo stesso host

Data: 2026-06-19
Stato: accettata
Contesto: il patching dei file JSP per compatibilità Jasper su Tomcat 9 veniva verificato
direttamente contro la produzione, l'unico ambiente disponibile, perché ogni rotta corretta
andava provata sull'istanza viva. Questo espone gli utenti agli errori introdotti da una patch
non ancora validata. Si vuole servire sulla LAN sia un ambiente di test per verificare le
modifiche sia quello di produzione per l'uso normale, partendo dallo stesso host VM810.
Decisione: affiancare alla produzione un secondo progetto compose isolato, di nome
`getrad-test`, radicato in `/srv/getrad-stack/test/`, con container `getrad-test-app` e
`getrad-test-db`, porta pubblicata 8090, rete dedicata e datadir DB proprio. La produzione resta
intatta sul progetto `getrad-stack` alla radice, porta 8080. L'ambiente di test è puramente
additivo: nessuna modifica alla produzione.
Motivazione: i riferimenti interni dell'applicazione sono relativi al container, non all'host.
La connessione al database passa per l'alias di rete `db` e Solr è interrogato su `localhost`
dentro lo stesso Tomcat. Chiamando `db` anche il servizio MariaDB del progetto di test e
mantenendo i percorsi interni `/usr/skn/getrad` e `/usr/skn/solr`, lo stesso `getrad.properties`
funziona immutato puntando al database di test, e l'unica differenza osservabile dall'esterno è
la porta. Il container app di test riusa l'immagine già costruita per la produzione
(`getrad-stack-app:latest`), garantendo runtime identico senza una seconda build.
Conseguenze: il flusso di patching diventa modifica e verifica su test alla porta 8090, poi
promozione in produzione tramite `test/promote-to-prod.sh`, che salva la versione di produzione
corrente prima di sovrascrivere e forza la ricompilazione Jasper. Il consumo di disco aggiuntivo
è di circa dieci gibibyte. Il database di test va riseminato dal dump di produzione quando serve
ripartire da dati freschi.

## ADR-008 - Scelte di fedeltà e isolamento dell'ambiente di test

Data: 2026-06-19
Stato: accettata
Contesto: l'ambiente di test introdotto in ADR-007 deve replicare la produzione in modo
sufficientemente fedele da riprodurre i bug, restando isolato così da non poter corrompere i
dati di produzione. I due alberi voluminosi sono gli allegati utente `files/` (quindici
gibibyte) e l'indice Solr (nove gibibyte e mezzo), mentre il codice applicativo vero e proprio
pesa circa duecentonovanta mebibyte.
Decisione: il codice è copiato a parte in `/srv/getrad-stack/test/extracted/getrad/`, così che
le modifiche ai JSP siano isolate. L'indice Solr è copiato per intero, per dare a test una
ricerca funzionante e indipendente. Gli allegati `files/` sono montati dalla produzione in sola
lettura, evitando la duplicazione di quindici gibibyte e impedendo per costruzione qualsiasi
scrittura sui dati di produzione. Il database di test nasce da inizializzazione fresca
dell'immagine MariaDB, che ricrea l'utente `getrad@'%'` con gli stessi privilegi e password
della produzione, seguita dal restore del dump logico pre-patching: si evita così la copia a
caldo del datadir InnoDB, che sarebbe incoerente, e non si tocca la produzione.
Motivazione: separare il rischio dal volume. Il rischio sta nel codice e nei dati scrivibili, il
volume sta negli allegati statici. Copiare solo ciò che cambia e montare in sola lettura ciò che
non deve cambiare massimizza la fedeltà e l'isolamento al minimo costo di disco.
Conseguenze: in test gli upload di nuovi allegati falliscono perché `files/` è in sola lettura,
comportamento atteso e desiderato. Un nuovo allegato caricato in produzione diventa visibile
anche in test senza azioni. Se in futuro test dovesse poter caricare allegati, si passerebbe a
una copia scrivibile dedicata.

## ADR-009 - Mailer applicativo disattivato in test, da disattivare anche in produzione

Data: 2026-06-19
Stato: accettata
Contesto: oltre al `MAIL` appender di log4j già rimosso in ADR-004, l'applicazione ha un mailer
applicativo proprio, configurato in `getrad.properties` con `mail.host=smtp.office365.com` su
porta 587 autenticata, che invia messaggi reali a destinatari reali, per esempio le notifiche di
preventivo a `preventivi@intrawelt.com`, su determinate azioni utente. La rete bridge del
container ha uscita verso internet, quindi un ambiente di test potrebbe inviare mail reali a
clienti veri. È inoltre confermato che questo gestionale non deve più inviare mail.
Decisione: nell'albero di test il mailer è disattivato in modo definitivo: `mail.host` e
`log4j.appender.MAIL.SMTPHost` puntati su `127.0.0.1` (nessun server SMTP[^1] in ascolto nel
container, connessione rifiutata), `mail.autentication` a zero e password Office365 rimossa
dall'albero di test, così che il segreto non sia nemmeno presente nella copia.
Motivazione: impedire per costruzione qualsiasi invio di mail dall'ambiente di test e ridurre la
superficie del segreto. Una connessione rifiutata fallisce subito, senza il ritardo di un
timeout.
Conseguenze: in test nessuna mail lascia il sistema. Resta da applicare la stessa disattivazione
alla produzione e da ruotare la password Office365 viva, che è riutilizzata su altri sistemi:
questa parte tocca la configurazione di produzione e va eseguita come passo confermato a parte,
non è inclusa nella predisposizione dell'ambiente di test.

[^1]: *SMTP*, Simple Mail Transfer Protocol - protocollo di invio della posta elettronica.

## ADR-010 - Allowlist di rete in DOCKER-USER, non in UFW

Data: 2026-06-19
Stato: accettata
Contesto: si vuole restringere l'accesso al gestionale ai soli IP statici autorizzati della LAN
(Fase 6 della roadmap). La via istintiva sarebbe riattivare UFW con regole sulle porte 8080 e 8090.
Verifica sul campo: Docker pubblica le porte inserendo regole iptables proprie nella catena di
forwarding e bypassa il filtro INPUT di UFW, quindi una regola UFW sulle porte pubblicate non
filtra il traffico verso i container. Inoltre `userland-proxy` e' attivo e dopo il DNAT entrambi i
container risultano in ascolto sulla 8080 interna, per cui non si puo' distinguere prod e test
guardando la porta di destinazione post-DNAT.
Decisione: applicare l'allowlist nella catena `DOCKER-USER`, prevista da Docker proprio per le
regole utente sul traffico verso i container, filtrando sulla porta di destinazione ORIGINALE
(pre-DNAT) tramite il modulo conntrack `ctorigdstport`. Le regole stanno in uno script idempotente
reso persistente da un servizio systemd che gira dopo Docker. UFW resta inattivo.
Motivazione: e' l'unico punto in cui il filtro e' effettivo per le porte pubblicate da Docker;
`ctorigdstport` permette di distinguere produzione (8080) e test (8090) nonostante il DNAT verso la
stessa porta interna; il RemoteAddrValve di Tomcat e' stato scartato come meccanismo primario
perche' con `userland-proxy` attivo l'IP visto dall'app non e' affidabile.
Conseguenze: le modifiche all'elenco IP si fanno nello script `getrad-firewall.sh` e si riapplicano
con un restart del servizio. La porta SSH non e' coinvolta, quindi non c'e' rischio di lockout
dall'accesso amministrativo. La verifica empirica del blocco richiede una prova da una postazione
non autorizzata. Se in futuro servisse l'IP reale del client a livello applicativo, andrebbe
disattivato `userland-proxy`.

## ADR-011 - Nome amichevole egetrad via reverse proxy nginx su porta 80

Data: 2026-06-19
Stato: accettata
Contesto: gli utenti di produzione dovevano poter raggiungere il gestionale con un nome
amichevole invece di IP e porta 8080, e idealmente una rotta di login facile da digitare. I PC
client sono tutti Windows 11. Non esiste un DNS di LAN gestibile internamente, e
`egetrad.intrawelt.com` risolve a un IP pubblico esterno.
Decisione: introdurre un reverse proxy nginx sulla porta 80, come progetto compose separato e
additivo agganciato alla rete della produzione, che reindirizza radice e `/egetrad-login` al login
e proxa il resto verso `getrad-app:8080`. La risoluzione del nome `egetrad` ed `egetrad-login`
avviene via file hosts sui PC autorizzati, data l'assenza di un DNS interno e il numero ridotto di
postazioni. La porta 80 entra nell'allowlist del firewall con gli stessi IP della produzione.
Motivazione: il proxy separato lascia intatto il compose di produzione ed è completamente
reversibile; il filtro su nome amichevole e porta 80 migliora l'usabilità senza esporre nulla in
piu', perche' la 80 e' ristretta come la 8080; il file hosts e' la via piu' affidabile senza un DNS
interno, con pochi PC. Il proxy predispone anche il punto naturale dove terminare HTTPS in Fase 7.
Conseguenze: per cambiare gli host che accedono o aggiungere nomi si modificano hosts dei client e,
se serve, il `server_name` in `nginx.conf`. Resta il caveat legacy delle URL assolute che l'app
genera verso `egetrad.intrawelt.com:8080`, da risolvere mappando anche quel nome sui client o
riscrivendo `getrad.properties` se quelle funzioni daranno problemi sulla LAN.

## ADR-012 - Ambito del repository e modello di allineamento agli ambienti

Data: 2026-06-19
Stato: accettata
Contesto: ci si e' chiesti a quale ambiente corrisponda cio' che e' pushato su GitHub, test o
produzione, e se si usino git worktree o branch per ambiente. La domanda nasce dal confronto con
progetti dove il codice sta in git e un branch o un worktree mappa un ambiente di deploy.
Decisione: il repository `getrad-migration` versiona soltanto documentazione, ovvero il dossier di
migrazione e operativo, le schede `.claude/` (sistema di progetto, memory, context), README,
HANDOFF, `getrad.properties.example` e il piano di camminata. Non versiona il codice
dell'applicazione ne' i dati. Gli ambienti non sono gestiti con git: test e produzione sono due
alberi di directory sul server VM810, rispettivamente `/srv/getrad-stack/test/extracted/getrad` e
`/srv/getrad-stack/extracted/getrad`. Non esistono worktree ne' branch per ambiente.
Motivazione: l'applicazione e' legacy e non se ne possiede il sorgente Java, solo i jar compilati e
i JSP; gli alberi pesano decine di gigabyte fra allegati e indice Solr; `getrad.properties`
contiene segreti. Versionare il codice sarebbe impraticabile e insicuro. Il deliverable di valore e
portabile e' la documentazione.
Conseguenze: il repo, in termini di codice, non e' allineato ne' a test ne' a produzione, perche'
non contiene un artefatto deployabile; la sua documentazione e' pero' centrata sulla produzione, con
il test descritto come aggiunta. Esistono due sorgenti di verita' distinte: per il codice e i dati
e' il filesystem del server, con promozione test verso produzione via rsync per contenuto e
ripristino dai backup `.tar.zst` in `/srv/getrad-stack/backups/`; per lo stato del lavoro e le
decisioni e' il repo git pushato su GitHub. Per sapere dove si e' arrivati si legge
`.claude/memory/index.md`; per ripristinare il codice si usano gli alberi sul server e i backup, non
un checkout git. Salvare lo stato su GitHub significa committare e pushare le schede aggiornate, non
il codice. Commit e push restano manuali dell'utente.
Alternativa scartata: codice in git con worktree o branch per ambiente. Adatta a progetti con
sorgente proprio e artefatti leggeri, non a questo caso legacy senza sorgente e con dati pesanti e
segreti.

## ADR-013 - Hardening applicato per livello: web.xml dell'app contro config del server

Data: 2026-06-19
Stato: accettata
Contesto: la Fase 7 prevede hardening da fare su test e poi promuovere. Gli interventi cadono su
livelli diversi dello stack, con percorsi di promozione diversi, e va deciso dove mettere ciascuno.
Decisione: gli header di sicurezza HTTP si aggiungono nel `web.xml` dell'applicazione, con il
filtro nativo Tomcat `HttpHeaderSecurityFilter`. Il mascheramento della versione Tomcat nelle
pagine di errore si fa con un `ErrorReportValve` nel `server.xml`, montato in sola lettura nel solo
container di test.
Motivazione: il `web.xml` vive nell'albero dell'app bind-montato, quindi una modifica e' isolata al
test e si promuove in produzione con lo stesso rsync usato per i JSP, restando coerente con il
modello test verso produzione. Il `server.xml` e' invece configurazione del server, fuori
dall'albero dell'app: per isolarlo al test si monta una copia modificata solo sul compose di test.
Conseguenza: la promozione e' asimmetrica. Il `web.xml` con gli header arriva in produzione con la
prossima sincronizzazione dell'albero (sta in `WEB-INF/`, va incluso nella rsync di promozione,
non solo `jsp/`). Il `server.xml` invece non si promuove via rsync: in produzione va montato lo
stesso file o cotta la modifica nel Dockerfile, come passo dedicato. Per il debito MD5, non
risolvibile senza sorgente, il controllo compensativo resta l'allowlist di rete della Fase 6.

## ADR-014 - HTTPS sulla LAN non adottato

Data: 2026-06-24
Stato: accettata
Contesto: la Fase 7 prevedeva di valutare HTTPS sulla LAN per proteggere le credenziali in transito,
dato che il login usa hash MD5 e il dato e' sensibile. L'impianto proposto era TLS terminato sul
reverse proxy nginx con certificato self-signed importato sui sei PC.
Decisione: non si adotta HTTPS sulla LAN, per ora.
Motivazione: l'esposizione e' gia' fortemente ridotta da controlli a monte. Il firewall ammette solo
sei IP statici della LAN verso il gestionale; il login e' ristretto a sei account interni; e' uno
strumento interno legacy su rete commutata, senza esposizione esterna. Il rischio residuo, lo
sniffing del traffico da parte di un attaccante gia' interno alla LAN (per esempio via ARP
spoofing), e' basso e viene accettato. Il costo, certificato self-signed con import nello store di
fiducia di ogni PC e riconfigurazione dei cookie, non e' giustificato a fronte di questo rischio.
Conseguenza: il flag `secure` sui cookie, che richiede HTTPS, resta non applicabile; l'eventuale
irrigidimento dei cookie si limiterebbe a `httponly`. La decisione e' rivedibile se cambia
l'esposizione, per esempio se il servizio dovesse diventare raggiungibile fuori dalla LAN.

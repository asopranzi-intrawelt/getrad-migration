---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - HANDOFF_getrad.md
last-verified-commit: 4f686bf
---

# Roadmap

## Direzione

Rendere operativi tutti i moduli dell'applicazione getrad su Tomcat 9 (Fase 5), poi
applicare l'hardening di sicurezza Tomcat secondo OWASP (Fase 6). Lo stack è già stabile
e in produzione; i lavori sono incrementali e non richiedono downtime prolungato.

## Priorità

**1. Fase 5 - completamento patching JSP** (in corso, priorità alta)

Otto file con errori noti e circa 24 file mai toccati con errori latenti probabili.
Ordine di lavoro suggerito: `fornitori/insupdfornitore.jsp` (modulo ad alto utilizzo, errori
alle righe 369, 370, 672, 713, 714), poi `ordini/index.jsp`, `fatture/index.jsp`,
`provvigioni/index.jsp`, `riepiloghi/index.jsp`, poi `traduttori/` e `lead/`, infine i file
della lista candidati mai toccati.

Prima di ogni sessione di patching: backup ad-hoc dell'intero albero webapp con il comando
documentato in `deployment.md` e in `HANDOFF_getrad.md` (sezione Note operative per Claude
Code).

**2. Fase 6 - restrizione dell'accesso ai soli IP autorizzati della LAN** (FATTA 2026-06-19, vedi `deployment.md` e ADR-010)

Realizzata con allowlist in `DOCKER-USER` resa persistente da `getrad-firewall.service`: produzione
8080 ai quattro IP autorizzati, test 8090 alla sola postazione di amministrazione. Resta da fare la
verifica empirica del blocco da una postazione non autorizzata. Descrizione storica della scelta di
seguito.

Si vuole irrobustire l'esposizione del servizio consentendo l'accesso soltanto a un insieme
definito di macchine della LAN, sfruttando il fatto che in rete gli host hanno IP statici.
L'allowlist va applicata sia alla produzione (porta 8080) sia all'ambiente di test (porta 8090).
Tre livelli possibili, da valutare e combinare: a livello di host con UFW, che al momento
risulta inattivo e andrebbe riattivato con regole che ammettono solo gli IP o la sottorete
autorizzati verso 8080 e 8090; a livello di Docker, pubblicando le porte solo su interfacce o
indirizzi specifici invece che su `0.0.0.0`; a livello applicativo con un RemoteAddrValve di
Tomcat nel context, che filtra per indirizzo remoto a prescindere dal firewall. Il filtro di
rete con UFW o publish mirato è il più robusto perché blocca prima che la richiesta raggiunga
Tomcat; il valve Tomcat è una difesa in profondità aggiuntiva. Da raccogliere preventivamente
l'elenco degli IP statici delle macchine che devono accedere.

**3. Fase 7 - hardening per stack legacy con dati sensibili** (dopo Fase 5, in parte parallela a Fase 6)

Lo stack è datato e custodisce dati sensibili (anagrafiche, fatture, dati economici), quindi
l'hardening va oltre il filtro di rete. Punti previsti, da dettagliare quando si affronta la
fase: hardening Tomcat secondo OWASP (riferimento https://wiki.owasp.org/index.php/Securing_tomcat,
non ancora consultato) con rimozione delle webapp manager/host-manager di default,
disabilitazione del directory listing, header HTTP di sicurezza e audit dei permessi del
filesystem nel container; introduzione di HTTPS anche sulla sola LAN, giustificata dalla
sensibilità dei dati e non più rimandata alla sola eventualità di esposizione esterna; cifratura
e restrizione dei permessi dei backup, che oggi contengono in chiaro sia i dati sia i segreti
applicativi presenti in `getrad.properties` e nelle sue copie storiche; disattivazione definitiva
del mailer applicativo, coerente con ADR-009 e con l'indicazione che il gestionale non deve più
inviare mail; rotazione della password Office365 tuttora viva; protezione del login contro
tentativi a forza bruta; irrigidimento di sessione e cookie. La bonifica delle copie storiche di
`getrad.properties` con segreti vivi sotto `WEB-INF/conf/` va trattata come debito di sicurezza.
Ulteriore debito emerso dall'audit del 2026-06-19: le password degli utenti in `an_utenti` sono
hash MD5 a 32 cifre, algoritmo debole e probabilmente non salato, con alcune password mai cambiate
da molti anni. Il rafforzamento dell'hashing richiede di toccare il codice di login Java e va
pianificato insieme a una rotazione delle password piu' vecchie.

**4. Test pratico disaster recovery** (opzionale, nessuna scadenza)

Il restore from-scratch non è mai stato testato. Da pianificare su host vergine Ubuntu 24.04.
La procedura è documentata in `README.md` (sezione Disaster recovery).

## Idee e ipotesi da verificare

Scorporo Solr in container separato: tecnicamente fattibile, non urgente per un gestionale
a consultazione statica con traffico basso. Da valutare solo se si presenta un problema di
stabilità imputabile a Solr.

Rotazione password Office365: da fare in produzione, indipendentemente dallo sviluppo
applicativo. Non blocca la roadmap tecnica ma è debito di sicurezza aperto.

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

**1. Fase 5 — completamento patching JSP** (in corso, priorità alta)

Otto file con errori noti e circa 24 file mai toccati con errori latenti probabili.
Ordine di lavoro suggerito: `fornitori/insupdfornitore.jsp` (modulo ad alto utilizzo, errori
alle righe 369, 370, 672, 713, 714), poi `ordini/index.jsp`, `fatture/index.jsp`,
`provvigioni/index.jsp`, `riepiloghi/index.jsp`, poi `traduttori/` e `lead/`, infine i file
della lista candidati mai toccati.

Prima di ogni sessione di patching: backup ad-hoc dell'intero albero webapp con il comando
documentato in `deployment.md` e in `HANDOFF_getrad.md` (sezione Note operative per Claude
Code).

**2. Fase 6 — hardening Tomcat secondo OWASP** (dopo completamento Fase 5)

Riferimento da consultare: https://wiki.owasp.org/index.php/Securing_tomcat (non ancora
consultato). Punti attesi: rimozione webapp manager/host-manager di default, disabilitazione
directory listing, header HTTP di sicurezza, audit permessi filesystem container, considerare
HTTPS se l'app venisse esposta fuori dalla LAN.

**3. Test pratico disaster recovery** (opzionale, nessuna scadenza)

Il restore from-scratch non è mai stato testato. Da pianificare su host vergine Ubuntu 24.04.
La procedura è documentata in `README.md` (sezione Disaster recovery).

## Idee e ipotesi da verificare

Scorporo Solr in container separato: tecnicamente fattibile, non urgente per un gestionale
a consultazione statica con traffico basso. Da valutare solo se si presenta un problema di
stabilità imputabile a Solr.

Rotazione password Office365: da fare in produzione, indipendentemente dallo sviluppo
applicativo. Non blocca la roadmap tecnica ma è debito di sicurezza aperto.

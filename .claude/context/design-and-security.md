---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - HANDOFF_getrad.md
  - README.md
last-verified-commit: 4f686bf
---

# Design e sicurezza applicativa

## Paradigmi di software design

Architettura MVC-ish anni 2000: le pagine JSP fungono da view e parzialmente da controller, le
classi Servlet gestiscono le azioni (inserimento, modifica, cancellazione). La tag library
custom `skn:*` (es. `skn:bottone`, `skn:lista`) genera markup HTML e governa i comportamenti UI.
La logica di business è distribuita tra JSP e classi Java nei JAR di `WEB-INF/lib`.

Non esiste ORM, DI container, né separazione formale tra layer. Il codice è monolitico,
compreso in una singola webapp `getrad`. L'internazionalizzazione è gestita dalla classe
`multiLang`, caricata in ogni JSP con `<jsp:useBean>` e usata come
`multiLang.getLabel(id_lingua, "chiave")`.

## Sicurezza applicativa

Autenticazione: form-based, sessione HTTP. L'URL legacy su Tomcat 6 passava credenziali nel
query string (`/getrad/Login?username=...&password=...`). Su Tomcat 9 la rotta è
`/getrad/jsp/Login.jsp`. [Da verificare: il Servlet di autenticazione usa POST o GET verso
il backend].

Gestione segreti: `getrad.properties` contiene password DB e SMTP in chiaro. Il file è
gitignored e vive solo sulla VM810. Non è cifrato a riposo.

Superficie esposta: porta 8080 su LAN (UFW allow 8080), nessun HTTPS, nessun reverse proxy.
Il traffico è in chiaro sulla rete interna.

Debito di sicurezza attivo:

- Password SMTP Office365 in chiaro nel file di configurazione, riutilizzata altrove, da ruotare.
- Nessun HTTPS. Se l'app diventasse accessibile fuori dalla LAN, obbligatorio.
- Header HTTP di sicurezza assenti (X-Content-Type-Options, X-Frame-Options, ecc.).
- Webapp manager e host-manager di Tomcat: stato da verificare (OWASP raccomanda la rimozione).
- Directory listing Tomcat: stato da verificare.
- Password DB `getradpwd`: da considerare rotazione dopo stabilizzazione.

Lavoro futuro pianificato: hardening secondo OWASP Tomcat Securing Guide (Fase 6, dopo
completamento patching JSP). Riferimento: https://wiki.owasp.org/index.php/Securing_tomcat
(non ancora consultato, contenuto da leggere e applicare).

## Diagrammi

Nessun diagramma formale. La topologia di rete è descritta in `README.md` (sezione
Architettura) e in `HANDOFF_getrad.md` (sezione Architettura attuale).

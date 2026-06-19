---
generated-from-commit: 4f686bf
generated-from-branch: main
generated-date: 2026-06-19
covers-paths:
  - GETRAD_TOMCAT_MIGRATION.md
  - HANDOFF_getrad.md
last-verified-commit: 4f686bf
---

# Test di sviluppo

Non esiste una suite di test automatizzata. La verifica è manuale: modificare il file JSP sul
bind mount dell'ambiente di test, forzare la ricompilazione Jasper, accedere alla rotta via
browser e leggere i log Tomcat in caso di errore HTTP 500. Dal 2026-06-19 le modifiche si
verificano prima sull'ambiente di test (porta 8090) e solo dopo si promuovono in produzione
(porta 8080); vedi ADR-007 in `memory/decisions.md`. La produzione non è più il banco di prova.

## Test runner e comandi

Ciclo completo di patch su test, poi promozione in produzione su VM810:

```bash
# 1. Backup pre-patching (una volta per campagna di patch): DB + albero codice, esclusi i
#    quindici gibibyte di files/ e il tmp/ rigenerabile.
TAG=pre-patching-$(date +%Y-%m-%d)
BACKUP_DIR=/srv/getrad-stack/backups
docker exec getrad-db mysqldump --single-transaction --routines --triggers --events getrad \
  | zstd -T0 -3 -o ${BACKUP_DIR}/getrad-db-${TAG}.sql.zst
sudo tar --numeric-owner --exclude=files --exclude=tmp -cf - -C /srv/getrad-stack/extracted getrad \
  | zstd -T0 -3 -o ${BACKUP_DIR}/getrad-code-${TAG}.tar.zst
cd ${BACKUP_DIR} && sha256sum getrad-db-${TAG}.sql.zst getrad-code-${TAG}.tar.zst | sudo tee SHA256SUMS-${TAG}
# Nota: il dump si esegue senza -u/-p, così mysqldump si connette come root@localhost via
# socket. Forzare l'utente getrad fa fallire --routines (niente privilegi su mysql.proc).

# 2. Modifica il file JSP sull'albero di TEST
sudo vim /srv/getrad-stack/test/extracted/getrad/jsp/<modulo>/<file>.jsp

# 3. Forza ricompilazione Jasper su test
/srv/getrad-stack/test/recompile-test.sh

# 4. Verifica via curl sulla porta di TEST (dalla VM810)
curl -sI http://127.0.0.1:8090/getrad/jsp/<modulo>/<file>.jsp
# atteso: HTTP/1.1 200 OK (o 302 redirect al login)
# errore: HTTP/1.1 500 Internal Server Error

# 5. Log errori in caso di 500 (container di test)
docker compose -f /srv/getrad-stack/test/docker-compose.yml logs --tail=100 app \
  | grep -i -A20 "exception\|error\|Unable to compile"

# 6. Quando la rotta su 8090 è corretta, promuovi in produzione (backup automatico + recompile)
sudo /srv/getrad-stack/test/promote-to-prod.sh jsp/<modulo>/<file>.jsp

# 7. Verifica la stessa rotta su produzione (porta 8080)
curl -sI http://127.0.0.1:8080/getrad/jsp/<modulo>/<file>.jsp
```

Differenza rispetto al processo su VM809 (Ubuntu 20.04 bare-metal):
- Su VM809: `sudo systemctl stop tomcat9 && sudo rm -rf /var/lib/tomcat9/work/Catalina/localhost/getrad && sudo systemctl restart tomcat9`
- Su VM810: `docker exec getrad-app rm -rf /usr/local/tomcat/work/Catalina/localhost/getrad && docker compose restart app`
- I file JSP su VM809 erano in `/usr/skn/getrad/jsp/`; su VM810 sono in `/srv/getrad-stack/extracted/getrad/jsp/`

## Rotte e dati mockati

Nessun mock. Il test avviene contro l'ambiente di test su VM810 (porta 8090), seminato dai dati
di produzione, isolato dalla produzione. Rotte di test per i moduli con file pending (da
`current-work.md`):

```
http://192.168.20.90:8090/getrad/jsp/fornitori/insupdfornitore.jsp?id=13577496633860
http://192.168.20.90:8090/getrad/jsp/preventivi/_combinazione.jsp
http://192.168.20.90:8090/getrad/jsp/ordini/index.jsp
http://192.168.20.90:8090/getrad/jsp/fatture/index.jsp
http://192.168.20.90:8090/getrad/jsp/provvigioni/index.jsp
http://192.168.20.90:8090/getrad/jsp/riepiloghi/index.jsp
```

## Hook e controlli di qualità

Nessun lint, type-check né CI. I controlli di qualità sono:

- Compilazione JSP da Jasper (errori visibili nei log Tomcat e nella risposta HTTP 500)
- Verifica manuale nel browser della pagina renderizzata
- SHA256 dei backup per integrità archiviazione

Prompt Windsurf per patching automatico JSP (genera `FILE_fixed.jsp` da confrontare via `diff`
prima di applicare):

```
Esegui una scansione completa del file JSP aperto e correggi in modo deterministico tutti
gli errori che possono impedire la traduzione JSP → servlet → compilazione Java da parte
di Jasper su Tomcat 9. Identifica ed elimina ogni literal Java racchiuso in apici singoli
che contiene più di un carattere all'interno di scriptlet, expression tag o declaration tag.
Converti sempre i literal sbagliati da 'xxx' a "xxx". Mantieni chiari i livelli di quoting:
apici singoli esterni per HTML/XML, doppi apici interni per Java. Rileva e correggi anche
tutti i casi di conflitto tra apici HTML e apici Java all'interno degli attributi JSP,
assicurando che il contenuto Java sia sempre racchiuso tra doppi apici e che l'attributo
esterno utilizzi apici singoli. Verifica che non compaiano apici tipografici non ASCII e
sostituiscili con apici standard 0x22 o 0x27. Controlla la chiusura corretta di tutti i
tag <% %>, <%= %> e <%! %>. Correggi i literal char non validi e rimuovi ogni dichiarazione
Java sintatticamente non valida introdotta da errori precedenti. Assicurati che il codice
risultante sia compilabile dal compilatore Java usato da Jasper (Java 8+). Applica tutte le
modifiche in modo puntuale, senza introdurre refactoring logico o funzionale, intervenendo
esclusivamente sugli errori sintattici che impediscono la compilazione.
```

Regole di patching (sintesi dei casi da correggere):

```
Caso A - attributo HTML con espressione JSP e doppi apici:
  PRIMA: label="<%= multiLang.getLabel(id_lingua, "label.salva") %>"
  DOPO:  label='<%= multiLang.getLabel(id_lingua, "label.salva") %>'

Caso B - jsp:param con valore JSP:
  PRIMA: <jsp:param name="id" value="<%= one.get("id") %>" />
  DOPO:  <jsp:param name="id" value='<%= one.get("id") %>' />

Caso C - direttiva include:
  PRIMA: <%@ include file="contatti.jspf" %>
  DOPO:  <%@ include file='contatti.jspf' %>

Caso D - char literal Java multi-carattere:
  PRIMA: one.get('id')        (Java illegale: 'id' non è un char)
  DOPO:  one.get("id")

Caso E - confronto con char literal:
  PRIMA: if(one.get("fl_sms") == 'S')
  DOPO:  if("S".equals(one.get("fl_sms")))
```

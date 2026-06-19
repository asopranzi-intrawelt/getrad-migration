# Migrazione getrad: Tomcat 6 → Tomcat 9 — stato al 18/02/2026

Questo documento sintetizza lo stato di avanzamento della migrazione
dell'applicazione JSP "getrad" da Tomcat 6 (Ubuntu 10.04) a Tomcat 9
(Ubuntu 20.04), basandosi sulla conversazione Teams tra Alessio Sopranzi
e Tommaso Vezeni nel periodo 11-18 febbraio 2026.

L'applicazione è un gestionale (traduzione, clienti, fornitori, preventivi,
ordini, fatture, provvigioni, riepiloghi) accessibile via browser all'indirizzo:
`http://192.168.20.90:8080/getrad/`

---

## Infrastruttura

| | Vecchio | Nuovo |
|---|---|---|
| IP | 192.168.20.5 | 192.168.20.90 |
| OS | Ubuntu 10.04 | Ubuntu 20.04 (VM609 su Proxmox) |
| Tomcat | 6 | 9.0.31 |
| Java | 6 | 8 |
| Codice JSP | `/usr/skn/getrad/jsp/` | `/usr/skn/getrad/jsp/` |

---

## Problema root cause

Jasper (il compilatore JSP di Tomcat 9) è più rigoroso di quello di Tomcat 6.
Genera errore HTTP 500 "Unable to compile class for JSP" con il messaggio:

```
Attribute value [...] is quoted with [""] which must be escaped when used within the value
```

Questo accade ogni volta che un attributo HTML usa doppi apici come delimitatori
esterni e il valore interno contiene codice Java con doppi apici. Esempio:

```jsp
<!-- PRIMA (funzionava con Tomcat 6, rotto con Tomcat 9) -->
<skn:bottone label="<%= multiLang.getLabel(id_lingua, "label.cerca") %>" />

<!-- DOPO (corretto per Tomcat 9) -->
<skn:bottone label='<%= multiLang.getLabel(id_lingua, "label.cerca") %>' />
```

La regola è: **apici singoli esterni all'attributo HTML, doppi apici interni al Java**.

---

## Tutti i casi di errore da correggere

### Caso A — label, href, alt, value con JSP expression e doppi apici interni

```jsp
<!-- PRIMA -->
label="<%= multiLang.getLabel(id_lingua, "label.salva") %>"
href="<%=path_base+"/jsp/clienti/insupdcliente.jsp" %>"

<!-- DOPO -->
label='<%= multiLang.getLabel(id_lingua, "label.salva") %>'
href='<%=path_base+"/jsp/clienti/insupdcliente.jsp" %>'
```

### Caso B — jsp:param con valore JSP e doppi apici interni

```jsp
<!-- PRIMA -->
<jsp:param name="id_appaltatore" value="<%= one.get("id") %>" />

<!-- DOPO -->
<jsp:param name="id_appaltatore" value='<%= one.get("id") %>' />
```

### Caso C — direttiva include dentro markup HTML

```jsp
<!-- PRIMA -->
<%@ include file="contatti.jspf" %>

<!-- DOPO -->
<%@ include file='contatti.jspf' %>
```

### Caso D — char literal Java con più di un carattere (errore "Invalid character constant")

```java
// PRIMA (Java illegale: 'id' non è un char, è due caratteri)
one.get('id')
one.get('id_categoria')

// DOPO (corretto)
one.get("id")
one.get("id_categoria")
```

### Caso E — confronto stringa con char literal

```java
// PRIMA
if(one.get("fl_sms") == 'S')

// DOPO
if("S".equals(one.get("fl_sms")))
```

---

## Prompt Windsurf per il patching automatico

Questo prompt è stato usato su Windsurf (AI coding assistant) per generare
FILE_fixed.jsp a partire da FILE.jsp:

```
Esegui una scansione completa del file JSP aperto e correggi in modo
deterministico tutti gli errori che possono impedire la traduzione JSP →
servlet → compilazione Java da parte di Jasper su Tomcat 9. Identifica ed
elimina ogni literal Java racchiuso in apici singoli che contiene più di un
carattere all'interno di scriptlet, expression tag o declaration tag. Converti
sempre i literal sbagliati da 'xxx' a "xxx". Mantieni chiari i livelli di
quoting: apici singoli esterni per HTML/XML, doppi apici interni per Java.
Rileva e correggi anche tutti i casi di conflitto tra apici HTML e apici Java
all'interno degli attributi JSP, assicurando che il contenuto Java sia sempre
racchiuso tra doppi apici e che l'attributo esterno utilizzi apici singoli.
Verifica che non compaiano apici tipografici non ASCII e sostituiscili con
apici standard 0x22 o 0x27. Controlla la chiusura corretta di tutti i tag
<% %>, <%= %> e <%! %>. Correggi i literal char non validi e rimuovi ogni
dichiarazione Java sintatticamente non valida introdotta da errori precedenti.
Assicurati che il codice risultante sia compilabile dal compilatore Java usato
da Jasper (Java 8+). Applica tutte le modifiche in modo puntuale, senza
introdurre refactoring logico o funzionale, intervenendo esclusivamente sugli
errori sintattici che impediscono la compilazione.
```

---

## Processo di patch per ogni file

1. Aprire il file in Windsurf, lanciare il prompt → genera FILE_fixed.jsp
2. Confrontare le differenze da PowerShell su Windows:
   ```powershell
   fc.exe /L /N "C:\percorso\FILE.jsp" "C:\percorso\FILE_fixed.jsp"
   ```
3. Su Ubuntu 20.04, rimuovere il vecchio file e copiare il nuovo:
   ```bash
   sudo rm -rf /usr/skn/getrad/jsp/<cartella>/FILE.jsp
   sudo cp -a /home/intrawelt/Scaricati/FILE_fixed.jsp /usr/skn/getrad/jsp/<cartella>/FILE.jsp
   ```
   (il file viene mandato da Windows via email a alesop.intrawelt@gmail.com e scaricato in `/home/intrawelt/Scaricati/`)
4. Forzare la ricompilazione Jasper riavviando Tomcat:
   ```bash
   sudo systemctl stop tomcat9
   sudo rm -rf /var/lib/tomcat9/work/Catalina/localhost/getrad
   sudo systemctl restart tomcat9
   ```
5. Verificare accedendo alla rotta corrispondente via browser
6. Se funziona, creare backup stabile:
   ```bash
   sudo mkdir "/home/intrawelt/Scrivania/backup_usr_skn_getrad_jsp_DDMMYYYY_HHMM"
   sudo cp -a /usr/skn/getrad/jsp/. "/home/intrawelt/Scrivania/backup_usr_skn_getrad_jsp_DDMMYYYY_HHMM"
   ```

---

## Stampa riga singola per debug

Per ispezionare un riga specifica prima di modificarla:

```bash
# riga singola
sed -n '171p' /usr/skn/getrad/jsp/clienti/index.jsp

# intorno della riga (10 righe prima e dopo)
sed -n '161,181p' /usr/skn/getrad/jsp/clienti/index.jsp
```

Per modificare con nano con numeri di riga visibili:
```bash
sudo nano -l /usr/skn/getrad/jsp/clienti/index.jsp
```

---

## Stato file al 18/02/2026

### FIXED (funzionanti, backup incluso nel `backup_usr_skn_getrad_jsp_18022026_1200`)

| File | Righe modificate | Note |
|---|---|---|
| `clienti/index.jsp` | 171, 253, 318 | Pulsanti nuovo/cerca |
| `fornitori/index.jsp` | 65, 119 | Pulsanti nuovo fornitore |
| `clienti/insupdcliente.jsp` | 1236, 1237, 1238, 1326, 1357, 1408, 1409, 1414, 1618, 1645, 1680, 1741, 1742, 1757, 1810, 1815, 1817, 1819, 1828 | File principale scheda cliente |
| `clienti/contatti.jspf` | 369, 371, 374, 375 | Include di insupdcliente |
| `clienti/commerciali.jspf` | 33 | Include di insupdcliente |
| `clienti/pm_rif.jspf` | 22 | Include di insupdcliente |
| `clienti/cli_gruppo.jspf` | 26, 59 | Include di insupdcliente |
| `_include/templateBottomBlank.jspf` | 1 | Template globale |
| `_include/templateBottom.jspf` | 4 | Template globale |
| `preventivi/index.jsp` | 314, 319, 350, 355, 387, 388, 539, 592, 600 | Lista preventivi |
| `preventivi/insupdpreventivo.jsp` | 1314, 1315, 1337, 1367, 1381, 1386, 1408, 1409, 1414, 1618, 1645, 1680, 1741, 1742, 1757, 1810, 1815, 1817, 1819, 1828 | Scheda preventivo |
| `preventivi/_combinazione.jsp` | 153 | Include combinazione |

### PENDING (da fare)

Questi file producono ancora HTTP 500 al 18/02/2026 o non sono mai stati toccati:

| File | Errore noto | Route di test |
|---|---|---|
| `fornitori/insupdfornitore.jsp` | righe 369, 370, 672, 713, 714 — `one.get('id')` alla riga 672 | `http://192.168.20.90:8080/getrad/jsp/fornitori/insupdfornitore.jsp?id=13577496633860` |
| `preventivi/_combinazione.jsp` | HTTP 500 "Unable to compile class for JSP" — log in `HTTP500_combinazione_log.txt` | `http://192.168.20.90:8080/getrad/jsp/preventivi/_combinazione.jsp` |
| `ordini/index.jsp` | line 211 col 56: `multiLang.getLabel(id_lingua, "label.cerca")` | `http://192.168.20.90:8080/getrad/jsp/ordini/index.jsp` |
| `fatture/index.jsp` | line 155 → via `fatture/index_tr.jspf` line 78 → `fatture/fatture_tr.jspf` line 479 | `http://192.168.20.90:8080/getrad/jsp/fatture/index.jsp` |
| `traduttori/banca.jsp` | mai toccato | |
| `traduttori/emailtrad.jsp` | mai toccato | |
| `provvigioni/index.jsp` | line 108 → via `provvigioni/provvigioni_env.jspf` line 20 | `http://192.168.20.90:8080/getrad/jsp/provvigioni/index.jsp` |
| `riepiloghi/index.jsp` | line 54 → via `riepiloghi/index_tr.jspf` line 86 | `http://192.168.20.90:8080/getrad/jsp/riepiloghi/index.jsp` |

Altri file identificati dalla scansione grep ricorsiva come candidati al patching
(non ancora toccati, possono avere errori latenti):

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
preventivi/insupdpreventivo.jsp  (già fixato sopra)
provvigioni/index.jsp             (pending)
search/index.jsp
societa/insupdsocieta.jsp
test/clienti_settori.jsp
traduttori/cercaTrad.jsp
traduttori/comb_ling_int.jsp
traduttori/comb_ling.jsp
traduttori/emailtrad.jsp          (pending)
traduttori/home_trad.jsp
traduttori/professionali_int.jsp
traduttori/professionali.jsp
vipa/amministra.jsp
vipa/index.jsp
```

---

## Attenzione: cosa NON modificare

- Non toccare le virgolette dentro blocchi `onclick="javascript:..."` puri HTML
  (nessuna espressione JSP dentro): quelle non causano errori
- Non cambiare la logica applicativa, solo la quoting degli attributi
- Non modificare file nelle sottocartelle `20140603/`, `20120523/`, `20210129/`
  (versioni storiche archiviate) a meno che non causino errori diretti

---

## Backup disponibili su Ubuntu 20.04

| Nome cartella | Data | Contenuto |
|---|---|---|
| `backup_usr_skn_11022026` | 2026-02-11 | Stato pre-migrazione (64GB, tutto /usr/skn/) |
| `backup_usr_skn_getrad_jsp_17022026` | 2026-02-17 ore 11:04 | Solo /usr/skn/getrad/jsp/ (11MB) |
| `backup_usr_skn_getrad_jsp_18022026_1200` | 2026-02-18 ore 11:58 | Dopo fix insupdcliente + preventivi |
| `backup_usr_skn_getrad_jsp_18022026_1440` | 2026-02-18 ore 14:38 | Ultimo backup stabile |

Tutti in `/home/intrawelt/Scrivania/`.

Ripristino in caso di emergency:
```bash
sudo rm -rf /usr/skn/getrad/jsp/
sudo mkdir /usr/skn/getrad/jsp/
sudo cp -a "/home/intrawelt/Scrivania/backup_usr_skn_getrad_jsp_18022026_1440/." /usr/skn/getrad/jsp/
sudo systemctl stop tomcat9
sudo rm -rf /var/lib/tomcat9/work/Catalina/localhost/getrad
sudo systemctl restart tomcat9
```

---

## Nota su Solr

La migrazione dell'indice Solr (su `http://192.168.20.90:8080/solr/`) è già completata
e verificata: confronto degli ID dei documenti tra vecchio e nuovo server con
`fc.exe` ha dato "FC: nessuna differenza riscontrata". Solr non è coinvolto
nei problemi JSP attuali.

---

## Prossimo step

1. Fixare `fornitori/insupdfornitore.jsp` (priorità alta — usato per inserimento fornitori)
2. Investigare `preventivi/_combinazione.jsp` (l'HTTP 500 ha un log allegato `HTTP500_combinazione_log.txt`)
3. Procedere per `ordini/index.jsp`, `fatture/index.jsp`, `provvigioni/index.jsp`, `riepiloghi/index.jsp`
4. Poi passare ai file "mai toccati" della lista: traduttori, lead, societa, ecc.
5. A ogni passo: backup stabile dopo verifica funzionamento

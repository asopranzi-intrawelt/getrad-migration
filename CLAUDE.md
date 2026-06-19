# getrad-migration

> Istruzioni di progetto, versionate. Documentazione operativa e di migrazione dello stack getrad. Le preferenze personali vivono in `CLAUDE.local.md` (ignorato).

## Cos'e questo progetto

Dossier di migrazione e documentazione operativa del gestionale getrad (eGeTrad, Intrawelt), applicazione Java su Tomcat con database MariaDB. Lo stack attuale gira come Docker compose sulla VM810 (Ubuntu 24.04 LTS). Il deliverable di valore e la documentazione: `README.md` descrive architettura, avvio, backup, restore, disaster recovery e troubleshooting dello stack containerizzato; `GETRAD_TOMCAT_MIGRATION.md` sintetizza i successivi sviluppi applicativi. Gli artefatti legacy, ovvero archivi, binari e le vecchie estrazioni da Ubuntu 10.04, restano locali e non sono versionati.

## Dati sensibili (mai versionati)

Il file `getrad.properties` contiene credenziali di produzione in chiaro, password del mailer Office365, password del database e token applicativi, ed e gitignored. Il riferimento versionato e `getrad.properties.example`, con i segreti sostituiti da `__SET_ME__`. Sono inoltre ignorati tutti gli archivi, i binari, i documenti Office grezzi e le cartelle `old/` e `appoggio_da_ubuntu_10_04/`. Il `.gitignore` adotta una logica di blocco per estensione con eccezione esplicita per gli `.example`. La password Office365 presente nel file e viva e riutilizzata: va rotata in produzione, indipendentemente dal versionamento.

## Sviluppo e identita

Lo sviluppo effettivo avviene su una macchina Linux, dove il repository viene clonato e fatto evolvere; la predisposizione e il primo push avvengono da Windows. L'identita git locale e `asopranzi@intrawelt` con alias SSH `github-corp`, remoto `github.com/asopranzi-intrawelt/getrad-migration`. Su Windows e forzato `core.sshCommand` all'OpenSSH di sistema; su Linux non serve perche `ssh` di sistema legge gia il proprio config. Commit e push restano manuali.

## Standard

Allineato allo standard portabile `.claude/PROJECT-SYSTEM.md`: regole, engine skills (`sync-context`, `git-sync`, `repo-status`, `onboard`), catalogo `.claude/templates/PACKAGES.md` e schede `context/` come scaffold da popolare, ancorabili con `sync-context` dopo il primo commit. Progetto a sola documentazione: nessun pacchetto MCP adottato perche non esiste un albero di codice sorgente da mappare.

# Snapshot di sincronizzazione

> Da leggere per primo a inizio sessione. Fotografa lo stato del progetto al commit di
> riferimento e mappa ogni scheda al suo stato di verifica. È la fonte di verità su cosa è
> fatto, non le spunte del diario.

## Stato

```
Branch attivo:          main
Commit di riferimento:  11b13e8
Data snapshot:          2026-06-19
```

## Stato di verifica delle schede

| Scheda | last-verified | Stato |
|---|---|---|
| STACK.md | 4f686bf | aggiornata |
| design-and-security.md | 4f686bf | aggiornata |
| deployment.md | 4f686bf | aggiornata |
| dev-testing.md | 4f686bf | aggiornata |
| current-work.md | 4f686bf | aggiornata |
| roadmap.md | 4f686bf | aggiornata |

## Punto di ripresa

SSH su VM810 (`ssh intrawelt@192.168.20.90`), eseguire backup pre-patching con il comando
in `deployment.md`, poi iniziare con `fornitori/insupdfornitore.jsp` (errori noti alle
righe 369, 370, 672, 713, 714 — vedi `current-work.md` per dettagli e rotta di test).

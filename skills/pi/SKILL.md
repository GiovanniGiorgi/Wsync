---
name: wsync
description: Gestisce il ciclo pull→lavoro→push con wsync (sync manuale stile git su cartelle remote via rsync/ssh). All'inizio di una sessione, cerca un nodo wsync nella cartella di lavoro o fino a due livelli sopra; prima di modificare file gestiti, assicura che la workingdir sia allineata al server. Usare quando si lavora dentro o vicino a una cartella wsync, o quando l'utente menziona wsync, pull/push del vault o snapshot del workspace.
---

# wsync — ciclo pull → lavoro → push

`wsync` sincronizza una cartella remota con una `workingdir/` locale, guardiani + nastro di snapshot hardlink in `.snapshots/`. Modello: **un solo umano alla volta**, flusso deliberato. Mai modificare gli snapshot.

## 0. Trovare il nodo (all'inizio di ogni sessione)

Un nodo wsync è una cartella che contiene lo script `wsync` eseguibile e `workingdir/`. Cercarlo **solo** qui, nell'ordine, senza andare oltre due livelli sopra la cwd:

```bash
find_wsync() {
  local d="$PWD" i
  for i in 0 1 2; do
    [ -x "$d/wsync" ] && [ -d "$d/workingdir" ] && { echo "$d"; return 0; }
    d="$(dirname "$d")"
  done
  return 1
}
NODE="$(find_wsync)"
```

- **Nessun nodo** → comportarsi normalmente, non menzionare wsync.
- **Nodo = cartella di lavoro** → procedere subito col punto 1 (pull automatico prima di toccare file).
- **Nodo più in alto (1–2 livelli)** → **chiedere all'utente** prima di qualsiasi pull/lavoro: *"Trovo un nodo wsync in `<path>` (sopra la cartella di corrente): la workingdir è `<node>/workingdir`. Vuoi che faccia il pull e lavori lì?"*. Non tirare a indovinare: la workingdir del nodo NON è la cwd.

## 1. Pull prima di lavorare (solo con nodo nella cwd)

```bash
"$NODE/wsync" status    # cosa c'è già di locale non pushato (istantaneo)
```
- status pulito → `"$NODE/wsync" pull` (riferimento di ciclo + snapshot del server)
- status sporco → **non sovrascrivere di nascosto**: mostra i file all'utente e chiedi. Le uscite sane sono: `push` prima (se il lavoro è suo e completo), oppure pull con salvataggio d'emergenza (lo script lo propone da solo in modalità interattiva; da bash non interattiva si autoblocca: riferire il messaggio all'utente)

## 2. Lavorare

Modificare file **solo dentro** `"$NODE/workingdir/"`. Durante il lavoro, `"$NODE/wsync" status` è gratis: usarlo spesso.

## 3. Riconciliare e pushare (a fine lavoro, o prima di sessioni lunghe)

```bash
"$NODE/wsync" diff      # deriva del server (auto-snapshot se c'è) + tuoi cambi
```
- poi `"$NODE/wsync" push` — i file sostituiti sul server vengono salvati automaticamente in `.snapshots/server-*-E/`
- **Se il server è cambiato (deriva), valutare SEMPRE prima del push — mai pushare alla cieca:**
  1. calcolare le due liste: file cambiati da lui (lista "server") e cambiati da noi (lista "tuoi")
  2. **Nessuna sovrapposizione** (cartelle o file disgiunti): proporre esplicitamente all'utente l'**unione**: copiare i file della deriva dalla foto appena scattata (`cp -a .snapshots/server-<R₂>/<file> workingdir/<file>`) **prima** di pushare, così le scritture remote non vengono sovrascritte. Spiegare in una riga cosa si sta unendo e chiedere ok
  3. **Sovrapposizione** (stesso file cambiato su due lati): conflitto reale → `wsync show <file>` (foto vs workingdir), mai risolvere da soli: mostrare le versioni, proporre il merge, scrivere il risultato solo dopo approvazione esplicita
  4. in entrambi i casi, ricordare che il guardiano del push c'è e che le eventuali vittime finiscono in `.snapshots/server-*-E/`: nessuna scrittura remota è persa per sempre, ma l'unione *prima* del push è la strada pulita

## Comandi di supporto

| Comando | Uso |
|---|---|
| `wsync hist` | nastro: 🏁 push, 📷 foto server, 🚨 vittime, 🛟 salvataggi |
| `wsync show <file>` | diff unificato: ultima foto ↔ workingdir (o tra due snapshot) |
| `wsync diff-snap A B` | differenze tra due snapshot (bastano frammenti di data) |
| `wsync keep [N]` | potatura snapshot (default 60) |
| `wsync repair` | recupera/pulisce staging residui di push interrotti sul server |

## Regole ferree

1. Mai-editare `.snapshots/`, `.wsync-cycle`, né lo script durante il flusso
2. Mai `WSYNC_YES=1` su `push` senza che l'utente abbia visto il `diff` — il guardiano serve proprio a quello
3. Il pull è l'unica operazione "di routine" autorizzata in automatico, e solo a workingdir pulita e solo su nodo nella cwd
4. Prima di scrivere qualsiasi file della workingdir, verificare che il pull del ciclo sia stato fatto (nodo in cwd) o che l'utente abbia esplicitamente detto di lavorare senza
5. Il modello è single-writer: se l'utente lavora sulla stessa cartella da due macchine, avvertire di finire il ciclo (push) sull'altra prima di continuare
6. `push` non è mai automatico a fine turno: si pusha solo su parola esplicita dell'utente ("push", "salva sul server", conferma del diff). **Può però essere proposto**: a fine lavoro, o dopo una modifica significativa, l'AI può chiedere *«pusho sul server?»* — mostrando prima il `diff`. Proposta sì, esecuzione senza risposta no

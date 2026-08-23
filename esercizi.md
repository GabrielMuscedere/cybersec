# Esercizi Consigliati per l'Esame Pratico (CyberChallenge & Olicyber)

Questo file contiene la checklist completa di tutti gli esercizi e le sfide pratiche consigliate all'interno delle slide (dal Modulo 02 al Modulo 05) per allenarsi in vista dell'esame di cybersecurity. Puoi utilizzarlo come una vera e propria tabella di marcia, spuntando i laboratori man mano che li completi.

---

## 🛠️ Modulo 02: Command and Code Injections

### 1. Test di Esfiltrazione Locale (Time-Based / Out-of-Bound)
*   **Piattaforma:** Ambiente locale di test (BlackBox)
*   **Obiettivo:** Verificare se un parametro è vulnerabile introducendo ritardi (`sleep 5`) o esfiltrando dati sensibili (come un file di testo) verso un server di ascolto esterno (ad esempio Webhook.site o il tuo listener locale in Python).
*   **Utilità:** Ottimo per testare la stabilità dei tuoi script di ricezione prima di interfacciarti con i server di gara.

### 2. PlottyBoy
*   **Piattaforma:** CyberChallenge
*   **URL:** http://plottyboy.challs.cyberchallenge.it/
*   **Obiettivo:** Sfruttare una vulnerabilità di iniezione all'interno dell'applicazione per individuare ed esfiltrare la flag presente nel percorso `/flag.txt`.

### 3. PHP is Love
*   **Piattaforma:** CyberChallenge
*   **URL:** http://phpislove.challs.cyberchallenge.it/
*   **Obiettivo:** Analizzare ed eseguire codice PHP arbitrario abusando delle funzioni di valutazione dinamica dell'interprete (PHP Code Injection).

---

## 📂 Modulo 03: File Disclosure & SSRF

### 1. Basic LFI
*   **Piattaforma:** CyberChallenge
*   **URL:** http://basiclfi.challs.cyberchallenge.it/
*   **Obiettivo:** Sfruttare vulnerabilità di Local File Inclusion e Path Traversal per evadere dalla Document Root del server web. L'esercizio richiede di leggere file di sistema (es. `/etc/passwd`) o i sorgenti PHP dell'applicazione (es. `config.php`) sfruttando i wrapper PHP (`php://filter`) per prevenirne l'esecuzione.

### 2. SSRF 1
*   **Piattaforma:** CyberChallenge
*   **URL:** http://ssrf1.challs.cyberchallenge.it/
*   **Obiettivo:** Forzare l'applicazione vulnerabile a inviare richieste di rete arbitrarie verso l'infrastruttura interna, permettendo di scansionare le porte di servizi locali (localhost) o di esplorare host privati normalmente nascosti dietro il firewall.

---

## 🗄️ Modulo 04: SQL Injection

### 1. SQLi Logic Bypass
*   **Piattaforma:** Olicyber
*   **URL:** http://web-17.challs.olicyber.it/logic
*   **Obiettivo:** Eludere i controlli di autenticazione nella schermata di login ed effettuare l'accesso come amministratore senza conoscere la password reale, iniettando una tautologia (es. `' OR 1=1 -- `).

### 2. Union-Based SQLi
*   **Piattaforma:** Olicyber
*   **URL:** http://web-17.challs.olicyber.it/union
*   **Obiettivo:** Individuare il numero esatto di colonne restituite dalla query legittima e utilizzare l'operatore `UNION` per fondere i risultati con una seconda query. Lo scopo è navigare nel database `INFORMATION_SCHEMA` per estrarre tabelle, colonne e infine i dati sensibili.

### 3. Boolean-Based Blind SQLi
*   **Piattaforma:** Olicyber
*   **URL:** http://web-17.challs.olicyber.it/blind
*   **Obiettivo:** Esfiltrare informazioni carattere per carattere dal database inviando condizioni logiche (Vero/Falso) e osservando come l'applicazione altera la risposta a schermo (oracolo booleano).

### 4. Time-Based Blind SQLi
*   **Piattaforma:** Olicyber
*   **URL:** http://web-17.challs.olicyber.it/time
*   **Obiettivo:** Estrarre dati in modo completamente cieco quando l'applicazione non mostra differenze visive, forzando ritardi temporali controllati (tramite la funzione `sleep()`) per dedurre la correttezza di ogni singolo carattere esfiltrato.

---

## 🌐 Modulo 05: XSS & CSRF

### 1. XSS 1
*   **Piattaforma:** CyberChallenge
*   **URL:** http://xss1.challs.cyberchallenge.it/
*   **Obiettivo:** Iniettare ed eseguire JavaScript arbitrario nel browser di un bot amministratore che simula la vittima. Lo scopo è catturare i suoi cookie di sessione (`document.cookie`) e spedirli indietro verso il proprio server di ascolto (HTTP listener).

### 2. Shops
*   **Piattaforma:** Olicyber
*   **URL:** http://shops.challs.olicyber.it/
*   **Obiettivo:** Sfruttare la mancanza di token di verifica o di configurazioni SameSite adeguate per indurre un utente legittimo e autenticato a compiere azioni dannose non intenzionali (es. un acquisto o una transazione) tramite richieste cross-site silenziose (CSRF).

### 3. NFlagT
*   **Piattaforma:** CyberChallenge
*   **URL:** http://nflagt.challs.cyberchallenge.it/
*   **Obiettivo:** Risolvere una sfida di CSRF di livello avanzato che richiede un'analisi approfondita delle restrizioni del flag SameSite sui cookie e la coordinazione di un exploit client-side strutturato.

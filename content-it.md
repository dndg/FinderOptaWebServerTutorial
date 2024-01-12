---
title: 'Web Server su Finder Opta'
description: "Imparare ad implementare un Web Server su Finder Opta, gestendo
              richieste HTTP di tipo POST e GET su multipli endpoint."
author: 'Fabrizio Trovato'
libraries:
  - name: 'ArduinoHttpClient'
    url: https://www.arduino.cc/reference/en/libraries/arduinohttpclient/
  - name: 'ArduinoJson'
    url: https://arduinojson.org
difficulty: intermediate
tags:
  - HTTP
  - Ethernet
  - JSON
software:
  - ide-v1
  - ide-v2
  - arduino-cli
  - web-editor
hardware:
  - hardware/07.opta/opta-family/opta
---

## Panoramica

Nei precedenti esempi abbiamo mostrato [come configurare un indirizzo IP sul
Finder
Opta](https://github.com/dndg/FinderOptaEthernetTutorial/blob/main/content-it.md)
e [come utilizzare le librerie Ethernet e HTTP per inviare richieste `POST` ad
un server HTTP](https://github.com/dndg/FinderOptaHttpClientTutorial). In
questo tutorial, sarà invece il Finder Opta a comportarsi da server: il
dispositivo riceverà richieste `POST` o `GET` da un client connesso via
Ethernet, e ne leggerà il contenuto per fornire una risposta o compiere
un'azione.

## Obiettivi

* Imparare a gestire richieste HTTP di tipo `POST` e `GET` con Finder Opta,
  implementando un semplice Web Server con multipli endpoint.
* Imparare a leggere richieste e generare riposte in formato JSON.

## Requisiti hardware e software

### Requisiti hardware

* PLC Finder Opta (x1).
* Cavo USB-C® (x1).
* Cavo ETH RJ45 (x1).

### Requisiti Software

* [Arduino IDE 1.8.10+](https://www.arduino.cc/en/software), [Arduino IDE
2.0+](https://www.arduino.cc/en/software) o [Arduino Web
Editor](https://create.arduino.cc/editor).
* Se si utilizza Arduino IDE offline, è necessario installare le librerie
  `ArduinoHttpClient` e `ArduinoJson` utilizzando il Library Manager
  dell'Arduino IDE.
* [Codice di esempio](assets/OptaWebServerExample.zip).

## Finder Opta e i protocolli Ethernet e HTTP

Grazie alla libreria `Ethernet`, Finder Opta può avviare un server in ascolto
su una specifica porta. Questo significa che il dispositivo rimarrà in attesa
di connessioni da parte di un client, per riceverne le richieste. Una volta
ricevute, Finder Opta può utilizzare la libreria `ArduinoHttpClient` per
interpretarne il contenuto.

## Finder Opta e il formato JSON

Grazie alla libreria `ArduinoJson`, Finder Opta può facilmente leggere e
generare documenti in formato JSON. Nel nostro esempio, questo risulta
particolarmente utile per leggere le richieste di tipo `POST` e generare le
risposte HTTP contenti lo stato dei LED di Finder Opta.

## Istruzioni

### Configurazione dell'Arduino IDE

Per seguire questo tutorial, sarà necessaria [l'ultima versione dell'Arduino
IDE](https://www.arduino.cc/en/software). Se è la prima volta che configuri il
Finder Opta, dai un'occhiata al tutorial [Getting Started with
Opta](/tutorials/opta/getting-started).

Assicurati di installare l'ultima versione delle librerie
[`ArduinoHttpClient`](https://www.arduino.cc/reference/en/libraries/arduinohttpclient/)
e [`ArduinoJson`](https://arduinojson.org) poiché verranno utilizzate per
leggere e creare il contenuto di richieste e risposte HTTP.

Per ulteriori dettagli su come installare manualmente le librerie, consulta
[questo
articolo](https://support.arduino.cc/hc/en-us/articles/5145457742236-Add-libraries-to-Arduino-IDE).

### Connettività

L'unico requisito di questo tutorial è che Finder Opta sia connesso tramite
Ethernet ad un dispositivo in grado di instradare i pacchetti dal Finder Opta
al client HTTP e viceversa.

### Panoramica del codice

Lo scopo del seguente esempio è avviare su Finder Opta un Web Server in grado
di ricevere richieste HTTP che permettano di pilotarne i LED. Il Web Server
dovrà identificare metodo e path della richiesta, in particolare:

* In caso di richiesta `GET` al path `/` il Web Server restituirà una pagina
  HTML contenente uno script Javascript, che permetterà di pilotare i LED di
  Finder Opta e mostrarne lo stato.
* In caso di richieste di qualsiasi altro tipo al path `/` il Web Server
  restituirà un errore HTTP di tipo `400 Bad Request`.
* In caso di richiesta `GET` al path `/led` il Web Server restituirà un JSON
  contenente lo stato dei LED di Finder Opta.
* In caso di richiesta `POST` al path `/led` il Web Server riceverà un comando
  in formato JSON per pilotare i LED ed eseguirà l'azione in esso contenuta. In
  seguito il Web Server fornirà una risposta in formato JSON contenente lo
  stato dei LED di Finder Opta.
* In caso di richieste di qualsiasi altro tipo al path `/led` il Web Server
  restituirà un errore HTTP di tipo `400 Bad Request`.
* In caso di richieste di qualsiasi tipo a path diversi da `/` o `led` il Web
  Server restituirà un errore HTTP di tipo `404 Not Found`.

#### Setup dello sketch

Nella parte iniziale dello sketch dichiariamo alcune costanti:

```cpp
#define HOME_PATH "/"
#define LED_PATH "/led"
#define HTTP_GET "GET"
#define HTTP_POST "POST"
#define MAX_METHOD_LEN 16
#define MAX_PATH_LEN 2048
```

In particolare definiamo metodi e path supportati dal Web Server, e la
lunghezza massima che possono avere all'interno delle richieste HTTP.
Successivamente dichiaramo le variabili necessarie per avviare un Web Server
raggiungibile ad un certo indirizzo IP statico e ad una certa porta:

```cpp
OptaBoardInfo *info;
OptaBoardInfo *boardInfo();

// IP address of the Opta server.
IPAddress ip(192, 168, 10, 15);
int port = 80;
// Ethernet server on port 80.
EthernetServer server(port);
```

In seguito dichiariamo due oggetti di tipo `StaticJsonDocument` che useremo per
richieste e risposte:

```cpp
// Reserve 128 bytes for the JSON.
StaticJsonDocument<128> res;
StaticJsonDocument<128> req;
```

Infine dichiariamo l'array di stringhe contenente gli stati dei LED dal numero
0 al numero 3, mappati con delle stringhe in modo da gestire facilmente i LED
tramite richieste HTTP evitando che l'utente debba inserire il valore intero
corrispondente allo stato del LED desiderato. Questa mappatura ci permetterà
inoltre di generare il contenuto delle risposte fornite dal nostro Web Server
in un formato comprensibile all'utente.

```cpp
// LEDs states.
String ledStates[] = {"LOW", "LOW", "LOW", "LOW"};
```

Nella funzione di `setup()` ci limitiamo ad assegnare l'indirizzo IP statico al
Finder Opta, avviando poi il server:

```cpp
void setup()
{
    Serial.begin(9600);

    info = boardInfo();
    // Check if secure informations are available since MAC Address is among them.
    if (info->magic = 0xB5)
    {
        // Assign static IP address.
        Ethernet.begin(info->mac_address, ip);
    }
    else
    {
        while (1)
        {
        }
    }

    // Start the server.
    server.begin();
}
```

#### Loop principale

La funzione `loop()` di questo sketch rimane in ascolto in attesa di client,
fino a quando non se ne connette uno. In caso il server riceva una connessione
da parte di un client inizializziamo un oggetto per interpretare la richiesta.

```cpp
void loop()
{
    // Check if any client is available.
    EthernetClient client = server.available();
    if (client)
    {
        HttpClient http = HttpClient(client, ip, port);
        IPAddress clientIP = client.remoteIP();
        Serial.println("Client with address " + clientIP.toString() + " available.");

        while (client.connected())
        {
            if (client.available())
            {

```

In seguito, il server interpreta metodo e path della richiesta usando la
funzione `getHttpMethodAndPath()`:

```cpp
                // Read HTTP method and path from the HTTP call.
                char method[MAX_METHOD_LEN], path[MAX_PATH_LEN];
                getHttpMethodAndPath(&http, method, path);
```

Questa funzione legge i byte della richiesta fino a quando non raggiunge un
separator, e salva prima il metodo HTTP e poi il path della richiesta, entrambi
all'interno di un array di `char` null-terminated:

```cpp
void getHttpMethodAndPath(HttpClient *http, char *method, char *path)
{
    size_t l = http->readBytesUntil(' ', method, MAX_METHOD_LEN - 1);
    method[l] = '\0';

    l = http->readBytesUntil(' ', path, MAX_PATH_LEN - 1);
    path[l] = '\0';
}
```

A questo punto il server confronterà metodo e path alle costanti definite in
precedenza per determinare quali azioni intraprende, sulla base di quanto
spiegato nel corso di questo tutorial.

Il codice seguente si occupa dell'endpoint `/`:

```cpp
                // If path matches "/".
                if (strncmp(path, HOME_PATH, MAX_PATH_LEN) == 0)
                {
                    Serial.println("Client with address " + clientIP.toString() + " connected to '/'...");
                    // This endpoint only accepts GET requests.
                    if (strncmp(method, HTTP_GET, MAX_METHOD_LEN) == 0)
                    {
                        sendHomepage(&client);
                    }
                    else
                    {
                        badRequest(&client);
                    }
                }
```

In caso di richieste `GET` viene invocata la funzione `sendHomepage`, il cui
codice è mostrato di seguito:

```cpp
void sendHomepage(EthernetClient *client)
{
    client->println("HTTP/1.1 200 OK");
    client->println("Connection: close");
    client->println("Content-Type: text/html");

    String html = R"(
    <!DOCTYPE html>
    <html>
    <body>
        <h2>Control the LEDs on the Finder Opta</h2>
        <form id="form">
            <p>LED 0:</p>
            <input type="radio" id="LED_D0" name="LED_D0" value="HIGH">
            <label for="HIGH">On</label><br>
            <input type="radio" id="LED_D0" name="LED_D0" value="LOW">
            <label for="LOW">Off</label><br>
            <br>
            <p>LED 1:</p>
            <input type="radio" id="LED_D1" name="LED_D1" value="HIGH">
            <label for="HIGH">On</label><br>
            <input type="radio" id="LED_D1" name="LED_D1" value="LOW">
            <label for="LOW">Off</label><br>
            <br>
            <p>LED 2:</p>
            <input type="radio" id="LED_D2" name="LED_D2" value="HIGH">
            <label for="HIGH">On</label><br>
            <input type="radio" id="LED_D2" name="LED_D2" value="LOW">
            <label for="LOW">Off</label><br>
            <br>
            <p>LED 3:</p>
            <input type="radio" id="LED_D3" name="LED_D3" value="HIGH">
            <label for="HIGH">On</label><br>
            <input type="radio" id="LED_D3" name="LED_D3" value="LOW">
            <label for="LOW">Off</label><br>
            <br>
            <button type="submit" id="submit-btn">Apply</button>
        </form>
        <script type="application/javascript">
            const form = document.getElementById('form');
            const dataPromise = fetch('http://192.168.10.15:80/led')
                .then(res => res.json())
                .then(data => { return data; });
            window.onload = async () => {
                let data = await dataPromise;
                form["LED_D0"].value = data["LED_D0"];
                form["LED_D1"].value = data["LED_D1"];
                form["LED_D2"].value = data["LED_D2"];
                form["LED_D3"].value = data["LED_D3"];
            };
            form.addEventListener('submit', async event => {
                event.preventDefault();
                const formData = new FormData(form);
                const body = JSON.stringify(Object.fromEntries(formData));
                try {
                    const res = await fetch(
                        'http://192.168.10.15:80/led',
                        {
                            method: 'POST',
                            body: body,
                        },
                    );
                    const data = await res.json();
                    let isOk = false;
                    if (res.ok) {
                        isOk = form["LED_D0"].value == data["LED_D0"];
                        isOk = form["LED_D1"].value == data["LED_D1"];
                        isOk = form["LED_D2"].value == data["LED_D2"];
                        isOk = form["LED_D3"].value == data["LED_D3"];
                        if (isOk) {
                            alert('LEDs were set.');
                        } else {
                            alert('LEDs were not set.');
                        }
                    } else {
                        alert('Server error.');
                    }
                } catch (err) {
                    console.log(err.message);
                }
            });
        </script>
    </body>
    </html>)";

    client->println("Content-Length: " + String(html.length() + 1));
    client->println();
    client->println(html);

    Serial.println("OK [200]");
}
```

Il server invierà quindi una pagina HTML contenente uno scipt Javascript che
permetterà di controllare i LED del Finder Opta. La pagina è mostrata in
figura:

<img src="assets/client.png" width="500">

Il codice seguente si occupa dell'endpoint `/led`:

```cpp
                // If path matches "/led".
                else if (strncmp(path, LED_PATH, MAX_PATH_LEN) == 0)
                {
                    Serial.println("Client with address " + clientIP.toString() + " connected to '/led'...");
                    // This endpoint accepts both GET and POST requests.
                    if (strncmp(method, HTTP_GET, MAX_METHOD_LEN) == 0)
                    {
                        // Respond to GET with LEDs states.
                        sendLEDsStates(&client);
                    }
                    else if (strncmp(method, HTTP_POST, MAX_METHOD_LEN) == 0)
                    {
                        // Skip headers and read POST request body.
                        http.skipResponseHeaders();
                        String body = http.readString();

                        // In case of POST requests with a body change LEDs states.
                        if (body != "")
                        {
                            parseRequest(body);
                        }

                        // In case of POST requests also respond with LEDs states.
                        sendLEDsStates(&client);
                    }
                    else
                    {
                        badRequest(&client);
                    }
                }
```

In caso di richiesta `GET` il server chiama la funzione `sendLEDsStates`, che
genera una risposta in formato JSON contenente lo stato dei LED di Finder Opta:

```cpp
void sendLEDsStates(EthernetClient *client)
{
    // Sent HTTP headers.
    client->println("HTTP/1.1 200 OK");
    client->println("Connection: close");
    client->println("Content-Type: application/json");

    // Read LEDs states.
    for (int i = 0; i <= 3; i++)
    {
        String field = "LED_D" + String(i);
        res[field] = ledStates[i];
    }

    // Compute JSON body Content Length and finisha headers.
    String size = String(measureJsonPretty(res));
    client->println("Content-Length: " + size);
    client->println();

    // Send serialized JSON body.
    String resBody;
    serializeJsonPretty(res, resBody);
    client->println(resBody);

    Serial.println("OK [200]");
}
```

Invece, in caso di richiesta `POST` viene chiamata la funzione
`parseRequest()`, il cui codice è mostrato di seguito:

```cpp
void parseRequest(String body)
{
    // Deserialize request body.
    char bodyChar[body.length() + 1];
    body.toCharArray(bodyChar, sizeof(bodyChar));
    DeserializationError error = deserializeJson(req, bodyChar);

    // Test if parsing succeeds.
    if (error)
    {
        Serial.print("JSON deserialization error: ");
        Serial.println(error.f_str());
        return;
    }
    else
    {
        // Print the request and change LEDs states accordingly.
        Serial.print("Request: ");
        for (int i = 0; i <= 3; i++)
        {
            String led = "LED_D" + String(i);
            String value = req[led];
            controlLED(i, value);
            Serial.print(led + " to " + value + ", ");
        }
        Serial.println();
    }
}
```

Questa funzione deserializza il JSON contenuto nel body della richiesta `POST`
utilizzando la funzione `deserializeJson()` fornita dalla libreria
`ArduinoJson`. In seguito, utilizza le indicazioni presenti nel JSON per
pilotare i LED da 0 a 3 del Finder Opta, tramite la funzione `controlLED()`:

```cpp
void controlLED(int led, String value)
{
    if (getState(value) != -1)
    {
        ledStates[led] = value;
    }
    switch (led)
    {
    case 0:
        digitalWrite(LED_D0, getState(ledStates[led]));
        break;
    case 1:
        digitalWrite(LED_D1, getState(ledStates[led]));
        break;
    case 2:
        digitalWrite(LED_D2, getState(ledStates[led]));
        break;
    case 3:
        digitalWrite(LED_D3, getState(ledStates[led]));
        break;
    }
}
```

Questa funzione aggiorna la variabile che contiene lo stato del LED del Finder
Opta passato come parametro, e in seguito aggiorna lo stato del LED con una
`digitalWrite()`, mappando la stringa ricevuta nella richiesta ad uno stato del
LED tramite la funzione `getState()`. Si noti che, in caso di stati invalidi
all'interno della richiesta il LED rimarrà nel proprio stato originale.

Terminate queste operazioni, anche in questo caso, il server chiama la funzione
`sendLEDsStates`, inviando quindi come risposta un JSON contente lo stato dei
LED del Finder Opta. Possiamo quindi concludere che tramite l'endpoint `/led`
il server aggiorna e restituisce lo stato dei LED del dispositivo. Per questo
motivo, la pagina web restituita dall'endpoint `/` interagisce con il server a
questo path, consentendo all'utente di interagire con il dispositivo
direttamente dal proprio browser. L'utente può infatti indicare per ciascun LED
uno stato desiderato ed applicare il comando cliccando il pulsante _Apply_. In
caso di successo, il browser mostrerà un pop-up di conferma e i LED del Finder
Opta si accenderanno o spegneranno come specificato, completando il set di
funzionalità previste per il nostro Web Server.

Per ciascun endpoint, in caso di richieste malformate il server risponderà
invocando la funzione `badRequest()`, mentre in caso di path non validi verrà
chiamata la funzione `notFound()`. Infine, in qualsiasi caso il server dovrà
occuparsi di svuotare il buffer in ricezione al termine di ciascuna richiesta,
invocando la funzione `consumeRxBuffer()`:

```cpp
void consumeRxBuffer(HttpClient *http)
{
    // Consume headers in RX buffer.
    http->skipResponseHeaders();
    // Consume body in RX buffer if it exists.
    if (http->contentLength() > 0)
    {
        http->responseBody();
    }
}
```

#### Formato di richieste e risposte

Il server si aspetta di ricevere richieste `GET`, o alternativamente richieste
`POST` aventi body in un formato del tipo:

```json
{
    "LED_D0": "LOW",
    "LED_D1": "HIGH",
    "LED_D2": "HIGH",
    "LED_D3": "LOW"
}
```

In questo documento JSON troviamo specificati i nomi dei LED con relativi
stati: i valori ammessi sono quelli presentati qui sopra. Un possibile esempio
di interazione è dato dal seguente comando:

```bash
curl -d '{"LED_D0":"LOW", "LED_D1":"HIGH", "LED_D2":"HIGH", "LED_D3":"LOW"}' -H "Content-Type: application/json" http://192.168.10.15:80/led
```

Alternativamente, l'utente può utilizzare la pagina web servita da Opta
all'endpoint `/`, che come spiegato in precedenza si occuperà di servire
richieste e consumare risposte correttamente.

#### Log lato server

Il Web server stamperà a monitor seriale un log delle richieste ricevute,
permettendo di tracciare il comportamento del client. Un esempio di output è
mostrato di seguito:

```text
Client with address 192.168.10.1 available.
Client with address 192.168.10.1 connected to '/'...
OK [200]
Client with address 192.168.10.1 disconnected.
Client with address 192.168.10.1 available.
Client with address 192.168.10.1 connected to '/led'...
OK [200]
Client with address 192.168.10.1 disconnected.
Client with address 192.168.10.1 available.
Client with address 192.168.10.1 connected to '/led'...
Request: LED_D0 to HIGH, LED_D1 to HIGH, LED_D2 to HIGH, LED_D3 to HIGH,
OK [200]
Client with address 192.168.10.1 disconnected.
Client with address 192.168.10.1 available.
Client with address 192.168.10.1 attempted connection to /test
Not Found [404]
Client with address 192.168.10.1 disconnected.
```

In questo caso, il client ha richiesto la pagina web, che è stata popolata da
una successiva chiamata `GET` all'endpoint `/led`, effettuata dal Javascript
della pagina. In seguito l'utente ha applicato un cambio di stato: il contenuto
della richiesta è stampato a terminale. Infine, l'utente ha provato a
raggiungere un endpoint non supportato, ed ha quindi ricevuto un codice di
errore HTTP `404 Not Found`.

## Conclusioni

Questo tutorial mostra come implementare su Finder Opta un Web Server con
multipli endpoint, capace di ricevere e decodificare richieste HTTP via
Ethernet. In particolare, il contenuto di queste richieste è stato utilizzato
per pilotare lo stato dei LED di Finder Opta, permettendo così di interagire
con il dispositivo da una pagina web servita dal server stesso.

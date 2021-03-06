# Caricamento delle risorse: onload e onerror

Il browser permette di tracciare il caricamente di risorse esterne -- script, iframe, immagini e così via.

Esistono 2 eventi per tracciare il caricamento:

- `onload` -- caricato con success,
- `onerror` -- si è verificato un errore.

## Caricamento di uno script

Diciamo che abbiamo necessità di caricare uno script di terze parti e chiamare una funzione che appartiene a questo script.

Possiamo caricarlo dinamicamente, in questo modo:

```js
let script = document.createElement('script');
script.src = "my.js";

document.head.append(script);
```

...Ma come possiamo eseguire la funzione dichiarata all'interno di quello script? Dobbiamo attendere finchè lo script sia caricato e solo allora possiamo chiamare la funzione. 

```smart
Per i nostri script dovremmo utilizzare i [moduli JavaScript](info:modules) in questo caso, ma non sono largamente adottati dalle librerie di terze parti.
```

### script.onload

Il principale helper è l'evento `load`. Si innesca dopo che lo script è stato caricato ed eseguito. 

Per esempio:

```js run untrusted
let script = document.createElement('script');

// si può caricare qualunque script, da qualunque dominio
script.src = "https://cdnjs.cloudflare.com/ajax/libs/lodash.js/4.3.0/lodash.js"
document.head.append(script);

*!*
script.onload = function() {
  // lo script crea una funzione helper "_"
  alert(_); // la funzione è disponibile
};
*/!*
```

Quindi nell'evento `onload` possiamo utilizzare le variabili dello script, eseguire funzioni, ecc.

...E cosa accade se il caricamento fallisce? Per esempio, quello script non esiste sul server (errore 404) o il server è fuori servizio (non disponibile).

### script.onerror

Gli errori che si verificano durante il caricamento dello script possono essere tracciati tramite l'evento `error`.

Per esempio, proviamo a richiedere uno script che non esiste:

```js run
let script = document.createElement('script');
script.src = "https://example.com/404.js"; // script che non esiste
document.head.append(script);

*!*
script.onerror = function() {
  alert("Caricamento fallito " + this.src); // Error loading https://example.com/404.js
};
*/!*
```

Notate bene che in questo punto non possiamo ottenere i dettagli dell'errore HTTP. Non sappiamo se è un errore 404 o 500 o qualcos'altro.

```warn
Gli eventi `onload`/`onerror` tracciano solo il caricamento stesso.

Gli errori durante il processamento e l'esecuzione sono fuori dall'ambito di questi eventi. Per tracciare gli errori dello script si può utilizzare l'handler globale `window.onerror`.
```

## Altre risorse

Gli eventi `load` e `error` funzionano anche per le altre risorse, praticamente per qualunque risorsa che ha un `src` esterno.

Per esempio:

```js run
let img = document.createElement('img');
img.src = "https://js.cx/clipart/train.gif"; // (*)

img.onload = function() {
  alert(`Immagine caricata, dimensione ${img.width}x${img.height}`);
};

img.onerror = function() {
  alert("Si è verificato un errore durante il caricamento dell'immagine");
};
```

Ci sono alcune note però:

- La maggior parte delle risorse inizia a caricarsi quando vengono aggiunte al document, ma `<img>` è un'eccezione. Inizia a caricarsi quando ottiene un src `(*)`.
- Per gli `<iframe>`, l'evento `iframe.onload` si aziona quando il caricamento dell'iframe è terminato, sia in caso di successo che in caso di errore. 

Questo avviene per ragioni storiche.

## Crossorigin policy

C'è una regola: gli script di un sito non possono accedere ai contenuti di un altro sito. Quindi, per esempio, uno script di  `https://facebook.com` non può leggere la casella di posta dell'utente di `https://gmail.com`.

Per essere più precisi, un'origine (tripletta dominio/porta/protocollo) non può accedere al contenuto di un'altra. Quindi se abbiamo un sottodominio, o anche solo un'altra porta, questo sarà un'origine differente e quindi non hanno accesso l'uno con l'altro.

Questa regola interessa anche le risorse di altri domini.

Se stiamo utilizzando uno script di un altro dominio e c'è un errore, non possiamo ottenere i dettagli di quell'errore.

Per esempio, prendiamo lo script `error.js`, che consiste in una singola chiamata ad una funzione (sbagliata):
```js
// 📁 error.js
noSuchFunction();
```

Ora caricatela dallo stesso sito su cui è situato lo script:

```html run height=0
<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script src="/article/onload-onerror/crossorigin/error.js"></script>
```

Vedremo il report dell'errore, come questo:

```
Uncaught ReferenceError: noSuchFunction is not defined
https://javascript.info/article/onload-onerror/crossorigin/error.js, 1:1
```

Ora carichiamo lo stesso script da un altro dominio:

```html run height=0
<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script src="https://cors.javascript.info/article/onload-onerror/crossorigin/error.js"></script>
```

Il report di errore è diverso rispetto a quello precedente, come questo:

```
Script error.
, 0:0
```

I dettagli potrebbero dipendere dal browser, ma l'idea è la stessa: qualunque informazione interno dello script, incluso lo stack trace dell'errore, è nascosta. Esattamente, perche lo script è di un altro dominio.

Perchè abbiamo bisogno dei dettagli di errore?

Ci sono molti servizi (e possiamo anche sviluppare il nostro) che stanno in ascolto sugli errori globali, utilizzando `window.onerror`, salvano gli errori e forniscono un interfaccia per accedere ed analizzarli. Fantastico, possiamo vedere i veri errori, scaturiti dai nostri utenti. Ma se uno script è caricato da un altro dominio non avremo nessuna informazioni sull'errore, come abbiamo appena visto.

Una policy cross-origin (CORS) simile viene applicata anche per altri tipi di risorse. 

**Per consentire l'accesso cross-origin il tag `<script>` deve avere l'attributo `crossorigin` e il server remoto deve fornire degli header speciali.**

Ci sono tre livelli di accesso cross-origin:

1. **Attributo `crossorigin` non presente** -- accesso vietato.
2. **`crossorigin="anonymous"`** -- accesso consentito se il server risponde con l'header `Access-Control-Allow-Origin` con il valore `*` o il nome della nostra origin (dominio). Il browser non manda dati e cookie sull'autenticazione al server remoto.
3. **`crossorigin="use-credentials"`** -- accesso consentito se il server manda indietro l'header `Access-Control-Allow-Origin` con la nostra origine (dominio) e `Access-Control-Allow-Credentials: true`. Il browser manda i dati e i cookie sull'autenticazione al server remoto.

```smart
Può approfondire l'accesso cross-origin nel capitolo <info:fetch-crossorigin>. Descrive il metodo `fetch` per le richieste di rete, ma la policy è esattamente la stessa.

Ad esempio i "cookies" sono un argomento fuori dal nostro attuale ambito, ma puoi leggere informazioni a proposito nel capitolo <info:cookie>.
```

Nel nostro caso non avevamo nessun attributo crossorigin, quindi l'accesso era vietato. Aggiungiamo l'attributo ora.

Possiamo scegliere tra `"anonymous"` (non vengono mandati cookie, è necessario un header lato server) e `"use-credentials"` (manda i cookie, sono necessari 2 header lato server).

Se non ci interessano i cookie allora `"anonymous"` è la scelta giusta:

```html run height=0
<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script *!*crossorigin="anonymous"*/!* src="https://cors.javascript.info/article/onload-onerror/crossorigin/error.js"></script>
```

Ora, supponendo che il server fornisca l'header `Access-Control-Allow-Origin`, riusciamo ad avere il report completo dell'errore.

## Riepilogo

Immagini `<img>`, fogli di stile esterni, script e altre risorse forniscono gli eventi `load` e `error` per tracciare i loro caricamento:

- `load` si aziona se il caricamento va a buon fine,
- `error` si azione se si verifica un errore durante il caricamento.

L'unica eccezione è `<iframe>`: per ragioni storiche scatta sempre l'evento `load`, per qualunque esito del caricamento, anche se la pagina non è stata trovata.

Possiamo monitorare il caricamento delle risorse anche tramite l'evento `readystatechange`, ma è poco utilizzato, perchè gli eventi `load/error` sono più semplici.

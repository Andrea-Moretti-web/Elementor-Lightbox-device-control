# Elementor Lightbox Device Controls (Site Settings)

> Disabilita il lightbox di Elementor su mobile e/o tablet — o sotto un breakpoint personalizzato — direttamente da **Site Settings → Lightbox**, senza toccare codice o CSS.

**Versione:** 1.3.0  
**Richiede:** WordPress 5.8+, PHP 7.4+, Elementor (Free o Pro)  
**Testato con:** Elementor 3.35.5  
**Variante:** Full (con tutte le opzioni)

---

## Indice

- [Installazione](#installazione)
- [Configurazione](#configurazione)
- [Come funziona](#come-funziona)
- [Analisi del codice PHP](#analisi-del-codice-php)
- [Analisi del codice JavaScript](#analisi-del-codice-javascript)
- [Struttura file](#struttura-file)
- [Compatibilità](#compatibilità)
- [Changelog](#changelog)
- [Differenze dalla versione Light](#differenze-dalla-versione-light)

---

## Installazione

1. Scarica il file `.zip`
2. In WordPress vai su **Plugin → Aggiungi nuovo → Carica plugin**
3. Carica il `.zip` e clicca **Installa ora**, poi **Attiva**
4. Vai su **Elementor → Site Settings → Lightbox**
5. Scorri fino alla sezione **"Device / Breakpoint Controls (extra)"** e configura le opzioni
6. Salva le impostazioni

> ⚠️ Non installare questa versione insieme alla versione **Light**: i due plugin operano sullo stesso DOM e creerebbero conflitti.

---

## Configurazione

Le opzioni sono accessibili da **Elementor → Site Settings → Lightbox**, nella sezione aggiuntiva in fondo al pannello.

### Modalità Device (Mobile / Tablet)

Attiva quando **"Use custom breakpoint"** è disattivo (impostazione predefinita).

| Opzione | Chiave interna | Descrizione |
|---|---|---|
| Disable Lightbox on Mobile | `disable_lightbox_mobile` | Disabilita il lightbox quando la viewport è ≤ 767px |
| Disable Lightbox on Tablet | `disable_lightbox_tablet` | Disabilita il lightbox quando la viewport è tra 768px e 1024px |
| Block ONLY Media File links on Mobile | `block_links_mobile` | Blocca solo i link a file immagine su mobile, lasciando funzionare gli URL personalizzati |
| Block ONLY Media File links on Tablet | `block_links_tablet` | Stessa logica applicata al tablet |

I toggle **Disable** e **Block** sono **indipendenti**: puoi bloccare solo i link media senza disabilitare l'intero lightbox, o viceversa.

### Modalità Custom Breakpoint

Attiva quando **"Use custom breakpoint"** è abilitato. Sostituisce completamente la modalità Device.

| Opzione | Chiave interna | Default | Descrizione |
|---|---|---|---|
| Use custom breakpoint | `use_custom_breakpoint` | off | Attiva questa modalità |
| Custom breakpoint (px) | `custom_breakpoint_px` | 1024 | Soglia in pixel. Le regole si applicano quando la viewport è ≤ questo valore |
| Disable Lightbox below breakpoint | `bp_disable_lightbox` | on | Disabilita il lightbox sotto la soglia |
| Block ONLY Media File links below breakpoint | `bp_block_links` | on | Blocca i link a file immagine sotto la soglia |

---

## Come funziona

Il plugin agisce su **tre livelli sovrapposti** per garantire la massima copertura su tutti i widget Elementor (immagini singole, gallerie, widget image, contenuti caricati in lazy).

```
┌─────────────────────────────────────────────────────────┐
│  Livello 1 — DOM Neutralization (al DOMContentLoaded)   │
│  • Imposta data-elementor-open-lightbox="no"            │
│  • Rimuove href dai link a file immagine                │
├─────────────────────────────────────────────────────────┤
│  Livello 2 — Click Blocker (capture phase)              │
│  • Blocca click residui su trigger non ancora           │
│    neutralizzati (widget lazy, popup DOM inject)        │
├─────────────────────────────────────────────────────────┤
│  Livello 3 — MutationObserver                           │
│  • Monitora il DOM per widget aggiunti dopo il boot     │
│  • Ri-esegue la neutralizzazione via scheduleApply      │
└─────────────────────────────────────────────────────────┘
```

Il plugin ripristina automaticamente i valori originali se il device cambia (es. rotazione da portrait a landscape che supera la soglia di breakpoint).

---

## Analisi del codice PHP

### File: `elementor-lightbox-device-controls.php`

#### Pattern Singleton

```php
final class AM_EL_Lightbox_Device_Controls {
    private static $instance = null;

    public static function instance() {
        if ( null === self::$instance ) {
            self::$instance = new self();
        }
        return self::$instance;
    }
}
AM_EL_Lightbox_Device_Controls::instance();
```

La classe usa il pattern Singleton per garantire che esista una sola istanza del plugin durante l'esecuzione. Il costruttore è privato, quindi non è istanziabile dall'esterno.

#### Inizializzazione sicura

```php
public function init() {
    if ( ! did_action( 'elementor/loaded' ) ) {
        add_action( 'admin_notices', array( $this, 'admin_notice_missing_elementor' ) );
        return;
    }
    // ...
}
```

Il plugin verifica che Elementor sia stato caricato prima di procedere. Se Elementor non è attivo, mostra un avviso nell'admin invece di generare errori fatali.

#### Hook per i controlli UI

```php
add_action(
    'elementor/element/kit/section_settings-lightbox/before_section_end',
    array( $this, 'register_site_settings_controls' ),
    10, 2
);
```

Questo hook è specifico di Elementor e consente di iniettare controlli custom direttamente nella sezione Lightbox del Kit (Site Settings). È un'API pubblica stabile presente in tutte le versioni 3.x.

#### Registrazione dei controlli

Ogni controllo usa `frontend_available: true`, che indica a Elementor di esporre il valore nel payload JavaScript passato al frontend via `wp_localize_script`. Senza questo flag, le impostazioni sarebbero visibili solo lato server.

```php
$element->add_control( 'am_el_lb_disable_lightbox_mobile', array(
    'type'               => \Elementor\Controls_Manager::SWITCHER,
    'return_value'       => 'yes',
    'default'            => '',
    'frontend_available' => true,
    'condition'          => array( 'am_el_lb_use_custom_breakpoint!' => 'yes' ),
) );
```

La `condition` con il suffisso `!` è la sintassi Elementor per "diverso da": il controllo viene mostrato solo quando `use_custom_breakpoint` NON è `'yes'`.

#### Lettura sicura delle impostazioni

```php
private function get_frontend_settings() {
    try {
        $kit = \Elementor\Plugin::$instance->kits_manager->get_active_kit_for_frontend();
        $settings = $kit ? $kit->get_settings() : array();
    } catch ( \Throwable $e ) {
        $settings = array();
    }
    // ...
}
```

L'accesso al kit viene wrappato in un `try/catch` con `\Throwable` (non solo `\Exception`) per intercettare anche errori fatali PHP 7+. In caso di errore, il plugin usa i valori di default senza rompere la pagina.

#### Enqueue dello script

```php
wp_register_script(
    self::SCRIPT_HANDLE,
    plugin_dir_url( __FILE__ ) . 'assets/frontend.js',
    array( 'jquery', 'elementor-frontend' ),
    self::VERSION,
    true  // in footer
);
wp_localize_script( self::SCRIPT_HANDLE, 'AMElLbDeviceControls', $this->get_frontend_settings() );
wp_enqueue_script( self::SCRIPT_HANDLE );
```

Lo script è registrato con dipendenza da `elementor-frontend`, garantendo che jQuery e il core JS di Elementor siano già caricati quando il plugin si esegue. Il `true` finale carica lo script nel footer, dopo il DOM.

Lo script viene enqueued sia su `elementor/frontend/after_enqueue_scripts` (frontend pubblico) che su `elementor/preview/enqueue_scripts` (editor preview in iframe).

---

## Analisi del codice JavaScript

### File: `assets/frontend.js`

Il JS è wrappato in una IIFE con jQuery come parametro per evitare conflitti con il `$` globale e mantenere lo scope isolato.

#### Oggetto di stato globale

```js
var state = {
    listenerInstalled: false,
    observerInstalled: false,
    scheduled: false,
    lastFlags: null
};
```

I flag `listenerInstalled` e `observerInstalled` prevengono la registrazione di listener o observer multipli, anche quando `applyOnce()` viene richiamato più volte dal MutationObserver. `lastFlags` memorizza l'ultimo stato calcolato così il click blocker non deve ricalcolarlo ad ogni click.

#### `isImageUrl(href)`

```js
function isImageUrl(href) {
    if (!href || typeof href !== "string") return false;
    if (/^(#|mailto:|tel:|javascript:)/i.test(href)) return false;
    return /\.(?:jpe?g|png|gif|webp|svg|avif)(?:\?.*)?(?:#.*)?$/i.test(href);
}
```

Esclude esplicitamente ancore (`#`), `mailto:`, `tel:` e `javascript:` prima di verificare l'estensione. La regex finale supporta query string e fragment hash dopo l'estensione.

#### `getDevice()` — perché NON usa le classi Elementor

```js
function getDevice() {
    if (window.matchMedia("(max-width: 767px)").matches)  return "mobile";
    if (window.matchMedia("(max-width: 1024px)").matches) return "tablet";
    return "desktop";
}
```

Elementor aggiunge le classi `elementor-device-mobile` e `elementor-device-tablet` al `<body>` **solo nell'editor preview**, non sul frontend reale. Sul sito pubblico quelle classi non esistono mai. Il plugin usa quindi `window.matchMedia` con i breakpoint standard di Elementor (767px e 1024px).

#### `getFlags()` — calcolo indipendente per ogni feature

```js
function getFlags() {
    if (cfg.use_custom_breakpoint) {
        var below = viewportBelowBreakpoint();
        return {
            active:          below,
            disableLightbox: below && !!cfg.bp_disable_lightbox,
            blockLinks:      below && !!cfg.bp_block_links
        };
    }
    var device = getDevice();
    if (device === "mobile") {
        return {
            active:          !!cfg.disable_lightbox_mobile || !!cfg.block_links_mobile,
            disableLightbox: !!cfg.disable_lightbox_mobile,
            blockLinks:      !!cfg.block_links_mobile
        };
    }
    // ...
}
```

`disableLightbox` e `blockLinks` sono calcolati **separatamente** e possono essere `true` in modo indipendente. `active` è `true` se almeno uno dei due è attivo, per evitare elaborazioni inutili quando entrambi sono disattivati.

#### `neutralizeLightboxTriggers()` — copertura estesa

```js
document.querySelectorAll("[data-elementor-open-lightbox]").forEach(function (el) {
    var cur = el.getAttribute("data-elementor-open-lightbox");
    if (!cur || cur === "no") return;
    if (!el.hasAttribute("data-am-open-lightbox")) {
        el.setAttribute("data-am-open-lightbox", cur);
    }
    el.setAttribute("data-elementor-open-lightbox", "no");
});
```

Il selettore cattura qualsiasi valore dell'attributo (`"yes"`, `"default"`, o altri), non solo `"yes"`. Il valore originale viene salvato in `data-am-open-lightbox` prima di sovrascriverlo, così `restoreOriginalValues()` può ripristinarlo esattamente. Il check `hasAttribute` previene la sovrascrittura del backup se la funzione viene chiamata più volte sullo stesso elemento.

#### `findAnchor(el)` — ricerca bidirezionale (fix bug 1.2.3)

```js
function findAnchor(el) {
    // Caso A: image widget — <a href="img.jpg"><img/></a>
    // Il trigger è dentro un <a>
    var up = el.closest ? el.closest("a[href]") : null;
    if (up) return up;
    // Caso B: gallerie — <div data-elementor-open-lightbox="yes"><a href="img.jpg">...</a></div>
    // Il trigger è il wrapper, l'<a> è un discendente
    var down = el.querySelector ? el.querySelector("a[href]") : null;
    return down || null;
}
```

La versione 1.2.3 usava solo `closest` (cerca verso l'alto) oppure solo `querySelector` (cerca verso il basso) come fallback, ma non entrambi in modo corretto. Nelle gallerie Elementor il trigger lightbox è il **contenitore** (`<div>`), quindi `closest` non trova mai l'`<a>` interno. La ricerca bidirezionale copre entrambi i casi.

#### `blockMediaFileHrefs()` — selettore combinato

```js
var sel = "[data-am-open-lightbox], [data-elementor-open-lightbox]:not([data-elementor-open-lightbox='no'])";
document.querySelectorAll(sel).forEach(function (el) {
    var a = findAnchor(el);
    // ...
});
```

Il selettore copre sia gli elementi già neutralizzati (che hanno `data-am-open-lightbox`) sia quelli ancora attivi. Questo garantisce che `blockMediaFileHrefs()` funzioni anche se chiamato prima di `neutralizeLightboxTriggers()`, o su widget aggiunti dinamicamente dopo la prima neutralizzazione.

#### `restoreOriginalValues()`

```js
function restoreOriginalValues() {
    document.querySelectorAll("[data-am-open-lightbox]").forEach(function (el) {
        var orig = el.getAttribute("data-am-open-lightbox");
        if (orig) {
            el.setAttribute("data-elementor-open-lightbox", orig);
        } else {
            el.removeAttribute("data-elementor-open-lightbox");
        }
        el.removeAttribute("data-am-open-lightbox");
    });
    document.querySelectorAll("a[data-am-href]").forEach(function (a) {
        var href = a.getAttribute("data-am-href");
        if (href) a.setAttribute("href", href);
        a.removeAttribute("data-am-href");
        a.style.cursor = "";
    });
}
```

Ripristina sia l'attributo lightbox che l'`href`. Viene chiamata da `applyOnce()` quando il device torna a desktop (es. rotazione schermo) o quando nessun flag è attivo.

#### `installClickBlocker()` — fix bug 1.2.3 per `blockLinks`

```js
// Blocco lightbox
if (flags.disableLightbox) {
    var lbEl = target.closest("[data-elementor-open-lightbox]");
    if (lbEl && lbEl.getAttribute("data-elementor-open-lightbox") !== "no") {
        e.preventDefault(); e.stopPropagation(); e.stopImmediatePropagation();
        return;
    }
}

// Blocco link media (fix: non controlla l'attributo sull'<a> ma sul contesto)
if (flags.blockLinks) {
    var a = target.closest("a") || (target.querySelector && target.querySelector("a"));
    if (a) {
        var href = a.getAttribute("href") || a.getAttribute("data-am-href") || "";
        if (href && isImageUrl(href)) {
            var lbCtx = a.closest("[data-elementor-open-lightbox], [data-am-open-lightbox]") ||
                        a.querySelector("[data-elementor-open-lightbox], [data-am-open-lightbox]");
            if (lbCtx) {
                e.preventDefault(); e.stopPropagation(); e.stopImmediatePropagation();
            }
        }
    }
}
```

Il bug della 1.2.3 era che il blocco per `blockLinks` controllava `data-elementor-open-lightbox` direttamente sull'`<a>`, ma Elementor mette quell'attributo sul **wrapper**, non sull'anchor. Il fix cerca il contesto lightbox sia verso l'alto (`closest`) che verso il basso (`querySelector`) rispetto all'`<a>`. Inoltre usa `data-am-href` come fallback nel caso in cui l'`href` sia già stato rimosso da `blockMediaFileHrefs()`.

Il listener è registrato in **capture phase** (`true` come terzo argomento), che viene eseguita prima della bubble phase dove Elementor gestisce i suoi eventi. Questo garantisce che il blocco avvenga prima che Elementor possa aprire il lightbox.

#### `scheduleApply()` — debounce con requestAnimationFrame

```js
function scheduleApply() {
    if (state.scheduled) return;
    state.scheduled = true;
    (window.requestAnimationFrame || function (fn) { setTimeout(fn, 16); })(function () {
        state.scheduled = false;
        applyOnce();
    });
}
```

Il flag `state.scheduled` garantisce che anche se il MutationObserver o il resize event vengono triggerati decine di volte in rapida successione (es. animazioni, scroll), `applyOnce()` venga eseguito una sola volta per frame. Il fallback a `setTimeout(fn, 16)` (~60fps) copre browser molto datati senza `requestAnimationFrame`.

#### `installObserver()` — sicurezza anti-duplicati

```js
function installObserver() {
    if (state.observerInstalled || !window.MutationObserver) return;
    state.observerInstalled = true;
    new MutationObserver(function () {
        scheduleApply(); // non reinstalla mai il listener
    }).observe(document.body, { childList: true, subtree: true });
}
```

Il flag `state.observerInstalled` garantisce che l'observer venga creato una sola volta, anche se `installObserver()` viene chiamata più volte. L'observer chiama solo `scheduleApply()` (che a sua volta chiama `applyOnce()`), senza mai reinstallare listener o altri observer.

#### Sequenza di boot

```
DOMContentLoaded
    └── boot()
         ├── applyOnce()          → neutralizza il DOM esistente
         ├── installClickBlocker() → registra il listener capture
         ├── resize listener      → scheduleApply su resize
         └── orientationchange   → scheduleApply su rotazione

elementor/frontend/init (jQuery event)
    ├── scheduleApply()     → ri-neutralizza widget lazy Elementor
    └── installObserver()   → avvia il MutationObserver
```

Il boot avviene al `DOMContentLoaded`, prima che Elementor inizializzi i propri handler. L'aggancio a `elementor/frontend/init` serve come secondo passaggio per catturare widget renderizzati in ritardo dall'engine di Elementor.

---

## Struttura file

```
am-elementor-lightbox-device-controls/
├── elementor-lightbox-device-controls.php   # Plugin principale
│   ├── class AM_EL_Lightbox_Device_Controls
│   │   ├── init()                           # Verifica Elementor, registra hook
│   │   ├── register_site_settings_controls()# Inietta controlli nel pannello UI
│   │   ├── get_frontend_settings()          # Legge il kit e prepara il payload JS
│   │   └── enqueue_frontend_script()        # Registra ed enqueue il JS
│   └── AM_EL_Lightbox_Device_Controls::instance()
├── assets/
│   └── frontend.js                          # Logica client-side
│       ├── state {}                         # Stato globale anti-duplicati
│       ├── isImageUrl()                     # Utility: riconosce URL immagine
│       ├── getDevice()                      # Rileva mobile/tablet via matchMedia
│       ├── viewportBelowBreakpoint()        # Modalità custom breakpoint
│       ├── getFlags()                       # Calcola disableLightbox e blockLinks
│       ├── neutralizeLightboxTriggers()     # Livello 1a: modifica attributi DOM
│       ├── findAnchor()                     # Ricerca bidirezionale dell'<a>
│       ├── blockMediaFileHrefs()            # Livello 1b: rimuove href immagini
│       ├── restoreOriginalValues()          # Ripristino per cambio device
│       ├── applyOnce()                      # Orchestratore principale
│       ├── scheduleApply()                  # Debounce con rAF
│       ├── installClickBlocker()            # Livello 2: listener capture phase
│       ├── installObserver()                # Livello 3: MutationObserver
│       └── boot()                           # Entry point
└── readme.txt
```

---

## Compatibilità

| Componente | Versione | Note |
|---|---|---|
| WordPress | 5.8 – 6.9 | |
| PHP | 7.4+ | Richiesto per `\Throwable` |
| Elementor Free | 3.x – 3.35.5 | Testato |
| Elementor Pro | 3.x | Compatibile per lightbox nativo |
| Browser | Moderni + IE11 | `matchMedia` e `closest` supportati ovunque |

> Il plugin **non** è compatibile con i **Popup** di Elementor Pro, che usano un sistema separato dal lightbox nativo del kit.

---

## Changelog

### 1.3.0
- **Fix — `findAnchor()` bidirezionale:** corretto il bug per cui la ricerca dell'`<a>` funzionava solo verso l'alto (image widget) o solo verso il basso (gallerie), ma non in entrambe le direzioni. Ora copre tutti i casi.
- **Fix — click blocker per `blockLinks`:** il controllo non usa più `data-elementor-open-lightbox` sull'`<a>` (dove Elementor non lo mette), ma cerca il contesto lightbox sul wrapper padre o sui figli. Usa `data-am-href` come fallback se l'href è già stato rimosso.
- Base: versione 1.2.3 (ChatGPT), che aveva già risolto listener duplicati, copertura valori lightbox estesa, `restoreOriginalValues()`, gestione resize/orientationchange.

### 1.2.3 (ChatGPT)
- Fix listener duplicati tramite `state.listenerInstalled` e `state.observerInstalled`
- `blockLinks` reso indipendente da `disableLightbox`
- Copertura estesa a tutti i valori `data-elementor-open-lightbox` (non solo `"yes"`)
- Aggiunto `restoreOriginalValues()` per cambio device
- Aggiunto `scheduleApply()` con `requestAnimationFrame`
- Aggiunti listener per `resize` e `orientationchange`

### 1.2.2
- Approccio DOM diretto invece del solo blocco click
- Aggiunto `MutationObserver` per widget dinamici

### 1.2.1
- Fix rilevamento device con `window.matchMedia`
- Fix supporto gallerie (elementi non-anchor)

### 1.2.0
- Release iniziale

---

## Differenze dalla versione Light

| Feature | Full (questa) | Light |
|---|---|---|
| Disable Lightbox su Mobile | ✅ | ✅ |
| Disable Lightbox su Tablet | ✅ | ✅ |
| Block Media File links | ✅ | ❌ |
| Modalità Custom Breakpoint | ✅ | ❌ |
| Opzioni indipendenti per device | ✅ | ❌ |
| Complessità JS | Media | Minima |

Usa la versione **Light** se vuoi solo disabilitare il lightbox su mobile/tablet senza altre opzioni.

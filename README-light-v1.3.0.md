# Elementor Lightbox Light

> Disabilita completamente il lightbox di Elementor su Mobile e/o Tablet. Plug & play — nessun'altra opzione.

**Versione:** 1.3.0  
**Richiede:** WordPress 5.8+, PHP 7.4+, Elementor (Free o Pro)  
**Testato con:** Elementor 3.35.5  
**Variante:** Light (solo disable lightbox, zero opzioni aggiuntive)

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
- [Differenze dalla versione Full](#differenze-dalla-versione-full)

---

## Installazione

1. Scarica il file `.zip`
2. In WordPress vai su **Plugin → Aggiungi nuovo → Carica plugin**
3. Carica il `.zip` e clicca **Installa ora**, poi **Attiva**
4. Vai su **Elementor → Site Settings → Lightbox**
5. Attiva i toggle **"Disabilita Lightbox su Mobile"** e/o **"Disabilita Lightbox su Tablet"**
6. Salva le impostazioni

> ⚠️ Non installare questa versione insieme alla versione **Full**: i due plugin operano sullo stesso DOM e creerebbero conflitti.

---

## Configurazione

Le opzioni sono accessibili da **Elementor → Site Settings → Lightbox**, nella sezione **"Disabilita Lightbox per Device"** aggiunta in fondo al pannello.

| Opzione | Chiave interna | Descrizione |
|---|---|---|
| Disabilita Lightbox su Mobile | `disable_mobile` | Disabilita il lightbox quando la viewport è ≤ 767px |
| Disabilita Lightbox su Tablet | `disable_tablet` | Disabilita il lightbox quando la viewport è ≤ 1024px |

I due toggle sono indipendenti. Puoi abilitare solo Mobile, solo Tablet, o entrambi.

---

## Come funziona

Il plugin agisce su **tre livelli sovrapposti** per garantire la massima copertura su tutti i widget Elementor.

```
┌─────────────────────────────────────────────────────────┐
│  Livello 1 — DOM Neutralization (al DOMContentLoaded)   │
│  • Imposta data-elementor-open-lightbox="no"            │
│  • Rimuove href dai link <a> che puntano a immagini     │
├─────────────────────────────────────────────────────────┤
│  Livello 2 — Click Blocker (capture phase)              │
│  • Blocca click su trigger non ancora neutralizzati     │
│    (widget lazy, contenuto iniettato dinamicamente)     │
├─────────────────────────────────────────────────────────┤
│  Livello 3 — MutationObserver                           │
│  • Monitora il DOM per widget aggiunti dopo il boot     │
│  • Ri-esegue la neutralizzazione via requestAnimFrame   │
└─────────────────────────────────────────────────────────┘
```

Il lightbox viene ripristinato automaticamente se il device torna a desktop (es. rotazione schermo che supera la soglia di 1024px).

---

## Analisi del codice PHP

### File: `elementor-lightbox-device-controls.php`

#### Pattern Singleton

```php
final class AM_EL_Lightbox_Light {
    private static $instance = null;

    public static function instance() {
        if ( null === self::$instance ) {
            self::$instance = new self();
        }
        return self::$instance;
    }
}
AM_EL_Lightbox_Light::instance();
```

Una sola istanza della classe per tutta l'esecuzione. Il costruttore privato impedisce l'istanziazione diretta dall'esterno.

#### Verifica dipendenza Elementor

```php
public function init() {
    if ( ! did_action( 'elementor/loaded' ) ) {
        add_action( 'admin_notices', array( $this, 'admin_notice_missing_elementor' ) );
        return;
    }
    // ...
}
```

Il plugin si blocca con un avviso nell'admin se Elementor non è attivo, senza generare errori fatali.

#### Hook per i controlli UI

```php
add_action(
    'elementor/element/kit/section_settings-lightbox/before_section_end',
    array( $this, 'register_controls' ),
    10, 2
);
```

Inietta i toggle direttamente nella sezione Lightbox del pannello Site Settings di Elementor, usando l'API pubblica stabile disponibile in tutte le versioni 3.x.

#### Controlli registrati

```php
$element->add_control( 'am_el_lb_light_disable_mobile', array(
    'label'              => esc_html__( 'Disabilita Lightbox su Mobile', 'am-el-lightbox-light' ),
    'type'               => \Elementor\Controls_Manager::SWITCHER,
    'return_value'       => 'yes',
    'default'            => '',
    'frontend_available' => true,
) );
```

`frontend_available: true` espone il valore al JavaScript frontend tramite `wp_localize_script`. Senza questo flag, le impostazioni resterebbero solo lato server. Non ci sono `condition` perché i due toggle sono sempre visibili e indipendenti.

#### Lettura sicura delle impostazioni

```php
public function enqueue_script() {
    $settings = array( 'disable_mobile' => false, 'disable_tablet' => false );
    try {
        $kit = \Elementor\Plugin::$instance->kits_manager->get_active_kit_for_frontend();
        $s   = $kit ? $kit->get_settings() : array();
        $settings['disable_mobile'] = $this->bool_setting( $s, 'am_el_lb_light_disable_mobile' );
        $settings['disable_tablet'] = $this->bool_setting( $s, 'am_el_lb_light_disable_tablet' );
    } catch ( \Throwable $e ) {}
    // ...
}
```

Il `try/catch` con `\Throwable` intercetta sia eccezioni che errori fatali PHP 7+. I valori di default sono `false` (tutto disattivato), così in caso di errore il comportamento è sempre sicuro: nessuna modifica al DOM.

#### Enqueue dello script

```php
wp_register_script(
    self::SCRIPT_HANDLE,
    plugin_dir_url( __FILE__ ) . 'assets/frontend.js',
    array( 'jquery', 'elementor-frontend' ),
    self::VERSION,
    true
);
wp_localize_script( self::SCRIPT_HANDLE, 'AMElLbLight', $settings );
wp_enqueue_script( self::SCRIPT_HANDLE );
```

La dipendenza da `elementor-frontend` garantisce che jQuery e il core JS di Elementor siano già presenti. Il payload `AMElLbLight` è un oggetto JSON minimale con solo due chiavi booleane.

---

## Analisi del codice JavaScript

### File: `assets/frontend.js`

Il JS è una IIFE con jQuery come parametro per isolare lo scope ed evitare conflitti con il `$` globale.

#### Stato globale

```js
var state = {
    listenerInstalled: false,
    observerInstalled: false,
    scheduled: false,
    active: false
};
```

`listenerInstalled` e `observerInstalled` prevengono la creazione di listener o observer multipli. `active` è il flag booleano corrente (aggiornato da `applyOnce`) usato dal click blocker per decidere se intervenire senza ricalcolare il device ad ogni click.

#### `shouldDisable()` — rilevamento device

```js
function shouldDisable() {
    if (cfg.disable_mobile && window.matchMedia("(max-width: 767px)").matches)  return true;
    if (cfg.disable_tablet && window.matchMedia("(max-width: 1024px)").matches) return true;
    return false;
}
```

Usa `window.matchMedia` con i breakpoint standard di Elementor. Le classi `elementor-device-mobile` e `elementor-device-tablet` che Elementor aggiunge al `<body>` esistono **solo nell'editor preview**, mai sul sito pubblico — per questo motivo non vengono usate.

Il check su `cfg.disable_mobile` e `cfg.disable_tablet` è fatto prima di `matchMedia` per evitare query CSS non necessarie quando l'opzione è disattivata.

#### `disableLightboxInDOM()` — doppia neutralizzazione

```js
function disableLightboxInDOM() {
    // — Parte 1: attributi lightbox —
    document.querySelectorAll(
        "[data-elementor-open-lightbox]:not([data-elementor-open-lightbox='no'])"
    ).forEach(function (el) {
        if (!el.hasAttribute("data-am-lb")) {
            el.setAttribute("data-am-lb", el.getAttribute("data-elementor-open-lightbox"));
        }
        el.setAttribute("data-elementor-open-lightbox", "no");
    });

    // — Parte 2: href a file immagine —
    document.querySelectorAll(".elementor a[href]").forEach(function (a) {
        var href = a.getAttribute("href") || "";
        if (/\.(?:jpe?g|png|gif|webp|svg|avif)(?:\?.*)?(?:#.*)?$/i.test(href)) {
            if (!a.hasAttribute("data-am-href")) {
                a.setAttribute("data-am-href", href);
            }
            a.removeAttribute("href");
            a.style.cursor = "default";
        }
    });
}
```

**Parte 1 — Attributi lightbox:** il selettore `:not([data-elementor-open-lightbox='no'])` esclude gli elementi già neutralizzati, evitando di sovrascrivere il backup in `data-am-lb`. Copre qualsiasi valore diverso da `"no"` (`"yes"`, `"default"`, ecc.).

**Parte 2 — Link href:** rimuovere l'`href` è il modo più affidabile per bloccare l'apertura diretta del file immagine, indipendentemente da come Elementor gestisce il click internamente. Il valore originale viene salvato in `data-am-href` per il ripristino. La limitazione al selettore `.elementor a[href]` evita di modificare link fuori dal contesto Elementor.

#### `restoreLightboxInDOM()`

```js
function restoreLightboxInDOM() {
    document.querySelectorAll("[data-am-lb]").forEach(function (el) {
        el.setAttribute("data-elementor-open-lightbox", el.getAttribute("data-am-lb"));
        el.removeAttribute("data-am-lb");
    });
    document.querySelectorAll("a[data-am-href]").forEach(function (a) {
        a.setAttribute("href", a.getAttribute("data-am-href"));
        a.removeAttribute("data-am-href");
        a.style.cursor = "";
    });
}
```

Ripristino simmetrico rispetto a `disableLightboxInDOM()`. Viene chiamato quando `shouldDisable()` torna `false` (es. rotazione schermo su landscape con viewport > 1024px).

#### `applyOnce()` — orchestratore

```js
function applyOnce() {
    var active = shouldDisable();
    state.active = active;
    if (active) {
        disableLightboxInDOM();
    } else {
        restoreLightboxInDOM();
    }
}
```

Funzione singola di ingresso per tutta la logica. Aggiorna `state.active` che viene usato dal click blocker. Non ha logica condizionale interna complessa: o disabilita, o ripristina.

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

Garantisce che `applyOnce()` non venga eseguito più di una volta per frame, anche se MutationObserver o gli event listener di resize vengono triggerati molte volte in rapida successione. Il fallback a `setTimeout(fn, 16)` (~60fps) copre browser senza `requestAnimationFrame`.

#### `installClickBlocker()` — secondo livello di difesa

```js
document.addEventListener("click", function (e) {
    if (!state.active) return;
    var target = e.target;
    if (!target || !target.closest) return;

    // Trigger lightbox non ancora neutralizzato
    var lbEl = target.closest(
        "[data-elementor-open-lightbox]:not([data-elementor-open-lightbox='no'])"
    );
    if (lbEl) {
        e.preventDefault();
        e.stopPropagation();
        e.stopImmediatePropagation();
        return;
    }

    // Link diretto a file immagine
    var a = target.closest("a[href]");
    if (a && /\.(?:jpe?g|png|gif|webp|svg|avif)(?:\?.*)?(?:#.*)?$/i.test(a.getAttribute("href") || "")) {
        e.preventDefault();
        e.stopPropagation();
        e.stopImmediatePropagation();
    }
}, true); // capture phase
```

Il listener è in **capture phase** (`true` come terzo parametro), che viene eseguita prima della bubble phase dove Elementor intercetta i click per aprire il lightbox. Questo garantisce che il blocco avvenga prima che Elementor possa agire.

Il check `state.active` evita qualsiasi elaborazione quando il plugin è inattivo (desktop), riducendo l'overhead al minimo.

#### `installObserver()` — widget dinamici

```js
function installObserver() {
    if (state.observerInstalled || !window.MutationObserver) return;
    state.observerInstalled = true;
    new MutationObserver(function () {
        scheduleApply();
    }).observe(document.body, { childList: true, subtree: true });
}
```

Cattura widget aggiunti dinamicamente dopo il boot iniziale (lazy render di Elementor, popup, contenuto caricato via AJAX). Il flag `state.observerInstalled` impedisce la creazione di observer multipli. L'observer chiama solo `scheduleApply()`, mai direttamente `applyOnce()`, per evitare flood di operazioni DOM.

#### Sequenza di boot

```
DOMContentLoaded
    └── boot()
         ├── applyOnce()           → neutralizza DOM esistente
         ├── installClickBlocker() → listener capture phase (unico)
         ├── resize listener       → scheduleApply su resize viewport
         └── orientationchange     → scheduleApply su rotazione

elementor/frontend/init
    ├── scheduleApply()       → ri-neutralizza widget lazy Elementor
    └── installObserver()     → avvia MutationObserver (unico)
```

Il boot al `DOMContentLoaded` avviene prima che Elementor inizializzi i suoi handler. L'aggancio a `elementor/frontend/init` (evento jQuery emesso da Elementor) copre i widget renderizzati in un secondo momento dal motore di Elementor.

---

## Struttura file

```
am-elementor-lightbox-light/
├── elementor-lightbox-device-controls.php   # Plugin principale
│   ├── class AM_EL_Lightbox_Light
│   │   ├── init()                           # Verifica Elementor, registra hook
│   │   ├── register_controls()              # Inietta i due toggle nel pannello UI
│   │   ├── bool_setting()                   # Helper: legge boolean dal kit
│   │   └── enqueue_script()                 # Legge settings, registra ed enqueue JS
│   └── AM_EL_Lightbox_Light::instance()
├── assets/
│   └── frontend.js                          # Logica client-side
│       ├── state {}                         # Flag globali anti-duplicati
│       ├── shouldDisable()                  # Controlla se il device è mobile/tablet
│       ├── disableLightboxInDOM()           # Livello 1: modifica attributi + href
│       ├── restoreLightboxInDOM()           # Ripristino per cambio device
│       ├── applyOnce()                      # Orchestratore principale
│       ├── scheduleApply()                  # Debounce con rAF
│       ├── installClickBlocker()            # Livello 2: listener capture phase
│       ├── installObserver()                # Livello 3: MutationObserver
│       └── boot()                           # Entry point al DOMContentLoaded
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
- Prima release pubblica della variante Light
- Basata sulla stessa architettura a tre livelli della versione Full 1.3.0
- Codice semplificato: nessuna logica per `blockLinks` o custom breakpoint
- Payload JS minimale: solo due chiavi booleane (`disable_mobile`, `disable_tablet`)

---

## Differenze dalla versione Full

| Feature | Light (questa) | Full |
|---|---|---|
| Disable Lightbox su Mobile | ✅ | ✅ |
| Disable Lightbox su Tablet | ✅ | ✅ |
| Block Media File links | ❌ | ✅ |
| Modalità Custom Breakpoint | ❌ | ✅ |
| Opzioni indipendenti per device | ❌ | ✅ |
| Complessità JS | Minima | Media |
| Dimensione JS | ~2 KB | ~4 KB |

Usa la versione **Full** se hai bisogno di bloccare solo i link a file media senza disabilitare tutto il lightbox, o se vuoi un breakpoint personalizzato in pixel.

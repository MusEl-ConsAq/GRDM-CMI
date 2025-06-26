# CAPITOLO 6: SINTASSI E SEMANTICA COMPOSITIVA

YAML (YAML Ain't Markup Language) emerge in Gamma non solo come formato di configurazione, ma come vero e proprio linguaggio di partitura per la composizione algoritmica. La scelta di YAML rispetto ad altri formati riflette la necessità di bilanciare leggibilità umana con precisione computazionale, creando un ponte tra l'intuizione compositiva e l'esecuzione algoritmica.

## 6.1 Struttura Gerarchica

La struttura compositiva in Gamma segue una gerarchia rigorosa che rispecchia l'organizzazione tradizionale della musica occidentale, adattandola alle esigenze della generazione algoritmica.

### La Gerarchia Fondamentale

Al livello più alto, una composizione è una lista di sezioni:

```yaml
- nome_sezione: "Introduzione"
  durata: 30
  layers:
    - nome_layer: "Texture di base"
      # parametri del layer
    - nome_layer: "Eventi puntuali"
      # parametri del layer

- nome_sezione: "Sviluppo"
  durata: 60
  layers:
    # altri layers
```

Questa struttura apparentemente semplice nasconde una ricchezza semantica considerevole. Ogni livello gerarchico porta con sé un dominio di parametri specifico e regole di ereditarietà implicite.

### Parametri per Livello Gerarchico

**Livello Sezione**: I parametri a questo livello influenzano tutti i layer contenuti:
- `durata`: Definisce il contenitore temporale
- `ratio_temporale`: Permette dilatazioni o compressioni senza modificare i valori numerici
- `inviluppo_sezione`: Applica una modulazione globale d'ampiezza

**Livello Layer**: Qui si definisce l'identità del flusso sonoro:
- `num_attivazioni`: Controlla la densità eventi
- `timing_model`: Determina la distribuzione temporale
- `lifespan`: Definisce quando il layer è attivo
- Stati (unico/iniziale/finale): Contengono le maschere di tendenza

La separazione dei domini parametrici non è arbitraria. Riflette una comprensione che certi aspetti musicali (come la durata totale di una sezione) sono strutturali, mentre altri (come la distribuzione delle altezze) sono texturali.

### Eredità e Override dei Valori

Il sistema implementa un modello di eredità implicita dove i valori di default si propagano attraverso la gerarchia:

```yaml
- nome_sezione: "Sezione con defaults"
  durata: 60
  # ratio_temporale assume valore 1.0
  # inviluppo_sezione assume 'continua'
  layers:
    - nome_layer: "Layer minimale"
      # num_attivazioni assume 10
      # timing_model assume {type: 'lineare'}
      stato_unico:
        ottava: {range: [4, 6]}
        # tutti gli altri parametri assumono defaults
```

Questa eredità permette specifiche concise quando i defaults sono appropriati, ma mantiene la possibilità di override granulare quando necessario. Il meccanismo di normalizzazione in Python garantisce che anche parametri specificati in forma abbreviata vengano espansi nella forma completa prima del processing.

## 6.2 Definizione delle Maschere

Le maschere di tendenza rappresentano il cuore semantico del sistema, trasformando il YAML da semplice formato di dati a linguaggio espressivo per la composizione.

### Sintassi delle Modalità di Generazione

La sintassi per le maschere supporta quattro modalità principali, ciascuna con la propria semantica:

**Range**: Definisce un intervallo di valori equiprobabili:
```yaml
ottava: {range: [3, 7]}
registro: {range: [1.0, 10.0]}  # float permette microtonalità
```

**Choices**: Permette selezione da un insieme discreto:
```yaml
dinamica: {choices: ['p', 'mf', 'f']}
# Con pesi per distribuzione non uniforme
tipo_ritmi: {choices: ['piccoli', 'medi'], weights: [0.3, 0.7]}
```

**Distribuzione Normale**: Per concentrazione attorno a un centro:
```yaml
durata_armonica: {mean: 2.0, std: 0.5}
```

**Valore Fisso**: Quando non si desidera variazione:
```yaml
senso_movimento: {value: -1}
```

### Normalizzazione Automatica

Il sistema permette sintassi abbreviate che vengono espanse automaticamente:

```yaml
# Forma abbreviata
dinamica: 'mf'

# Viene normalizzata in
dinamica: {value: 'mf'}
```

Questa normalizzazione avviene nel metodo `_normalize_mask()` e permette di mantenere il YAML leggibile senza sacrificare la consistenza interna del sistema.

### Parametri Interpolabili vs Fissi

Non tutti i parametri supportano l'interpolazione. La distinzione riflette la natura musicale dei parametri:

**Interpolabili**:
- Parametri numerici continui (ottava, registro, durata)
- Distribuzioni (mean, std di una normale)
- Pesi di scelte discrete (quando le scelte sono identiche)

**Non Interpolabili**:
- Stringhe che rappresentano categorie (tipo_ritmi quando usa categorie)
- Liste di valori (explicit_values per ritmi)
- Parametri strutturali (timing_model)

Questa distinzione è gestita automaticamente dal sistema di interpolazione, che applica strategie appropriate per ogni tipo.

## 6.3 Controlli Avanzati

Oltre ai parametri musicali di base, Gamma offre controlli avanzati che permettono di gestire aspetti sottili ma cruciali della generazione.

### Lifespan e Ciclo di Vita dei Layer

Il parametro `lifespan` permette controllo fine su quando un layer è attivo:

```yaml
layers:
  - nome_layer: "Introduzione graduale"
    lifespan: [0.0, 0.3]  # Solo nel primo 30%
  
  - nome_layer: "Corpo principale"  
    lifespan: [0.2, 0.9]  # Dal 20% al 90%, sovrapposizione con intro
  
  - nome_layer: "Coda"
    lifespan: [0.8, 1.0]  # Ultimo 20%, sovrapposizione con corpo
```

Questa specifica crea una forma ad arco con sovrapposizioni controllate, impossibile da ottenere con semplice sequenzialità.

### Safety Buffer e Leeway

Due meccanismi complementari gestiscono i bordi temporali:

```yaml
- nome_layer: "Eventi lunghi"
  usa_safety_buffer: true  # Default, previene sconfinamenti
  leeway_fine_layer: 2.0   # Permette 2 secondi extra alla fine
```

Il safety buffer sottrae tempo dalla generazione per garantire che nessun evento ecceda i limiti. Il leeway aggiunge tempo extra alla fine per permettere code naturali. La combinazione permette controllo preciso del comportamento ai bordi mantenendo flessibilità espressiva.

### Modalità Solo e Veteran Mode

Due modalità speciali facilitano il workflow compositivo:

```yaml
- nome_layer: "Layer in sviluppo"
  solo: true  # Solo questo layer sarà renderizzato
  
- nome_layer: "Layer completato"
  veteranMode: true  # Riusa il WAV esistente se presente
```

La modalità `solo` isola layer specifici per testing e rifinitura. Il `veteranMode` ottimizza i tempi di rendering riutilizzando materiale già generato. Entrambe le modalità operano a livello di orchestrazione Python senza influenzare la generazione Csound.

### Implicazioni Compositive della Sintassi

La sintassi YAML di Gamma non è neutra - incorpora assunzioni e affordance che guidano il processo compositivo. La struttura gerarchica suggerisce di pensare in termini di sezioni e layer. La sintassi delle maschere incoraggia il pensiero probabilistico. I controlli avanzati permettono raffinamenti che sarebbero complessi in notazione tradizionale.

Questa non è semplicemente una questione di convenienza. Il linguaggio che usiamo per descrivere la musica influenza profondamente come pensiamo alla musica stessa. YAML in Gamma diventa così non solo un formato di dati, ma un medium compositivo che shapes il pensiero musicale verso paradigmi stocastici e stratificati, mantenendo al contempo una connessione con i concetti tradizionali di struttura e forma musicale.

# INTRODUZIONE E ARCHITETTURA GENERALE

Il `generative_composerYaml2.py` è un sistema di composizione algoritmica che traduce una descrizione astratta di una struttura musicale, definita in formato YAML, in un file audio (WAV). Lo fa generando uno score per il software di sintesi sonora Csound.

L'architettura del programma è basata su tre componenti principali e un'esecuzione a fasi:

1.  **`GenerativeComposer`**: La classe principale che contiene la logica per interpretare la partitura YAML, generare i parametri stocastici degli eventi sonori e creare i file di score `.csd` per Csound.
2.  **`CompositionDebugger`**: Una classe di utilità dedicata esclusivamente alla creazione di una visualizzazione grafica (in formato PDF) della composizione generata, simile a un "piano roll" arricchito con informazioni sulle tendenze parametriche.
3.  **`TimeScheduler`**: Una classe specializzata nella generazione di sequenze temporali (gli *onset*, o istanti di inizio) degli eventi, secondo diversi modelli (lineare, accelerando, ritardando, etc.).

Il processo, orchestrato nel blocco `if __name__ == "__main__":`, non è monolitico ma suddiviso in fasi distinte e sequenziali, che permettono di separare la generazione, il rendering e la visualizzazione.

## Fase di Input: Caricamento della Struttura della Composizione

Il punto di partenza è un file YAML. Il programma supporta la definizione di composizioni complesse, articolate in più parti, utilizzando la sintassi multi-documento di YAML (documenti separati da `---`).

La funzione `load_all_compositions_from_yaml` si occupa di questo compito:

```python
def load_all_compositions_from_yaml(file_path):
    """
    Carica una o più composizioni da un singolo file YAML.
    I documenti multipli devono essere separati da '---'.
    Restituisce una lista di strutture di composizione.
    """
    print(f"Caricamento partiture dal file multi-documento: {file_path}")
    composizioni = []
    try:
        with open(file_path, 'r') as f:
            docs = list(yaml.safe_load_all(f))
        for i, composition in enumerate(docs):
            if composition is None: continue 
            composizioni.append(composition)
        return composizioni
```

Ogni documento YAML caricato rappresenta una "Parte" della composizione. Ogni parte è una lista di "Sezioni", e ogni sezione può contenere uno o più "Layer". Questa struttura gerarchica (Parte -> Sezione -> Layer -> Evento) è il modello concettuale su cui si basa tutta la logica successiva.

In aggiunta, viene caricato un file `tables.yaml` che definisce le caratteristiche degli inviluppi (es. attacco, rilascio) che verranno usati da Csound.

## Il Nucleo Generativo: Dal Concetto ai Parametri

Il cuore del sistema risiede nella classe `GenerativeComposer` e nella sua capacità di trasformare le "maschere" parametriche definite nel YAML in valori numerici concreti per ogni evento sonoro.

La generazione avviene all'interno di un "layer". Un layer può essere:
-  **Statico**: Definito da uno `stato_unico`. Tutti gli eventi generati in questo layer attingeranno da un'unica maschera di parametri.
-  **Dinamico**: Definito da uno `stato_iniziale` e uno `stato_finale`. I parametri degli eventi evolvono nel tempo, interpolando tra queste due maschere.

La funzione `_process_layer` gestisce un singolo layer. I suoi passaggi chiave sono:
1.  **Calcolo del Timing**: Determina la durata effettiva del layer basandosi sul `lifespan` (una finestra temporale relativa alla sezione, es. `[0.0, 0.5]` per la prima metà).
2.  **Generazione degli Onset**: Utilizza `TimeScheduler` per calcolare gli istanti di attivazione dei cluster di eventi all'interno della durata del layer.
3.  **Generazione degli Eventi**: Per ogni onset, determina la maschera parametrica (statica o interpolata) e genera un "cluster" di eventi sonori.

### Generazione Stocastica dei Parametri (`_generate_params_from_mask`)

Questa funzione è il motore stocastico. Prende una "maschera" (un dizionario che descrive un parametro) e produce un valore numerico. Supporta diversi tipi di generazione:

-  **Distribuzione Uniforme**: Se la maschera contiene una chiave `range`.

```python
elif 'range' in p_mask:
    min_val, max_val = p_mask['range']
    # ...
    val = random.uniform(min_val, max_val)
```
-   **Distribuzione Normale**: Se la maschera contiene `mean` e `std`.
 
```python
if 'mean' in p_mask and 'std' in p_mask:
    mean = p_mask['mean']
    std = p_mask['std']
    val = np.random.normal(loc=mean, scale=std)
```
-   **Scelta Pesata**: Se la maschera contiene `choices` ed opzionalmente `weights`.

```python
elif 'choices' in p_mask:
    val = random.choices(p_mask['choices'], weights=p_mask.get('weights'), k=1)[0]
```

Questa logica viene applicata a tutti i parametri (altezza, durata, etc.), rendendo il sistema flessibile.

### Interpolazione dei Parametri (`_interpolate_mask`)

Per i layer dinamici, questa funzione calcola una maschera intermedia tra `start_mask` e `end_mask` in base a un valore di `progress` (da 0 a 1). L'interpolazione è intelligente e si adatta al tipo di parametro:
-  I parametri numerici (come `mean`, `std`, `range`) vengono interpolati linearmente.
-  I parametri basati su scelte (`choices`) subiscono un *cross-fade* dei loro pesi (`weights`), creando una transizione probabilistica graduale da un set di scelte a un altro.

```python
# Esempio di interpolazione di un range
if 'range' in s_mask:
    s_min, s_max = s_mask['range']
    e_min, e_max = e_mask.get('range', s_mask['range'])
    i_min = s_min + (e_min - s_min) * shaped_progress
    i_max = s_max + (e_max - s_max) * shaped_progress
    interp_mask[key]['range'] = [i_min, i_max]
```

### Generazione dello Score Csound (`generate_csd`)

Una volta generata la sequenza completa di eventi, la funzione `generate_csd` assembla il file `.csd`. Non scrive codice Csound complesso, ma piuttosto popola un template.

1.  **Tabelle Dinamiche (`f-statements`)**: Crea le tabelle per i pattern ritmici e gli inviluppi.
2.  **Eventi (`i-statements`)**: Itera su ogni evento generato e scrive una riga di score (`i "Voce" ...`) con tutti i parametri calcolati.

```python
# Frammento della riga di score generata
score_lines += (f'i "Voce"\t{event_time:.4f}\t{p["durata_totale"]:.3f}\t'
                f'{p["ritmi_tab_num"]}\t{p["durata_armonica"]:.3f}\t\t{p["dynamic_index"]:.6f}\t'
                # ... altri parametri ...
               )
```
Questo file `.csd` è un output intermedio, pronto per essere processato da Csound per generare un file audio.

## L'Orchestrazione del Rendering (Blocco `__main__`)

L'approccio del compositore al rendering è granulare e mira a ottimizzare i tempi di lavoro, specialmente su composizioni complesse. Questo avviene attraverso una sequenza di fasi ben definita.

### Fase 1: `plan_render_jobs`

Questa funzione analizza l'intera struttura della partitura e crea un "piano di lavoro". Non esegue alcun rendering, ma definisce *cosa-deve essere renderizzato. Per ogni layer che necessita di rendering, crea un "job", ovvero un dizionario contenente:
-  La definizione del layer e della sezione a cui appartiene.
-  I percorsi per i file `.csd` e `.wav` di output per quel singolo layer.
-  Il tempo di inizio assoluto della sezione, calcolato tenendo conto del parametro `offset_inizio`.

### Fase 2: `execute_layer_rendering_and_collect_data`

Questa fase esegue i "job" di rendering dei layer.
1.  Per ogni job, invoca la logica di `GenerativeComposer` per generare gli eventi solo per quel layer.
2.  Genera un file `.csd` specifico per il layer.
3.  Lancia un processo Csound (`subprocess.Popen`) per renderizzare il `.csd` del layer in un file `.wav`.
4.  **Crucialmente**, raccoglie tutti i dati degli eventi generati in una struttura dati (`plot_data`). Questi dati sono essenziali per la visualizzazione.

Questo approccio permette di renderizzare solo i layer modificati se si utilizza la `veteranMode`, una modalità che salta il rendering dei layer non contrassegnati come `veteranMode: True`.

### Fase 3: `generate_composition_plot` e la Cache di Visualizzazione

Questa fase si occupa della visualizzazione. La sua caratteristica principale è l'uso di un file cache, `visual_cache.json`:
1.  **Lettura della Cache**: Carica i dati di visualizzazione da esecuzioni precedenti, se disponibili.
2.  **Merge**: Se sono stati generati nuovi dati (da `execute_layer_rendering...`), questi vengono uniti alla cache, sostituendo i dati vecchi per i layer che sono stati ri-renderizzati.
3.  **Plotting**: Usa la classe `CompositionDebugger` per creare un PDF multi-pagina che mostra:
    -  Gli eventi sonori come rettangoli.
    -  Le "maschere di tendenza" (le buste grigie/arancioni) che mostrano l'evoluzione dei range parametrici.
    -  L'evoluzione delle dinamiche.
    -  Marcatori per sezioni e layer.
4.  **Scrittura della Cache**: Salva lo stato aggiornato dei dati di visualizzazione nel file JSON. Questo garantisce che, alla prossima esecuzione in `veteranMode`, il grafico mostri comunque l'intera composizione (parti vecchie e nuove). La funzione `_sanitize_data_for_json` è un helper fondamentale qui, poiché converte tipi di dati specifici di NumPy e `pathlib` in formati compatibili con JSON.

### Fase 4 e 5: Assemblaggio

Il rendering finale non avviene generando un unico, enorme file CSD. Avviene invece tramite un processo di assemblaggio gerarchico:

1.  `execute_section_assembly`: Per ogni sezione, genera un CSD "assembler". Questo CSD non produce suono, ma si limita a leggere e mixare i file `.wav` dei singoli layer (generati nella Fase 2) per creare un unico file `.wav` per l'intera sezione.

2.  `execute_final_assembly`: Genera un ultimo CSD "assembler" che prende i file `.wav` di tutte le sezioni e li posiziona in sequenza (rispettando gli `offset_inizio`) per creare il file `.wav` finale e completo della composizione.

Questo approccio a "render per layer, poi assembla" ha il vantaggio di essere più gestibile e di non richiedere la rigenerazione dell'intera composizione per piccole modifiche.

Il `generative_composerYaml2.py` implementa un flusso di lavoro completo e disaccoppiato per la composizione algoritmica. Le sue caratteristiche tecniche salienti sono:

-   **Configurazione Esterna (YAML)**: Offre un'interfaccia di alto livello per la descrizione musicale, separando la logica del codice dai dati della composizione.
-   **Generazione Stocastica Multi-modello**: Fornisce un set flessibile di strumenti per definire il comportamento dei parametri sonori.
-   **Rendering Granulare e a Fasi**: Scompone il problema del rendering in sotto-problemi più piccoli (layer, sezioni), ottimizzando il processo di lavoro iterativo.
-   **Caching della Visualizzazione**: Garantisce che il feedback visivo sia sempre coerente e completo, anche quando si lavora solo su parti della composizione.
-   **Assemblaggio Gerarchico**: Utilizza Csound non solo per la sintesi ma anche come uno strumento di montaggio audio per assemblare i componenti finali.
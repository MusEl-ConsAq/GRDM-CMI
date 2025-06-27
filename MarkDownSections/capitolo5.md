# CONTROLLO TEMPORALE E PROCESSAMENTO LAYER

Il controllo temporale in un sistema compositivo generativo rappresenta una sfida particolare: deve essere sufficientemente flessibile per esprimere gesti musicali complessi, ma anche deterministico e riproducibile. Questo capitolo esamina come Gamma affronta questa sfida attraverso modelli temporali parametrizzabili e un sistema sofisticato di processamento dei layer compositivi.

## TimeScheduler e Modelli Temporali

La classe TimeScheduler incapsula la logica di distribuzione temporale degli eventi, offrendo diversi modelli che corrispondono a gesti musicali archetipici. La scelta di separare questa funzionalità in una classe dedicata riflette il riconoscimento che il tempo musicale richiede un trattamento specializzato.

### Implementazione dei Modelli Base

Il metodo centrale `generate_onsets` implementa cinque modelli temporali distinti:

```python
def generate_onsets(self, model, duration, num_events):
    if num_events == 0: return []
    if num_events == 1: return [0.0]
    
    base_progress = np.linspace(0, 1, num_events, endpoint=False)
    final_progress = np.zeros_like(base_progress)
    model_type = model.get('type', 'linear')
```

La generazione inizia sempre con una progressione lineare normalizzata, che viene poi trasformata secondo il modello scelto. Questa architettura a due fasi separa la logica di distribuzione dalla durata effettiva, permettendo riutilizzo e composizione dei modelli.

**Modello Lineare**: La distribuzione di default, dove gli eventi sono equidistanti:
```python
if model_type == 'linear':
    final_progress = base_progress
```

Sebbene semplice, questo modello è fondamentale per creare pulsazioni regolari o griglie ritmiche di riferimento.

**Modello Ritardando**: Gli eventi rallentano progressivamente:
```python
elif model_type == 'ritardando':
    shape = model.get('shape', 2.0)
    final_progress = base_progress ** shape
```

La funzione potenza con esponente > 1 crea una curva che sale rapidamente all'inizio e poi si appiattisce. Musicalmente, questo corrisponde a eventi che iniziano ravvicinati e gradualmente si diradano. Il parametro `shape` controlla l'intensità dell'effetto: valori più alti creano ritardandi più drammatici.

**Modello Accelerando**: L'inverso del ritardando:
```python
elif model_type == 'accelerando':
    shape = model.get('shape', 2.0)
    final_progress = 1 - (1 - base_progress) ** shape
```

La formula `1 - (1 - x) ** shape` è matematicamente elegante: inverte la progressione, applica la potenza, e inverte di nuovo. Il risultato è una curva che parte lentamente e accelera, creando tensione crescente.

**Modello Stocastico**: Distribuzione casuale ma ordinata:
```python
elif model_type == 'stochastic':
    final_progress = np.sort(np.random.rand(num_events))
```

Generare valori casuali e poi ordinarli garantisce che gli eventi mantengano una progressione temporale coerente pur essendo distribuiti irregolarmente. Questo modello è particolarmente efficace per creare texture "naturali" o "organiche".

### L'Algoritmo Breakpoint Avanzato

Il modello più sofisticato è il breakpoint, che permette di definire curve temporali arbitrarie attraverso punti di controllo:

```python
if model_type == 'breakpoint':
    points = model['points']
    
    for i in range(len(points) - 1):
        start_point = points[i]
        end_point = points[i+1]
        
        t_start, v_start = start_point[0], start_point[1]
        t_end, v_end = end_point[0], end_point[1]
        shape = end_point[2] if len(end_point) > 2 else 1.0
```

Ogni segmento tra due punti può avere la propria curva di shaping. Consideriamo un esempio concreto:

```yaml
timing_model:
  type: breakpoint
  points:
    - [0.0, 0.0]      # Inizio
    - [0.3, 0.7, 0.5] # 70% degli eventi nei primi 30% del tempo, curva ease-out
    - [0.8, 0.9, 2.0] # Solo 20% degli eventi nel mezzo, curva ease-in
    - [1.0, 1.0]      # Fine
```

Questo creerebbe un profilo temporale con alta densità iniziale, rarefazione centrale, e piccola ripresa finale.

L'implementazione gestisce l'interpolazione shaped per ogni segmento:

```python
# Trova quali eventi cadono in questo intervallo
segment_mask = (base_progress >= t_start) & (base_progress < t_end)
segment_times = base_progress[segment_mask]

# Normalizza il tempo all'interno del segmento
time_in_segment = (segment_times - t_start) / (t_end - t_start)

# Applica la funzione di shaping
shaped_time = time_in_segment ** shape

# Interpola i valori usando il tempo curvato
interpolated_values = v_start + (v_end - v_start) * shaped_time
final_progress[segment_mask] = interpolated_values
```

La normalizzazione del tempo all'interno di ogni segmento permette di applicare trasformazioni locali senza influenzare gli altri segmenti. Questo approccio modulare rende possibile la costruzione di profili temporali estremamente complessi mantenendo il controllo locale.

### Implicazioni Musicali dei Modelli Temporali

Ogni modello temporale suggerisce particolari applicazioni musicali:

- **Lineare**: Ideale per strutture metriche, ostinati, o quando si vuole enfatizzare regolarità
- **Accelerando/Ritardando**: Per creare archi di tensione, approcci a climax, o dissolvenze
- **Stocastico**: Per texture pointillistiche, imitazione di processi naturali, rottura della griglia metrica
- **Breakpoint**: Per gesti complessi, forme composite, sincronizzazione con altri parametri musicali

La possibilità di cambiare modello per ogni layer permette la sovrapposizione di diverse logiche temporali, creando poliritmie e politemporalità complesse.

## Processamento Layer

Il processamento dei layer rappresenta il cuore operativo del sistema, dove le specifiche astratte si trasformano in eventi concreti. Il metodo `_process_layer` gestisce questa trasformazione con attenzione particolare ai dettagli temporali e alla gestione delle risorse.

### Calcolo del Lifespan e Timing Assoluto

Ogni layer può essere attivo solo per una porzione della sezione che lo contiene:

```python
def _process_layer(self, layer, layer_idx, current_time_offset, 
                   scaled_section_duration, time_ratio, 
                   section_env_table_num, section_name):
    
    # Estrai e interpreta il lifespan
    lifespan = layer.get('lifespan', [0.0, 1.0])
    start_ratio, end_ratio = lifespan
    
    # Calcola durata e tempo di inizio ASSOLUTI
    layer_start_time_abs = current_time_offset + (start_ratio * scaled_section_duration)
    layer_duration_abs = (end_ratio - start_ratio) * scaled_section_duration
    
    if layer_duration_abs <= 0:
        print(f"ATTENZIONE: Layer '{layer_name}' ha durata nulla o negativa.")
        return [], []
```

Il concetto di lifespan permette coreografie temporali sofisticate. Un layer con `lifespan: [0.2, 0.8]` sarà attivo solo nella parte centrale della sezione (dal 20% all'80%), permettendo entrate e uscite scaglionate. La conversione da ratios relativi a tempi assoluti è cruciale per la sincronizzazione globale.

### Il Sistema di Safety Buffer Adattivo

Uno dei problemi pratici nella generazione di eventi è evitare che l'ultimo evento di una sezione "sbordi" nella successiva. Il safety buffer risolve questo problema:

```python
# Determina la maschera per il calcolo del buffer
mask_for_buffer = layer.get('stato_unico') if is_static_layer else layer.get('stato_finale')

# Estrai la durata armonica massima
max_harmonic_dur_unscaled = mask_for_buffer['durata_armonica']['range'][1]
max_harmonic_dur_scaled = max_harmonic_dur_unscaled * time_ratio

# Trova il moltiplicatore massimo
moltiplicatore_mask = mask_for_buffer.get('moltiplicatore_durata')
if moltiplicatore_mask and 'choices' in moltiplicatore_mask:
    max_duration_multiplier = max(moltiplicatore_mask['choices'])
else:
    max_duration_multiplier = 1.6  # Default del sistema

# Calcola il buffer
safety_buffer = max_harmonic_dur_scaled * max_duration_multiplier
generation_duration = layer_duration_abs - safety_buffer
```

Il calcolo considera il caso peggiore: la durata armonica massima moltiplicata per il fattore di sovrapposizione massimo. Sottraendo questo valore dalla durata disponibile, si garantisce che anche l'evento più lungo non ecceda i limiti temporali.

### Generazione degli Eventi per Cluster

Il sistema supporta la generazione di "cluster" di eventi per ogni punto temporale:

```python
for onset_relative_to_layer in cluster_onsets_relative_to_layer:
    absolute_onset_time = layer_start_time_abs + onset_relative_to_layer
    
    # Determina la maschera attuale
    if is_static_layer:
        center_mask = layer['stato_unico']
    else:
        progress = onset_relative_to_layer / generation_duration
        center_mask = self._interpolate_mask(start_mask, end_mask, progress)
    
    # Estrai densità del cluster
    dens_range = center_mask.get('densita_cluster', {'range': [1,1]})['range']
    num_events_in_cluster = random.randint(int(dens_range[0]), int(dens_range[1]))
```

La densità del cluster può evolvere nel tempo per layer dinamici, permettendo texture che si addensano o si rarefanno. Ogni evento nel cluster riceve la stessa maschera di base ma genera parametri indipendenti, creando micro-variazioni attorno a un'identità comune.

### Gestione del Jitter Temporale

Per evitare una griglia temporale troppo rigida, il sistema introduce perturbazioni controllate:

```python
jitter_scale = params.get('onset_jitter', 0.05)
jitter = np.random.normal(loc=0.0, scale=jitter_scale)
final_event_time = absolute_onset_time + jitter
```

Il jitter segue una distribuzione normale centrata su zero, meaning che gli eventi tendono a rimanere vicini alla loro posizione nominale ma con occasionali deviazioni. Il parametro `onset_jitter` controlla l'ampiezza di queste deviazioni: valori piccoli (0.01) creano micro-deviazioni quasi impercettibili, valori più grandi (0.1) possono creare sensazioni di "scivolamento" temporale.

### Il Sistema di Leeway

Il parametro `leeway_fine_layer` aggiunge flessibilità alla fine del layer:

```python
layer_leeway = layer.get('leeway_fine_layer', 0.0)
leeway_bonus = random.uniform(0.0, layer_leeway) if layer_leeway > 0 else 0.0
desired_total_duration = base_total_duration + leeway_bonus

# Calcola durata massima disponibile
layer_end_time_with_leeway = layer_start_time_abs + layer_duration_abs + layer_leeway
available_duration = max(0.0, layer_end_time_with_leeway - final_event_time)

# Durata finale come minimo tra desiderata e disponibile
params['durata_totale'] = min(desired_total_duration, available_duration)
```

Questo meccanismo permette agli ultimi eventi di un layer di "sfumare" oltre il lifespan nominale, creando code più naturali. È particolarmente utile per evitare troncamenti bruschi in texture continue.

## Sistema Multi-Documento e Parti

Gamma supporta composizioni strutturate su multiple scale temporali attraverso un sistema gerarchico che va dal documento YAML alle singole note.

### Architettura Multi-Documento

Il supporto per documenti YAML multipli permette di organizzare composizioni lunghe in parti gestibili:

```python
def load_all_compositions_from_yaml(file_path):
    with open(file_path, 'r') as f:
        docs = list(yaml.safe_load_all(f))
    
    for i, composition in enumerate(docs):
        if composition is None: 
            continue
        
        if not isinstance(composition, list):
            print(f"ERRORE nel documento #{i+1}: deve contenere una lista di sezioni.")
            sys.exit(1)
        composizioni.append(composition)
```

Ogni documento YAML può rappresentare una parte o movimento della composizione. La separazione con `---` nel file YAML diventa una separazione logica nel processing, permettendo di lavorare su parti individuali mantenendo la coerenza del tutto.

### Gestione degli Offset tra Sezioni

Il sistema permette controllo preciso sulla relazione temporale tra sezioni:

```python
for sec_idx, section in enumerate(part_structure):
    # Calcola il tempo di inizio usando il contatore globale
    offset = section.get('offset_inizio', 0.0)
    absolute_section_onset = max(0.0, end_time_of_last_section + offset)
    
    # Calcola durata e punto finale
    scaled_duration = section.get('durata', 0) * section.get('ratio_temporale', 1.0)
    section_end_time = absolute_section_onset + scaled_duration
    
    # Aggiorna il contatore per la prossima iterazione
    end_time_of_last_section = section_end_time
```

Gli offset possono essere:
- **Positivi**: Creano pause tra sezioni
- **Zero**: Sezioni contigue senza pause
- **Negativi**: Permettono sovrapposizioni controllate

Questa flessibilità permette di creare forme musicali complesse dove sezioni possono dissolversi l'una nell'altra o essere separate da silenzi strutturali.

### Assemblaggio Gerarchico

L'assemblaggio finale segue la gerarchia strutturale:

```python
# Layer -> Sezioni
section_assembly_jobs.append({
    'name': section_name_base,
    'csd_path': dirs['sections_csd'] / f"{section_name_base}_assembler.csd",
    'output_wav': section_wav_path,
    'input_layers': layer_files_for_this_section
})

# Sezioni -> Composizione finale
final_assembly_parts.append({
    'wav_path': section_wav_path,
    'onset': absolute_section_onset
})
```

Questo approccio bottom-up permette parallelizzazione efficiente: mentre alcune sezioni sono ancora in rendering, altre possono già essere assemblate. Il sistema genera automaticamente file CSD di assemblaggio che utilizzano l'opcode `diskin2` per combinare i file WAV con timing preciso.

### Gestione dei Silenzi Strutturali

Quando una sezione non contiene layer attivi, il sistema genera file di silenzio:

```python
if not layers_in_section:
    if scaled_duration > 0:
        generate_silent_wav(section_wav_path, scaled_duration)
    continue
```

Questo approccio mantiene la coerenza strutturale: ogni sezione produce sempre un file WAV, semplificando l'assemblaggio finale e permettendo silenzi strutturali intenzionali.

Il sistema di controllo temporale e processamento layer di Gamma dimostra come la complessità della musica generativa possa essere gestita attraverso astrazione e modularizzazione. Dalla distribuzione degli onset alla gestione di composizioni multi-parte, ogni livello del sistema bilancia flessibilità espressiva con determinismo computazionale, permettendo al compositore di lavorare alla scala temporale più appropriata per le proprie necessità creative.
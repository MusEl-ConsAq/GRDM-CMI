# VISUALIZZAZIONE E ANALISI

La visualizzazione in un sistema compositivo generativo non è un accessorio decorativo ma uno strumento epistemologico essenziale. Quando i processi algoritmici generano migliaia di eventi con parametri multidimensionali, la rappresentazione visiva diventa l'unico mezzo pratico per comprendere le strutture emergenti e verificare la corrispondenza tra intenzione compositiva e risultato generato.

## Sistema di Plotting Multi-Pagina

La classe `CompositionDebugger` implementa un sistema di visualizzazione che bilancia completezza informativa con leggibilità, affrontando la sfida di rappresentare simultaneamente molteplici dimensioni parametriche su scale temporali estese.

### Architettura del Piano Roll Esteso

Il metodo principale `plot_piano_roll()` genera visualizzazioni che vanno oltre il tradizionale piano roll MIDI, integrando informazioni su dinamiche, maschere di tendenza e struttura formale:

```python
def plot_piano_roll(self, events, all_onsets, composition_name, composition_structure, 
                   composer, onsets_by_section, title=None, partitura_mode=False, 
                   page_duration_s=60):
    
    with PdfPages(plot_filename) as pdf:
        num_pages = int(np.ceil(total_duration / page_duration_s)) if partitura_mode else 1
        
        for i in range(num_pages):
            fig, (ax_pitch, ax_dyn_linear, ax_dyn_prob) = plt.subplots(
                nrows=3, ncols=1, 
                figsize=(A3_LANDSCAPE_WIDTH_INCHES, A3_LANDSCAPE_HEIGHT_INCHES),
                sharex=True, gridspec_kw={'height_ratios': [3, 1, 1]}
            )
```

La scelta di utilizzare formato A3 landscape riflette la necessità di visualizzare dettagli fini su composizioni di lunga durata. La divisione in tre assi con proporzioni 3:1:1 privilegia la visualizzazione delle altezze mantenendo spazio sufficiente per le dinamiche.

### Rappresentazione degli Eventi Sonori

Gli eventi sono visualizzati come rettangoli colorati dove multiple dimensioni sono mappate a proprietà visive:

```python
for item in plot_data_list:
    if item['start'] < page_end_time and (item['start'] + item['duration']) > page_start_time:
        ax_pitch.add_patch(
            plt.Rectangle(
                (item['start'], item['pitch'] - 0.04), 
                item['duration'], 
                0.08, 
                color=plt.cm.viridis(item['amp_norm']), 
                alpha=0.6, 
                zorder=3
            )
        )
```

La mappatura utilizza:
- **Posizione X**: Tempo di inizio
- **Larghezza**: Durata
- **Posizione Y**: Altezza (ottava.registro)
- **Colore**: Dinamica (attraverso colormap viridis)
- **Trasparenza**: Permette visualizzazione di sovrapposizioni

L'altezza fissa dei rettangoli (0.08) bilancia visibilità e densità informativa.

### Visualizzazione delle Maschere di Tendenza

Le maschere di tendenza sono rappresentate come aree ombreggiate che mostrano l'evoluzione dei parametri:

```python
def _plot_parameter_envelope(self, ax, times, data, color, label_key, label_text):
    if not data:
        return
        
    label = label_text if label_key not in self._labels_added else ""
    ax.fill_between(times, data['lower'], data['upper'], 
                    color=color, alpha=0.2, zorder=1, label=label)
    self._labels_added.add(label_key)
```

L'uso di `fill_between` crea un'area che rappresenta visivamente il range di valori possibili in ogni momento. La trasparenza (alpha=0.2) permette la sovrapposizione di multiple maschere mantenendo la leggibilità.

### Gestione delle Dinamiche: Lineare vs Probabilistica

Il sistema distingue tra due modalità di visualizzazione delle dinamiche, riflettendo la differenza tra interpolazione deterministica e evoluzione probabilistica:

```python
def _plot_dynamics(self, ax_linear, ax_prob, times, dynamics_data, layer_color, layer_name):
    if dynamics_data['is_probabilistic']:
        # Stackplot per probabilità
        if dynamics_data['prob_weights']:
            weights_per_dynamic = np.array(dynamics_data['prob_weights']).T
            ax_prob.stackplot(times, weights_per_dynamic, 
                             labels=stackplot_labels, colors=stackplot_colors,
                             alpha=0.6, zorder=2)
    else:
        # Linea per interpolazione
        if dynamics_data['line_trend']:
            ax_linear.plot(valid_times, valid_dynamics, color=layer_color, 
                          linewidth=2.5, label=f"Dinamica: {layer_name}", 
                          zorder=3, alpha=0.8)
```

Lo stackplot mostra l'evoluzione delle probabilità per ogni dinamica, mentre il grafico lineare traccia l'interpolazione continua dell'indice dinamico. Questa doppia rappresentazione cattura la differenza fondamentale tra i due approcci compositivi.

### Modalità Partitura per Composizioni Lunghe

Per composizioni che eccedono una singola pagina, il sistema genera PDF multi-pagina:

```python
if partitura_mode:
    page_start_time = i * page_duration_s
    page_end_time = (i + 1) * page_duration_s
    ax_pitch.set_xlim(page_start_time, page_end_time)
    
    page_title = f"{title} (Pagina {i+1}/{num_pages})"
    fig.suptitle(page_title, fontsize=14)
```

Ogni pagina mantiene la stessa struttura visiva ma mostra una finestra temporale diversa. I marcatori di sezione e le etichette sono gestiti per evitare duplicazioni tra pagine.

## Cache e Aggiornamento Incrementale

Il sistema di cache rappresenta una soluzione al problema dei lunghi tempi di calcolo per la visualizzazione di composizioni complesse.

### Serializzazione e Deserializzazione

I dati di visualizzazione vengono salvati in formato JSON:

```python
def _sanitize_data_for_json(data):
    if isinstance(data, dict):
        return {k: _sanitize_data_for_json(v) for k, v in data.items()}
    elif isinstance(data, list):
        return [_sanitize_data_for_json(v) for v in data]
    elif isinstance(data, Path):
        return str(data)
    elif isinstance(data, np.ndarray):
        return data.tolist()
    elif isinstance(data, (np.float64, np.int64)):
        return float(data)
    
    return data
```

La sanitizzazione gestisce tipi Python/NumPy non serializzabili nativamente in JSON. La conversione di Path objects in stringhe e di array NumPy in liste mantiene la portabilità dei dati.

### Meccanismo di Merge Selettivo

L'aggiornamento incrementale identifica e sostituisce solo i dati modificati:

```python
if fresh_data.get('render_jobs_info'):
    fresh_job_keys = set(
        (job['section']['nome_sezione'], job['layer_idx']) 
        for job in fresh_jobs_info
    )
    
    # Rimuovi vecchi dati per layer aggiornati
    final_data['events'] = [
        e for e in final_data['events']
        if (e['params']['section_name'], e['params'].get('layer_idx_ref')) 
           not in fresh_job_keys
    ]
```

L'identificazione attraverso tupla (`section_name`, `layer_idx`) garantisce unicità. La rimozione dei vecchi dati prima dell'inserimento dei nuovi previene duplicazioni.

### Benefici del Sistema di Cache

La cache trasforma il workflow compositivo in diversi modi:

1. **Iterazione Rapida**: Modificare un singolo layer non richiede ricalcolo dell'intera visualizzazione
2. **Persistenza tra Sessioni**: I dati sopravvivono al riavvio del sistema
3. **Debugging Storico**: È possibile confrontare visualizzazioni di versioni diverse

### Gestione della Coerenza

Il sistema mantiene coerenza tra cache e stato attuale attraverso:

```python
# Allineamento della struttura onsets_by_section
onsets_map = {}
for job in final_data.get('render_jobs_info', []):
    key = (job['section']['nome_sezione'], job['layer_idx'])
    onsets_map[key] = job.get('adjusted_onsets', [])

for section_yaml in composition_structure:
    section_data = {'section_name': section_yaml['nome_sezione'], 'layers': []}
    for layer_idx_yaml, layer_yaml in enumerate(layers_in_yaml):
        key = (section_yaml['nome_sezione'], layer_idx_yaml)
        onsets_for_this_layer = onsets_map.get(key, [])
        section_data['layers'].append({
            'layer_name': layer_yaml.get('nome_layer', f'Layer {layer_idx_yaml+1}'),
            'onsets': onsets_for_this_layer
        })
```

Questo allineamento garantisce che la struttura dei dati di visualizzazione corrisponda sempre alla struttura YAML corrente, anche quando layer sono stati aggiunti o rimossi.

Il sistema di visualizzazione e analisi di Gamma trasforma dati numerici astratti in rappresentazioni visive che rivelano la struttura e l'evoluzione della composizione. L'integrazione di multiple dimensioni parametriche in una singola visualizzazione, combinata con il sistema di cache incrementale, crea uno strumento che non solo documenta il risultato compositivo ma diventa parte integrante del processo creativo stesso, permettendo al compositore di "vedere" la musica emergere dai processi generativi e di intervenire con precisione chirurgica dove necessario.
# SCOLPIRE IL TEMPO: MODELLI TEMPORALI E MICRO-RITMICA

La classe `TimeScheduler` incapsula la logica per la distribuzione temporale, offrendo un toolkit di modelli che corrispondono a gesti musicali archetipici. La scelta di isolare questa funzionalità in una classe dedicata sottolinea come il tempo musicale non sia un semplice parametro, ma una dimensione fondamentale che richiede un trattamento specializzato.

Il metodo `generate_onsets` è il cuore di questa classe. La sua architettura è elegante e flessibile: parte sempre da una progressione lineare di base, che viene poi "deformata" o "rimappata" secondo il modello scelto nel file YAML.

```python
def generate_onsets(self, model, duration, num_events):
    base_progress = np.linspace(0, 1, num_events, endpoint=False)
    final_progress = np.zeros_like(base_progress)
    model_type = model.get('type', 'linear')
    return final_progress * duration
```

Questa architettura a due fasi (generazione di una progressione normalizzata e successiva trasformazione) permette di definire gesti temporali indipendentemente dalla durata effettiva, rendendoli riutilizzabili e scalabili.

### I Modelli Archetipici

**Lineare (`type: linear`)**

Il modello di default, che distribuisce gli eventi in modo equidistante. Sebbene semplice, è fondamentale per creare pulsazioni regolari, ostinati o griglie ritmiche stabili su cui altri layer possono costruire complessità.

**Ritardando (`type: ritardando`)** 
```python
elif model_type == 'ritardando':
    shape = model.get('shape', 2.0)
    final_progress = base_progress ** shape
```
Applicando una funzione di potenza con esponente maggiore di 1, la curva di progressione si "piega", concentrando gli eventi all'inizio e diradandoli verso la fine. Il parametro `shape` controlla l'intensità del gesto: un valore più alto crea un ritardando pronunciato.

**Accelerando (`type: accelerando`)**
```python
elif model_type == 'accelerando':
    shape = model.get('shape', 2.0)
    final_progress = 1 - (1 - base_progress) ** shape
```
La formula inverte la progressione, applica la potenza e la inverte nuovamente.
**Stocastico (`type: stochastic`)**

```python
elif model_type == 'stochastic':
    final_progress = np.sort(np.random.rand(num_events))
```
Il metodo genera un set di istanti casuali e poi li ordina. Questo garantisce che, pur essendo irregolari, gli eventi mantengano una progressione temporale in avanti. È un modello efficace per rompere la rigidità della griglia metrica.

### Il Modello `breakpoint`: Curve Temporali su Misura

Il modello più potente e flessibile è `breakpoint`. Permette al compositore di "disegnare" una curva di distribuzione temporale definendo una serie di punti di controllo. Ogni segmento tra due punti può avere la propria curvatura, consentendo la creazione di profili complessi.

Consideriamo un esempio:
```yaml
timing_model:
  type: breakpoint
  points:
    - [0.0, 0.0]        # Inizio
    - [0.3, 0.7, 0.5]   # Il 70% degli eventi avviene nel primo 30% del tempo (curva concava, ease-out)
    - [1.0, 1.0, 3.0]   # Il restante 30% degli eventi si distribuisce nel 70% del tempo rimanente (curva convessa, ease-in)
```
Questo YAML descrive un gesto di "esplosione e diradamento": un'alta densità di eventi all'inizio, seguita da una lunga coda rarefatta.

L'implementazione gestisce questa complessità in modo modulare:
```python
time_in_segment = (segment_times - t_start) / (t_end - t_start)
shaped_time = time_in_segment ** shape
interpolated_values = v_start + (v_end - v_start) * shaped_time
final_progress[segment_mask] = interpolated_values
```
La logica chiave è la normalizzazione del tempo *all'interno di ogni segmento*. Questo permette di applicare una curva di `shape` locale senza che questa influenzi gli altri segmenti, rendendo il sistema potente e intuitivo.

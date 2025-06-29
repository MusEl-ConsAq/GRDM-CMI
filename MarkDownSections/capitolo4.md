# Dall'Intenzione alla Nota: Maschere di Tendenza e Generazione Parametrica 

La generazione parametrica è il motore alchemico di Gamma, il processo centrale attraverso cui le intenzioni del compositore, espresse come  maschere di tendenza  nel file YAML, vengono trasformate in valori numerici concreti per ogni singolo evento sonoro. Se l'orchestra Csound è il corpo esecutore, i metodi analizzati in questo capitolo rappresentano il cervello decisionale che lo pilota. Analizzeremo come il sistema traduce l'astrazione in suono, con particolare attenzione alle tecniche di generazione, interpolazione e gestione della coerenza strutturale.

## Il Motore Generativo: `_generate_params_from_mask()` 

Il metodo `_generate_params_from_mask()` è il punto di convergenza tra l'astrazione compositiva e la concretezza numerica. Implementa la logica che trasforma una singola maschera di tendenza nei parametri specifici per un evento sonoro, gestendo una varietà di strategie generative per rispondere a diverse necessità espressive.

###  Architettura e Gestione Specializzata 

Il processo non è monolitico. Il metodo prima isola un insieme di chiavi che richiedono una gestione specializzata, poiché non rappresentano parametri diretti o necessitano di logiche di trasformazione complesse.

```python
SKIPPED_KEYS = {
    'choices', 'weights', 'distribution',  # Metadati per la generazione
    'dynamic_index', 'dinamica',           # Logica di dinamica complessa
    'nonlinear_mode', 'senso_movimento',   # Parametri di controllo per Csound
    'inviluppo_attacco', 'tipo_ritmi',      # Richiedono traduzione o generazione complessa
    'densita_cluster'                      
}
```
Questa separazione permette a un loop generico di gestire i parametri puramente numerici, mentre logiche dedicate si occupano di tradurre concetti come `'dinamica': 'f'` nell'indice numerico richiesto da Csound o di generare intere sequenze ritmiche da una categoria come `'medi'`.

###  Le Modalità di Generazione: Un Toolkit Espressivo 

Il cuore del metodo itera sui parametri, applicando la modalità di generazione più appropriata in base alla struttura della maschera.

```python
for key, p_mask in mask.items():
    if key in SKIPPED_KEYS: continue
    
    # 1. Distribuzione Normale (Gaussiana)
    if 'mean' in p_mask and 'std' in p_mask:
        val = np.random.normal(loc=p_mask['mean'], scale=p_mask['std'])
    
    # 2. Distribuzione Uniforme (Range)
    elif 'range' in p_mask:
        min_val, max_val = p_mask['range']
        val = random.randint(min_val, max_val) if isinstance(min_val, int) else random.uniform(min_val, max_val)
    
    # 3. Scelta Discreta Pesata
    elif 'choices' in p_mask:
        val = random.choices(p_mask['choices'], weights=p_mask.get('weights'), k=1)[0]
    
    # 4. Valore Fisso
    elif 'value' in p_mask:
        val = p_mask['value']
```

Questo toolkit di quattro modalità offre al compositore un controllo granulare sul grado di determinismo e casualità:

1.   Distribuzione Normale : Ideale per creare una "massa" sonora attorno a un centro tonale o timbrico. Un'ottava definita come `{mean: 5, std: 0.5}` tenderà a rimanere nell'ottava 5, ma con occasionali e naturali "fughe" verso le ottave vicine. La deviazione standard diventa un parametro espressivo che controlla la "disciplina" del materiale.
2.   Range Uniforme : Utile quando tutti i valori in un intervallo sono ugualmente desiderabili. Un `range: [1, 10]` per il registro crea salti discreti, mentre un `range: [1.0, 10.0]` abilita micro-intervalli, mostrando come una scelta tecnica influenzi direttamente il risultato sonoro.
3.   Scelta Pesata : Permette di definire il "colore" statistico di una sezione. Una dinamica specificata come `{choices: ['p', 'mf', 'f'], weights: [0.6, 0.3, 0.1]}` assicura una predominanza di eventi piano, pur mantenendo la varietà.
4.   Valore Fisso : Garantisce il determinismo assoluto, essenziale per parametri strutturali come il senso di movimento spaziale.

###  La Logica Gerarchica dei Ritmi 

La generazione dei ritmi è un esempio emblematico di come il sistema supporti molteplici livelli di astrazione, dal controllo totale alla delega generativa.

```python
if 'explicit_values' in rhythm_mask:
    # Modalità 1: Controllo totale con una lista esplicita
    params['ritmi'] = rhythm_mask['explicit_values']
elif 'choices' in rhythm_mask:
    choice = random.choices(rhythm_mask['choices'], ...)[0]
    if isinstance(choice, list):
        # Modalità 2: Scelta tra pattern pre-composti
        params['ritmi'] = choice
    else:
        # Modalità 3: Astrazione massima tramite categorie ('piccoli', 'medi'...)
        params['ritmi'] = self._generate_rhythm_pattern(choice)
```
Questa architettura permette al compositore di scegliere il livello di dettaglio più consono: specificare un pattern esatto, scegliere da una libreria di pattern, o semplicemente indicare una "qualità" ritmica desiderata.

##  3.2 L'Evoluzione nel Tempo: Interpolazione delle Maschere 

Se la generazione da una singola maschera crea eventi statici, l'interpolazione tra due maschere (`stato_iniziale` e `stato_finale`) dà vita a processi dinamici e trasformativi. Il metodo `_interpolate_mask()` implementa questa logica con una particolare attenzione alla robustezza e alla coerenza musicale.

###  Gestione delle Transizioni Asimmetriche 

Un problema chiave nell'interpolazione è come gestire parametri che compaiono solo nello stato finale. Gamma adotta una strategia di "riempimento" che ne aumenta la flessibilità.

```python
all_keys = set(start_mask.keys()) | set(end_mask.keys())
for key in all_keys:
    s_mask = start_mask.get(key)
    e_mask = end_mask.get(key)
    
    if s_mask is None: s_mask = e_mask
    if e_mask is None: e_mask = s_mask
```
Se un parametro è definito solo alla fine, il sistema assume che fosse presente fin dall'inizio con lo stesso valore finale. Questo permette di "introdurre" un nuovo processo (es. un glissando) senza doverne specificare un valore nullo all'inizio, semplificando la scrittura delle partiture YAML.

###  Strategie di Interpolazione Differenziate 

L'interpolazione si adatta al tipo di parametro, producendo transizioni musicalmente significative:

-    Parametri Numerici (Range, Mean/Std) : I loro valori vengono interpolati linearmente. Questo permette di creare effetti come un "restringimento" del campo sonoro (interpolando verso un range più piccolo) o una "focalizzazione" (interpolando verso una deviazione standard minore).

-    Scelte Discrete (Choices) : Il sistema tenta un  cross-fade probabilistico . Se le scelte sono le stesse, i loro pesi (`weights`) vengono interpolati. Questo rea una transizione graduale nella probabilità di occorrenza, ad esempio passando da una predominanza di dinamiche piano a una di forte. Se le scelte sono diverse, il sistema effettua una transizione a scalino a metà del percorso.

È importante notare che questo processo di interpolazione non solo guida la generazione degli eventi sonori, ma fornisce anche i dati per la visualizzazione grafica. Le "buste di tendenza" visibili nel PDF generato da `CompositionDebugger` sono la rappresentazione visiva diretta dei valori interpolati in ogni punto del tempo. Questo crea una coerenza totale tra ciò che il compositore specifica, ciò che il sistema visualizza e ciò che l'orchestra Csound suona.

##  3.3 Un Caso di Studio: Il Sistema Gerarchico del Glissando 

Il glissando in Gamma non è una semplice transizione di frequenza, ma un sistema gerarchico che illustra l'approccio progettuale del compositore: fornire opzioni potenti con priorità chiare.

1.  Modalità Offset (Priorità Massima): Specifica un intervallo di glissando relativo alla nota di partenza (es. `offset_ottava: 2`). È ideale per creare pattern di movimento che mantengono la loro coerenza intervallare a diverse altezze.

2.   Modalità Assoluta (Priorità Media) : Specifica una destinazione fissa (es. `ottava_arrivo: 8`). È utile per creare convergenze armoniche, dove più voci, partendo da punti diversi, si dirigono verso un'unica regione tonale.

3.   Default (Nessun Glissando) : In assenza di specifiche, la frequenza rimane statica.

Questa gerarchia viene risolta in `_generate_params_from_mask()`, che calcola i parametri `ottava_arrivo` e `registro_arrivo` finali. Questi vengono poi passati a Csound, dove la funzione `calcFrequenza` è chiamata due volte, una per la frequenza di partenza e una per quella di arrivo, garantendo che entrambe rispettino la logica dell'intonazione pitagorica del sistema.

In conclusione, l'architettura di generazione parametrica di Gamma è stratificata e flessibile. Separa nettamente la specifica compositiva (l'intenzione nel YAML) dall'implementazione tecnica (la logica di generazione e interpolazione in Python). Questo permette al compositore di operare al livello di astrazione più consono, che sia definire un pattern nota per nota o semplicemente tracciare una "tendenza" evolutiva, lasciando al sistema il compito di tradurre questa intenzione in una texture sonora ricca e coerente.
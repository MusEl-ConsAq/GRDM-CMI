# CONFIGURAZIONE E PARAMETRIZZAZIONE

Il sistema Gamma implementa una separazione netta tra logica di sintesi e configurazione dei parametri, permettendo al compositore di modificare profondamente il comportamento del sistema senza toccare il codice Csound. Questo approccio modulare facilita la sperimentazione e l'estensione del sistema.

## Il File tables.yaml

Il file `tables.yaml` rappresenta il cuore configurabile del sistema, definendo tutte le tabelle di forma d'onda e inviluppo utilizzate nella sintesi. La sua struttura gerarchica separa chiaramente gli inviluppi per eventi singoli da quelli per sezioni intere.

Ogni tabella è definita attraverso quattro parametri fondamentali:

```yaml
nome_simbolico:
  number: [numero della f-table in Csound]
  size: [dimensione in campioni]
  gen_routine: [numero della GEN routine]
  parameters: [lista dei parametri per la GEN]
```

Vediamo un esempio concreto:

```yaml
event_envelopes:
  lineare:
    number: 2
    size: 4096
    gen_routine: 6
    parameters: [0.001, 2048, 0.5, 2048, 1]  # Linea retta da 0 a 1
```

Questa definizione genera in Csound:
```csound
f 2 0 4096 6 0.001 2048 0.5 2048 1
```

Il sistema distingue due categorie di inviluppi con funzioni distinte:

`Event Envelopes` Applicati ai singoli eventi sonori:

```yaml
event_envelopes:
  impulsivo:
    number: 5
    size: 4096
    gen_routine: 5
    parameters: [0.001, 512, 1, 3584, 0.0001]
    
  lento:
    number: 6
    size: 4096
    gen_routine: 7
    parameters: [0, 3072, 1, 1024, 0]
    
  sostenuto:
    number: 7
    size: 4096
    gen_routine: 7
    parameters: [0, 512, 1, 3072, 1, 512, 0]
```

L'inviluppo `impulsivo` utilizza GEN 5 (segmenti esponenziali) per creare un attacco rapidissimo (512 campioni su 4096, circa 1/8 della durata) seguito da un decadimento esponenziale. Il valore finale di 0.0001 invece di 0 evita discontinuità nell'interpolazione esponenziale.

L'inviluppo `lento` con GEN 7 crea un attacco graduale per 3/4 della durata, ideale per tessiture ambient o crescendi graduali.

`Section Envelopes` Modulano interi gruppi di eventi:

```yaml
section_envelopes:
  crescendo_diminuendo:
    number: 24
    size: 4096
    gen_routine: 7
    parameters: [0, 2048, 1, 2048, 0]
    
  impulso:
    number: 25
    size: 4096
    gen_routine: 6
    parameters: [1, 4096, 0.001]
```

Gli inviluppi di sezione operano su una scala temporale maggiore. Il numero di tabella parte da 20 per convenzione, distinguendoli chiaramente dagli inviluppi evento nel codice Csound:

```csound
if i_ifn_section_env > 20 && i_section_duration > 0 then
    ; Applica inviluppo di sezione
endif
```

## Sistema di Macro e Costanti Globali

Il template CSD di Gamma definisce un sistema di macro che parametrizza l'intero spazio frequenziale:

```csound
#define FONDAMENTALE #32#
#define OTTAVE #10#
#define INTERVALLI #200#
#define REGISTRI #50#
```

Le macro `OTTAVE`, `INTERVALLI` e `REGISTRI` definiscono la risoluzione del sistema di intonazione:

- `OTTAVE` (10): Copre l'intero range udibile da 32 Hz a ~32 kHz
- `INTERVALLI` (200): Numero di divisioni per ottava nel sistema pitagorico
- `REGISTRI` (50): Suddivisioni macro all'interno di ogni ottava

La relazione tra questi parametri determina la granularità frequenziale:
```
Totale frequenze = OTTAVE * INTERVALLI = 2000
Risoluzione per registro = INTERVALLI / REGISTRI = 4 intervalli
```

Il file `tables.yaml` viene letto dal generatore Python che:
1. Carica le configurazioni all'inizializzazione
2. Genera automaticamente gli f-statement nel CSD
3. Mantiene mappe nome→numero per riferimenti simbolici

Questo permette di riferirsi agli inviluppi per nome nel YAML compositivo:
```yaml
inviluppo_attacco: { value: 'impulsivo' }
```

Invece di numeri magici:
```yaml
inviluppo_attacco: { value: 5 }  # Meno leggibile e manutenibile
```

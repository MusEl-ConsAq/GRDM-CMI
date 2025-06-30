# INTRODUZIONE

Il presente lavoro documenta lo sviluppo di "Gamma", un sistema compositivo algoritmico che rappresenta una tappa fondamentale nel più ampio progetto del ciclo "Delta". Questa tesina nasce dall'esigenza di formalizzare e analizzare un percorso compositivo che, partito con ambizioni di complessità adattiva, ha rivelato la necessità di un passaggio intermedio attraverso un sistema di composizione algoritmica.

## Il Ciclo Delta e la Genesi di Gamma

Il ciclo compositivo "Delta" nasce dalla volontà di studiare come modellare un sistema musicale complesso adattivo. Concepito come sistema chiuso e acusmatico, Delta rappresenta un passo preliminare e necessario prima di approcciare lo studio e la realizzazione di ecosistemi performativi aperti, come quelli esplorati da compositori come Agostino Di Scipio. La scelta di lavorare inizialmente con un sistema chiuso non è limitativa, ma strategica: permette di concentrarsi sulla comprensione e modellazione delle dinamiche interne senza le variabili aggiuntive dell'interazione in tempo reale con l'ambiente o con i performer.

La sfida principale che ha portato alla nascita di Gamma non risiedeva nel mantenere un controllo compositivo bensì nello studio in vitro ovvero in una dimensione estremamente controllate di uno degli agenti che stavo costruendo per il sistema complesso Delta. Ho ritenuto utile scrivere prima un brano preparatorio utilizzando esclusivamente uno dei sistemi messi in relazione in Delta, ovvero lo strumento 'Comportamento' che in Gamma prende il nome di 'Voce'. 'Voce' ha all'interno un comportamento caotico derivato dall'utilizzo di una mappa logistica in feedback che utilizzo per ricavarmi i ritmi futuri (vedremo in seguito nella tesina e discuteremo i vari casi).

Gamma è un laboratorio dove sperimentare, comprendere e affinare gli strumenti compositivi prima di lanciarsi nella complessità delle dinamiche caotiche e adattive.

La scelta del nome "Gamma" riflette precisamente questa funzione: rappresenta una gamma di possibilità esperibili da Delta. Se Delta è la foce dove tutti i flussi convergono attraverso reti di relazioni, Gamma è la sorgente - il luogo dove questi flussi nascono chiari e distinguibili, dove è possibile osservare e comprendere ogni singolo rivolo prima che si mescoli con gli altri.

## Dalle Note alle Nuvole: Un'Eredità Compositiva

Gamma si inserisce nella tradizione della composizione sistematica ispirandosi liberamente ai metodi di lavoro sviluppati da Iannis Xenakis e Barry Truax. L'approccio delle "maschere di tendenza" che caratterizza il sistema non è un'interpretazione personale e un'implementazione specifica di tecniche già consolidate nella letteratura della computer music. Xenakis aveva esplorato l'uso di distribuzioni probabilistiche per la generazione di masse sonore. Truax sviluppò l'approccio delle maschere di tendenza per necessità pratiche legate alla sintesi granulare: quando si lavora con tecniche che richiedono la generazione di milioni di parametri per controllare nuvole di grani sonori, diventa impossibile specificare ogni singolo valore. Le maschere di tendenza emergono quindi come soluzione naturale per gestire questa complessità, permettendo di definire comportamenti statistici globali piuttosto che valori individuali.

Ciò che Gamma apporta a questa tradizione è una sistematizzazione particolare di questi concetti. Il sistema implementa quattro modalità distinte di generazione parametrica (range, choices, distribuzione normale, valore fisso), organizzate in una gerarchia compositiva chiara (composizione → sezioni → layer → eventi). Questa strutturazione permette di gestire la complessità mantenendo un controllo compositivo significativo, preparando il terreno per l'evoluzione verso il sistema adattivo previsto.

L'uso delle maschere di tendenza in Gamma permette al compositore di lavorare su diversi livelli di astrazione simultaneamente. A livello micro, si possono definire distribuzioni precise per singoli parametri; a livello macro, si possono creare evoluzioni graduali attraverso l'interpolazione tra stati.

## Struttura e Obiettivi della Tesina

Il presente lavoro si propone di documentare e analizzare il sistema Gamma sotto molteplici prospettive, fornendo sia una comprensione teorica dei principi sottostanti sia una guida pratica all'implementazione e all'uso del sistema.

Gli obiettivi principali sono:

1. **Documentare l'architettura del sistema**: Fornire una descrizione dettagliata e sistematica di tutti i componenti software che costituiscono Gamma, dalle strutture dati Python agli strumenti Csound, dalla sintassi YAML al sistema di visualizzazione.

4. **Valutare criticamente il sistema**: Identificare punti di forza e limitazioni di Gamma, sia dal punto di vista tecnico che estetico, fornendo spunti per sviluppi futuri.

5. **Preparare il terreno per Delta**: Comprendere come l'esperienza di Gamma informi e prepari lo sviluppo del sistema adattivo completo previsto per Delta.

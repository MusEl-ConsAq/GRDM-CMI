# CAPITOLO 1: L'ORCHESTRA GAMMA - ARCHITETTURA E FLUSSO SONORO (versione rivista)

L'orchestra Csound di Gamma rappresenta il cuore pulsante del sistema di sintesi, dove i parametri astratti generati dal motore Python si trasformano in eventi sonori concreti. Questo capitolo esplora in dettaglio l'architettura degli strumenti principali e il flusso di elaborazione che porta dal numero al suono.

## 1.1 Lo Strumento Voce: Generatore di Comportamenti

Lo strumento `Voce` costituisce il livello più alto della gerarchia di sintesi in Gamma. Non genera direttamente suoni, ma orchestra la creazione di sequenze di eventi sonori secondo logiche compositive complesse. La sua definizione inizia con una ricca parametrizzazione:

```csound
instr Voce
    ; -----------------------------------------------------------------------
    ; 1. INIZIALIZZAZIONE E ACQUISIZIONE PARAMETRI
    ; -----------------------------------------------------------------------
    i_CAttacco       = p2             ; Tempo di attacco del comportamento
    i_Durata         = p3             ; Durata complessiva
    i_RitmiTab       = p4             ; Tabella dei ritmi
    i_DurataArmonica = p5             ; Durata armonica di riferimento
    i_DynamicIndex   = p6         
    i_Ottava         = p7             
    i_Registro       = p8             
    i_ottava_arrivo = p9
    i_registro_arrivo = p10
    i_PosTab         = p11             ; Tabella delle posizioni
    i_IdComp         = p12            ; ID del comportamento
    i_NonlinearMode  = (p13 == 0 ? 3 : p13)
    i_SensoMovimento = (p14 == 0 ? 1 : p14) 
    i_ifnAttacco     = (p15 == 0 ? 10 : p15)
    i_ifn_section_env = p16 
    i_section_start_time = p17
    i_duration_leeway = p19
    i_section_duration = p18 + p19
    i_section_end = i_section_start_time + i_section_duration
    iSafetyBuffer = p20
```

Ogni parametro ha un significato musicale preciso:

- **i_CAttacco** e **i_Durata**: definiscono la finestra temporale in cui il comportamento è attivo
- **i_RitmiTab**: punta a una tabella contenente la sequenza di valori ritmici che determinano sia la temporalità che le frequenze degli eventi
- **i_DurataArmonica**: il valore di riferimento per il calcolo delle durate reali degli eventi
- **i_Ottava** e **i_Registro**: coordinate nello spazio delle altezze di partenza
- **i_ottava_arrivo** e **i_registro_arrivo**: destinazione per eventuali glissandi
- **i_NonlinearMode**: seleziona l'algoritmo di generazione per nuovi ritmi

### Il Loop Generativo Principale

Il cuore dello strumento Voce è un loop while che genera eventi fino al raggiungimento della durata specificata:

```csound
i_EventIdx = 0
i_whileTime = 0

while i_whileTime < i_Durata do
    ; -------- 3.1 GESTIONE RITMI --------
    if i_EventIdx < i_LenRitmiTab then
        i_RitmoCorrente tab_i i_EventIdx, i_TempRitmiTab
        if i_RitmoCorrente == 0 then
            goto generateNewRhythm
        endif
        i_Vecchio_Ritmo = (i_EventIdx == 0) ? 1 : tab_i(i_EventIdx - 1, i_TempRitmiTab)
    else
        generateNewRhythm:
        i_Vecchio_Ritmo tab_i i_EventIdx - 1, i_TempRitmiTab
        i_RitmoCorrente NonlinearFunc i_Vecchio_Ritmo, i_NonlinearMode
        tabw_i i_RitmoCorrente, i_EventIdx, i_TempRitmiTab
    endif
```

Questo codice implementa una logica sofisticata: inizialmente legge i ritmi dalla tabella fornita, ma quando questa si esaurisce, genera nuovi valori usando l'opcode `NonlinearFunc`, creando potenzialmente sequenze infinite che evolvono secondo regole caotiche o deterministiche.

### Calcolo Temporale degli Eventi

Il timing di ogni evento dipende dal ritmo precedente secondo la formula:

```csound
if i_EventIdx == 0 then
    i_EventAttack = i_CAttacco
else
    i_RitmoNormalizzato = 1 / i_Vecchio_Ritmo
    i_PreviousAttack tab_i gi_Index - 1, gi_eve_attacco
    i_EventAttack = i_DurataArmonica * i_RitmoNormalizzato + i_PreviousAttack
endif
```

Questa relazione inversamente proporzionale significa che valori ritmici più alti producono eventi più ravvicinati, creando accelerazioni, mentre valori bassi generano rarefazioni temporali.

### Gestione della Tabella Ritmi Temporanea

Una caratteristica importante è la creazione di una tabella temporanea estesa per i ritmi:

```csound
i_LenRitmiTab = ftlen(i_RitmiTab)
i_TempRitmiTab ftgen 0, 0, i_LenRitmiTab + 10000, -2, 0

; Copia i ritmi iniziali nella tabella temporanea
i_IndexCopy = 0
while i_IndexCopy < i_LenRitmiTab do
    i_ValRitmo tab_i i_IndexCopy, i_RitmiTab
    tabw_i i_ValRitmo, i_IndexCopy, i_TempRitmiTab
    i_IndexCopy += 1
od
```

Questo approccio permette di estendere dinamicamente la sequenza ritmica oltre i valori iniziali senza modificare la tabella originale, mantenendo la purezza dei dati di input mentre si esplora lo spazio generativo.

### Sistema di Scheduling degli Eventi

La creazione effettiva degli eventi sonori avviene attraverso la chiamata a `schedule`:

```csound
schedule "eventoSonoro", i_EventAttack - p2, i_EventDuration, i_DynamicIndex, i_Freq1, i_Pos,
        i_RitmoCorrente, i_Freq2, i_ifnAttacco, gi_Index, i_IdComp, i_SensoMovimento, 
        i_ifn_section_env, i_section_start_time, i_section_duration
```

Notare come `i_EventAttack - p2` converta il tempo assoluto in tempo relativo all'inizio dello strumento Voce, mantenendo la coerenza temporale nella gerarchia degli strumenti.

## 1.2 EventoSonoro: Dal Parametro al Suono

Lo strumento `eventoSonoro` è responsabile della generazione effettiva del suono. Riceve i parametri calcolati da Voce e li trasforma in segnale audio attraverso sintesi e processamento.

### Inizializzazione e Validazione Parametri

```csound
instr eventoSonoro
   i_DynamicIndex = p4
   i_debug=gi_debug
   ifreq1 = limit(p5, 20, sr/2)   
   iwhichZero = abs(p6)    
   iHR = max(1, abs(p7))

   iPeriod = $M_PI * 2 / iHR
   
   iradi = (iwhichZero > 0 ? (iwhichZero - 1) * iPeriod : 0)
   ifreq2 = limit(p8, 20, sr/2)

   ifn_shape = (p9 == 0 ? 2 : p9)
```

I parametri vengono immediatamente validati e limitati per evitare valori che potrebbero causare problemi:
- Le frequenze sono limitate tra 20 Hz e la frequenza di Nyquist
- `iHR` (harmonic ratio) è forzato ad essere almeno 1
- `ifn_shape` ha un default alla tabella 2 se non specificato

### Sistema di Compensazione Isofonica dell'Ampiezza

Una delle caratteristiche più sofisticate di Gamma è l'implementazione di un sistema di calibrazione dell'ampiezza basato sulle curve isofoniche ISO 226:2003. Questo garantisce che la percezione di loudness rimanga costante indipendentemente dalla frequenza.

Il calcolo dell'ampiezza avviene attraverso una catena di UDO specializzati:

```csound
kamp GetIsoAmp_k i_DynamicIndex, ifreq1, ifreq2
```

Questo UDO k-rate gestisce glissandi compensando dinamicamente l'ampiezza. Vediamo l'implementazione:

```csound
opcode GetIsoAmp_k, k, iii
    iDynamicIndex, iFreqStart, iFreqEnd xin
    
    ; 1. Calcola le ampiezze isofoniche per i punti di inizio e fine a i-rate
    iAmpStart       GetIsoAmp       iFreqStart, iDynamicIndex
    iAmpEnd         GetIsoAmp       iFreqEnd, iDynamicIndex

    if iAmpStart > iAmpEnd then
        kf expseg  1, p3, 0.0001
        kFinalAmp = (kf * (iAmpStart-iAmpEnd))+iAmpEnd
    elseif iAmpStart < iAmpEnd then
        kf expseg  0.0001, p3, 1
        kFinalAmp = (kf * (iAmpEnd-iAmpStart))+iAmpStart
    else
        kFinalAmp = iAmpStart
    endif
    xout kFinalAmp
endop
```

Il cuore del sistema è l'UDO `GetIsoAmp`:

```csound
opcode GetIsoAmp, i, ii
    iFrequency, iDynamicIndex xin

    iSafeFrequency = limit(iFrequency, 20, 12500)

    ; 1. Recupera i parametri di base per la dinamica data
    iPhonLevel, iDbfsRef1kHz GetDynamicParams iDynamicIndex
    
    ; 2. Calcola il dB SPL target per la frequenza e il livello phon dati
    iDbSplTarget    PhonToSpl_i     iPhonLevel, iSafeFrequency
    
    ; 3. Il dB SPL di riferimento a 1kHz è per definizione uguale al livello Phon
    iDbSplRef1kHz   =               iPhonLevel
    
    ; 4. Calcola l'offset di compensazione
    iFrequencyOffset = iDbSplTarget - iDbSplRef1kHz
    
    ; 5. Applica l'offset al livello dBFS di riferimento
    iFinalDbfs      = iDbfsRef1kHz + iFrequencyOffset
    
    ; 6. Converti il dBFS finale in ampiezza lineare
    iFinalAmp       = ampdbfs(iFinalDbfs)
    
    xout iFinalAmp
endop
```

La conversione da Phon a SPL utilizza le curve ISO interpolate:

```csound
opcode PhonToSpl_i, i, ii
    iphon, ifreq    xin
        
    ; Interpolazione lineare dalle tabelle ISO
    iaf             Interp  ifreq, giIsoFreqs, giAf
    ilu             Interp  ifreq, giIsoFreqs, giLu
    itf             Interp  ifreq, giIsoFreqs, giTf
    
    ; Formula ISO 226:2003
    iterm1          =       4.47e-3 * (pow(10, 0.025 * iphon) - 1.15)
    iterm2_exp      =       (itf + ilu) / 10.0 - 9
    iterm2          =       pow(0.4 * pow(10, iterm2_exp), iaf)
    iaf_value       =       iterm1 + iterm2
    
    if iaf_value <= 0 then
        ispl        =       itf + (iphon / 40.0) * 20
    else
        ispl        =       (10.0 / iaf) * log10(iaf_value) - ilu + 94.0
    endif
    
    if abs(ifreq - 1000) < 0.1 then
        ispl = iphon
    endif
    
    xout            ispl
endop
```

### Sistema di Spazializzazione Mid-Side e Armoniche Spaziali

La spazializzazione in Gamma va oltre il semplice panning stereofonico, implementando un sistema basato su "armoniche spaziali" che deriva dalla teoria delle armoniche ritmiche. Il concetto chiave è che i valori ritmici non solo organizzano il tempo e selezionano le frequenze, ma definiscono anche il movimento nello spazio stereofonico.

Vediamo come si sviluppa questo sistema partendo dai parametri di base:

```csound
; Parametri di base per la spazializzazione
iwhichZero = abs(p6)    ; quale "zero" della funzione trigonometrica usare
iHR = max(1, abs(p7))   ; Harmonic Ratio - il numero di "spicchi" della circonferenza

; Calcolo del periodo e della posizione iniziale
iPeriod = $M_PI * 2 / iHR
iradi = (iwhichZero > 0 ? (iwhichZero - 1) * iPeriod : 0)
```

Il parametro `iHR` (Harmonic Ratio) determina in quanti "spicchi" viene suddivisa la circonferenza. Ad esempio:
- `iHR = 1`: un solo periodo, movimento completo 0-360°
- `iHR = 4`: quattro periodi, la circonferenza è divisa in quadranti
- `iHR = 7`: sette spicchi, creando una suddivisione asimmetrica

Il parametro `iwhichZero` determina da quale zero della funzione trigonometrica iniziare il movimento:

```csound
; Evoluzione temporale della posizione angolare
kndx_local line 0, p3, 1
ktab tab kndx_local, ifn_shape, 1
krad = iradi + (ktab * iPeriod * i_senso)
```

Qui `krad` evolve nel tempo secondo l'inviluppo specificato da `ifn_shape`, modulato dal senso di movimento (`i_senso` = 1 o -1 per movimento orario/antiorario).

La generazione dell'inviluppo locale usa una modifica della funzione seno quando `ifn_shape == 2`:

```csound
if ifn_shape == 2 then
    kEnv_local = abs(sin(krad * iHR / 2))
else
    kEnv_local tab kndx_local, ifn_shape, 1
endif
```

La formula `abs(sin(krad * iHR / 2))` genera curve polari modificate. Questa trasformazione:
- Prende il valore assoluto, creando lobi sempre positivi
- Moltiplica per `iHR / 2`, dimezzando il numero di lobi rispetto agli spicchi spaziali
- Crea una correlazione diretta tra movimento spaziale e ampiezza

Per comprendere meglio, consideriamo il codice Python fornito che visualizza queste funzioni:

```python
def genera_e_plotta_polare_sine(self):
    theta = np.linspace(0, 2 * np.pi, 500)
    num_funzioni = 10
    
    # Base delle funzioni sinusoidali
    r_base = [np.abs(np.sin(theta * i / 2)) for i in range(1, num_funzioni + 1)]
```

Questo mostra come per `i` crescenti si ottengono curve polari con sempre più lobi, che in Csound diventano pattern di inviluppo sempre più complessi.

La conversione finale da coordinate polari a stereo avviene con:

```csound
; Calcolo delle componenti Mid-Side
kMid = cos(krad)
kSide = sin(krad)

; Applicazione dell'inviluppo al segnale
aMid = kMid * asigEnv 
aSide = kSide * asigEnv

; Conversione a Left-Right con matrice di rotazione
aL = (aMid + aSide) / $SQRT2
aR = (aMid - aSide) / $SQRT2
```

Questa matrice di rotazione:
- Mantiene potenza costante durante il movimento
- Crea un campo stereofonico coerente
- Permette movimenti fluidi nello spazio

### Gestione degli Inviluppi Multipli

Il sistema gestisce due livelli di inviluppo che interagiscono moltiplicativamente:

```csound
; Inviluppo locale dell'evento (derivato dalle armoniche spaziali)
asigLocalEnv = asig * kEnv_local

; Inviluppo di sezione (se presente)
kEnv_section = 1
if i_ifn_section_env > 20 && i_section_duration > 0 then
    k_time_absolute times      
    k_time_since_section_start = k_time_absolute - i_section_start_time
    kndx_section = limit(k_time_since_section_start / i_section_duration, 0, 1)
    kEnv_section tablei kndx_section, i_ifn_section_env, 1
endif

; Combinazione degli inviluppi
asigEnvPre = asigLocalEnv * kEnv_section
asigEnv dcblock asigEnvPre
```

L'inviluppo di sezione permette modulazioni globali su tutti gli eventi di una sezione, mentre l'inviluppo locale (potenzialmente derivato dalle armoniche spaziali) definisce la forma del singolo evento.

## 1.3 Il Sistema di Intonazione Pitagorica

Il sistema di altezze in Gamma si basa su una implementazione personalizzata dell'intonazione pitagorica, gestita dall'opcode `GenPythagFreqs`:

```csound
opcode GenPythagFreqs, i, iiii
  iFund, iNumIntervals, iNumOctaves, iTblNum xin
  iTotalLen = iNumIntervals * iNumOctaves
  iFreqs[] init iTotalLen
  
  iOctave = 0
  iBaseIndex = 0
  
  while iOctave < iNumOctaves do
    iFifth = 3/2
    iFreqs[iBaseIndex] = iFund * (2^iOctave)
    
    ; Genera la serie di quinte per questa ottava
    indx = 1
    iLastRatio = 1
    while (indx < iNumIntervals) do
      iRatio = iLastRatio * iFifth
      ; Riduci all'ottava di riferimento
      while (iRatio >= 2) do
        iRatio = iRatio / 2
      od
      iFreqs[iBaseIndex + indx] = iFund * iRatio * (2^iOctave)
      iLastRatio = iRatio
      indx += 1
    od
```

### Costruzione della Tabella Frequenze

Il sistema genera una tabella bidimensionale concettuale dove:
- Ogni ottava contiene `iNumIntervals` frequenze (200 nel nostro caso)
- Le frequenze sono generate attraverso iterazioni della quinta perfetta (3/2)
- Ogni quinta che supera l'ottava viene riportata all'interno tramite divisione per 2

### Ordinamento e Memorizzazione

Dopo la generazione, le frequenze vengono ordinate all'interno di ogni ottava:

```csound
; Ordina le frequenze per questa ottava
indx = iBaseIndex
while (indx < (iBaseIndex + iNumIntervals - 1)) do
  indx2 = indx + 1
  while (indx2 < (iBaseIndex + iNumIntervals)) do
    if (iFreqs[indx2] < iFreqs[indx]) then
      iTemp = iFreqs[indx]
      iFreqs[indx] = iFreqs[indx2]
      iFreqs[indx2] = iTemp
    endif
    indx2 += 1
  od
  indx += 1
od
```

Questo bubble sort garantisce che le frequenze siano accessibili in ordine crescente all'interno di ogni ottava.

### Mappatura Ottava-Registro-Ritmo

L'accesso alle frequenze avviene attraverso la funzione `calcFrequenza`:

```csound
opcode calcFrequenza, i, iii
    i_Ottava, i_Registro, i_RitmoCorrente xin
    
    ; Calculate octave register
    i_Indice_Ottava = int(i_Ottava * $INTERVALLI)
    ; Calculate interval offset within the octave
    i_OffsetIntervallo = i_Indice_Ottava + int(((i_Registro * $INTERVALLI) / $REGISTRI))
    
    ; Get the frequency from the table using the calculated offset
    i_Freq table max(1, i_OffsetIntervallo + i_RitmoCorrente), gi_Intonazione
    ifreq = min(i_Freq, sr/2-1)
    xout ifreq
endop
```

La formula di indicizzazione `i_OffsetIntervallo + i_RitmoCorrente` crea una relazione diretta tra il valore ritmico e l'altezza selezionata. Questo significa che:

- Ritmi identici in registri diversi producono intervalli correlati
- La sequenza ritmica diventa una sequenza melodica
- Valori ritmici alti tendono verso frequenze più acute all'interno del registro

### Implicazioni Compositive

Questa architettura crea una profonda interconnessione tra dimensione temporale, frequenziale e spaziale. Un pattern ritmico [3, 5, 8, 13] non solo definisce:
- Le durate relative degli eventi (durataArmonica/3, durataArmonica/5, etc.)
- Le altezze selezionate dalla tabella pitagorica
- Il numero di suddivisioni spaziali e il pattern di movimento stereofonico
- La forma dell'inviluppo di ampiezza quando si usano le armoniche spaziali

L'uso dell'intonazione pitagorica invece del temperamento equabile aggiunge ulteriore ricchezza armonica: le quinte sono pure (rapporto 3:2), ma questo genera comma pitagorici e intervalli microtonali che colorano il risultato sonoro con battimenti e risonanze particolari.

L'orchestra Gamma dimostra come un'architettura ben progettata possa creare connessioni profonde tra parametri apparentemente indipendenti, trasformando relazioni numeriche astratte in strutture musicali percettivamente significative. La gerarchia Voce → eventoSonoro, supportata dal sistema di intonazione pitagorica, dalle sofisticate tecniche di compensazione isofonica e dal sistema di armoniche spaziali, fornisce al compositore uno strumento di straordinaria flessibilità espressiva, capace di generare tessiture sonore complesse da specifiche relativamente semplici.

## 1.4 NonlinearFunc: Il Generatore di Ritmi Caotici

L'opcode `NonlinearFunc` rappresenta uno degli elementi più innovativi di Gamma, fornendo un sistema sofisticato per la generazione di sequenze ritmiche che evolvono nel tempo secondo principi deterministici, periodici o caotici. Questo UDO (User Defined Opcode) estende le possibilità compositive oltre i pattern ritmici predefiniti, permettendo l'esplorazione di territori ritmici emergenti.

### Struttura e Parametri dell'Opcode

```csound
opcode NonlinearFunc, i, ippo
  iX, iMode, iMinVal, iMaxVal xin
  
  ; Valori di default per min/max se non specificati
  iMinVal = (iMinVal == 0) ? 1 : iMinVal
  iMaxVal = (iMaxVal == 0) ? 35 : iMaxVal
  
  ; Assicurati che iX sia entro limiti sensati
  iX = limit(iX, 1, 100)
  
  iPI = 4 * taninv(1.0)  
  iTemp = 0
```

L'opcode accetta quattro parametri:
- **iX**: Il valore di input, tipicamente il ritmo precedente nella sequenza
- **iMode**: Selettore della modalità operativa (0-3)
- **iMinVal**: Valore minimo del range di output (default: 1)
- **iMaxVal**: Valore massimo del range di output (default: 35)

La prima operazione importante è la normalizzazione e limitazione dei valori di input per garantire stabilità numerica. Il valore di iX viene limitato tra 1 e 100 per evitare overflow o comportamenti indefiniti nelle funzioni matematiche successive.

### Modalità 0: Convergente

```csound
if iMode == 0 then
    ; --- MODALITÀ 0: CONVERGENTE ---
    iR = 2.8
    iTemp = iR * iX * (1 - iX/40)
```

Questa modalità implementa una variante della mappa logistica con comportamento convergente. Il parametro `iR = 2.8` è scelto specificamente per rimanere nella regione stabile del diagramma di biforcazione della mappa logistica, dove il sistema converge verso un punto fisso.

La formula `iR * iX * (1 - iX/40)` differisce dalla classica mappa logistica `r * x * (1 - x)` per il fattore di scala 40. Questo adattamento:
- Permette di lavorare con valori di input nell'intervallo 1-100 invece di 0-1
- Rallenta la convergenza, rendendo l'evoluzione ritmica più graduale
- Crea una traiettoria prevedibile verso un valore stabile

Matematicamente, per `iR = 2.8`, il sistema convergerà verso il punto fisso:
```
x* = 40 * (1 - 1/iR) ≈ 25.71
```

Questo significa che sequenze ritmiche in modalità convergente tenderanno gradualmente verso valori intorno a 26, creando un effetto di stabilizzazione ritmica.

### Modalità 1: Periodica

```csound
elseif iMode == 1 then
    ; --- MODALITÀ 1: PERIODICA ---
    iP1 = sin(iX * iPI/18)
    iP2 = cos(iX * iPI/10)
    iTemp = abs(iP1 * iP2) * 20 + 10
```

La modalità periodica utilizza l'interferenza di due funzioni trigonometriche con periodi incommensurabili per generare pattern complessi ma deterministici.

L'analisi matematica rivela:
- `sin(iX * π/18)`: periodo di 36 unità
- `cos(iX * π/10)`: periodo di 20 unità
- Il minimo comune multiplo è 180, creando un super-periodo

Il prodotto `iP1 * iP2` genera un'interferenza costruttiva e distruttiva tra le due onde:
- Quando entrambe le funzioni sono vicine ai loro massimi/minimi, il prodotto è grande
- Quando una è vicina a zero, il prodotto si annulla
- Il valore assoluto garantisce output positivi

La trasformazione finale `abs(iP1 * iP2) * 20 + 10`:
- Scala il range da [0, 1] a [0, 20]
- Aggiunge un offset di 10, risultando in valori tra 10 e 30
- Garantisce che i ritmi generati rimangano in un range musicalmente utile

Questa modalità produce sequenze che si ripetono dopo 180 iterazioni ma con una struttura interna ricca di variazioni locali.

### Modalità 2: Caotica Deterministica

```csound
elseif iMode == 2 then
    ; --- MODALITÀ 2: CAOTICA DETERMINISTICA ---
    iR = 3.99
    iNormX = (iX % 100) / 100
    iNormX = limit(iNormX, 0.01, 0.99)
    iLogistic = iR * iNormX * (1 - iNormX)
    iNoise = random:i(-0.05, 0.05)
    iLogistic = limit(iLogistic + iNoise, 0, 1)
    iRange = iMaxVal - iMinVal + 1
    iTemp = iMinVal + (iLogistic * iRange)
```

Questa modalità implementa la mappa logistica nella sua regione caotica con l'aggiunta di una piccola perturbazione stocastica.

Il parametro `iR = 3.99` posiziona il sistema al limite del caos:
- Per r > 3.57, la mappa logistica entra nel regime caotico
- A r = 3.99, siamo nella regione di caos sviluppato
- Piccole variazioni nell'input producono grandi divergenze nell'output

Il processo di normalizzazione `(iX % 100) / 100`:
- Utilizza l'operatore modulo per mantenere i valori ciclici
- Normalizza nell'intervallo [0, 1] richiesto dalla mappa logistica
- Il limite `[0.01, 0.99]` evita i punti fissi instabili a 0 e 1

L'aggiunta di rumore `random:i(-0.05, 0.05)`:
- Introduce una componente stocastica del 5%
- Previene cicli perfetti che potrebbero emergere anche nel caos deterministico
- Simula le imperfezioni del mondo reale

### Modalità 3: Caos Vero (Default)

```csound
else
    ; --- MODALITÀ 3: CAOS VERO (DEFAULT) ---
    ; 1. Componente deterministica (60%)
    iSeed1 = (iX * 1.3) % 10
    iSeed2 = (iX * 0.7) % 10
    iSeed3 = (iX * 2.5) % 10
    iNonlinear1 = abs(sin(iSeed1 * iPI/5 + iSeed2))
    iNonlinear2 = abs(cos(iSeed2 * iPI/3 + iSeed3))
    iNonlinear3 = abs(tan(iSeed3 * iPI/7 + iSeed1) % 1)
    iDeterministic = (iNonlinear1 + iNonlinear2 + iNonlinear3) / 3
```

La modalità "Caos Vero" rappresenta l'approccio più sofisticato, combinando molteplici generatori non lineari con componenti stocastiche.

La generazione dei seed utilizza moltiplicatori irrazionali approssimati:
- 1.3 ≈ √1.69 
- 0.7 ≈ 1/√2
- 2.5 ≈ √6.25

Questi valori garantiscono che i tre seed evolvano a velocità diverse e incommensurabili, massimizzando la complessità dell'output.

Le tre funzioni non lineari utilizzano:
- **sin** con accoppiamento additivo: sensibile alle fasi relative
- **cos** con accoppiamento additivo: sfasato di π/2 rispetto a sin
- **tan** con modulo: introduce discontinuità controllate

```csound
    ; 2. Componente casuale (40%)
    iRandom = random:i(0, 1)
    
    ; 3. Combina le componenti
    iMixRatio = 0.6
    iCombined = (iDeterministic * iMixRatio) + (iRandom * (1 - iMixRatio))
```

Il bilanciamento 60/40 tra deterministico e stocastico è calibrato per:
- Mantenere una struttura riconoscibile (componente deterministica)
- Introdurre sufficiente imprevedibilità (componente random)
- Evitare sia la monotonia che il rumore bianco

```csound
    ; 4. Perturbazione periodica
    iPerturbation = 0
    if (iX % 7 == 0) then 
      iPerturbation = random:i(-0.3, 0.3)
    endif
    
    ; 5. Mappa al range finale
    iRange = iMaxVal - iMinVal + 1
    iTemp = iMinVal + (iCombined * iRange) + (iPerturbation * iRange)
```

La perturbazione periodica ogni 7 iterazioni:
- Introduce eventi rari ma significativi
- Il numero 7 (primo) evita risonanze con altri periodi nel sistema
- L'ampiezza ±30% può causare salti drammatici nel ritmo

### Integrazione con il Sistema Gamma

Nel contesto dello strumento Voce, NonlinearFunc viene chiamato quando la tabella dei ritmi predefiniti si esaurisce:

```csound
i_RitmoCorrente NonlinearFunc i_Vecchio_Ritmo, i_NonlinearMode
```

Questo crea una transizione fluida da:
1. **Fase deterministica**: Ritmi composti e memorizzati in tabella
2. **Fase generativa**: Ritmi creati algoritmicamente

L'output di NonlinearFunc influenza direttamente:
- **Temporalità**: Attraverso la formula `i_DurataArmonica / i_RitmoCorrente`
- **Altezza**: Il ritmo viene usato come indice nella tabella delle frequenze
- **Spazializzazione**: Determina il parametro iHR per le armoniche spaziali

### Implicazioni Compositive e Estetiche

L'uso di NonlinearFunc permette di esplorare diverse estetiche ritmiche:

- **Modalità 0 (Convergente)**: Crea un senso di "arrivo" o "risoluzione" ritmica, utile per conclusioni o punti di stasi
- **Modalità 1 (Periodica)**: Genera groove complessi ma ripetitivi, ideale per sezioni di sviluppo
- **Modalità 2 (Caotica Deterministica)**: Produce variazioni continue senza ripetizioni, perfetta per tessiture in evoluzione
- **Modalità 3 (Caos Vero)**: Bilancia imprevedibilità e coerenza, creando interesse sostenuto

La possibilità di cambiare modalità durante la composizione (attraverso il parametro YAML `nonlinear_mode`) permette di modulare il grado di prevedibilità/caos nel flusso ritmico, creando archi formali che vanno dall'ordine al disordine e viceversa.

L'implementazione di NonlinearFunc dimostra come principi matematici complessi possano essere tradotti in strumenti compositivi pratici, offrendo al compositore un controllo parametrico su processi generativi sofisticati senza richiedere una comprensione profonda della matematica sottostante.
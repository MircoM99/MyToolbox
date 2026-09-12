# LLC Design Tool — Descrizione

**File:** `llc_design_tool_standalone.html`
**Tipo:** applicazione web standalone (HTML singolo, nessuna installazione)
**Ambito:** dimensionamento di un convertitore risonante LLC half-bridge, secondo la metodologia FHA (First Harmonic Approximation)

---

## 1. Cosa fa

Il tool implementa in modo interattivo l'intero flusso di dimensionamento di un convertitore LLC risonante half-bridge con raddrizzatore secondario center-tap, dall'inserimento dei requisiti elettrici fino allo stress sui componenti e alla verifica dello ZVS (Zero Voltage Switching). Ogni modifica ai parametri di ingresso ricalcola istantaneamente tutti i risultati a valle.

A differenza dell'approccio manuale classico (dove si impone una frequenza di risonanza *target* e si itera manualmente sul rapporto m), qui:

- **fr è un dato di progetto fisso**, scelto liberamente dall'utente in base a densità di potenza/perdite desiderate;
- **fs_min è un output calcolato**, non un vincolo di ingresso;
- **il rapporto m converge automaticamente per bisezione numerica**, entro un range configurabile dall'utente, invece di essere aggiustato a tentativi.

---

## 2. Come si usa

Il file `.html` è completamente autonomo: si apre con doppio click nel browser (Chrome, Edge, Firefox), senza installazione né account. Le librerie (React, Tailwind) vengono caricate da CDN pubblici al primo avvio — serve quindi una connessione internet almeno alla prima apertura.

---

## 3. Interfaccia

- **Pannello input** (sinistra): tutti i parametri di progetto, organizzati per sezione (Ingresso, Uscita, Tank risonante, Trasformatore, Condensatori di uscita).
- **Pannello risultati** (destra): un blocco per ciascuno degli 11 step del flusso di calcolo, aggiornato in tempo reale.
- **Grafico curva di guadagno K(Fx)**: rendering SVG nativo con crosshair interattivo al passaggio del mouse, tre curve (Q≈0, Qmax/2, Qmax) e riferimenti a Mmax target, Mnom=1 e Fx_min.
- **Pulsante "Metodologia"**: apre un pannello con la procedura completa, tutte le formule esatte implementate e le ipotesi del modello — consultabile senza uscire dal tool.
- **Pulsante "Progetti"**: salvataggio/caricamento di configurazioni complete, persistenti nel browser (localStorage), più export/import su file `.json` per backup o trasferimento su altra macchina.
- **Toggle tema chiaro/scuro**: preferenza salvata e ripristinata automaticamente.

---

## 4. Flusso di calcolo (11 step)

| Step | Contenuto |
|---|---|
| 01 | Guadagni richiesti: Mmax, Mmin, Mnom, rapporto spire n |
| 02 | Resistenza di carico riflessa Rac |
| 03–05 | Iterazione automatica su m (bisezione numerica) e ricerca di Fx_min |
| 06–07 | Componenti risonanti Lr, Cr, Lm e range di frequenza di switching risultante |
| 08 | Dimensionamento trasformatore (Np, Ns, verifica induzione B) |
| 09 | Stress sui componenti (MOSFET, Cr, diodi raddrizzatori) |
| 10 | Verifica ZVS su tutto il range di carico |
| 11 | Dimensionamento condensatori di uscita (corrente RMS, numero in parallelo, ripple di tensione) |
| 12 | Verifica capacità di commutazione ZVS in funzione del tempo morto e della Coss dei MOSFET |

---

## 5. Formule chiave implementate

**Guadagno del tank risonante (FHA):**

```
K(Fx,m,Q) = Fx²(m−1) / √{ (m·Fx²−1)² + Fx²(Fx²−1)²Q²(m−1)² }
```

Verificata numericamente contro l'esempio di progetto dell'Infineon AN 2012-09 (Qmax=0.4, m=6.3 → Fx_min=0.489, K=1.974). A Fx=1 dà sempre K=1 indipendentemente da m e Q — proprietà caratteristica della topologia LLC.

**Rapporto spire e guadagni richiesti:**

```
n = Vin_nom / (2·(Vout_nom+VF))
M = 2·n·Vout / Vin
Mmax_target = Mmax_richiesto × (1 + margine_FHA)
```

**Componenti risonanti:**

```
Lr = (Qmax·Rac) / (2π·fr)
Cr = 1 / (2π·fr·Qmax·Rac)
Lm = (m−1)·Lr
```

**Corrente nel condensatore di uscita (raddrizzamento a onda intera, center-tap):**

```
I_Cout(t) = (π·Io/2)·|sin(ωt)| − Io
I_Cout_RMS = Io·√(π²/8 − 1) ≈ 0.483·Io        (integrata numericamente, coincide con la forma chiusa nota in letteratura)
N = ⌈ I_Cout_RMS / I_C_rated ⌉
ΔV_ripple = (ESR_singolo/N) · (π/2)·Io          (ESR_eff × escursione picco-picco della corrente)
```

**Verifica capacità di commutazione ZVS (tempo morto):**

Il guadagno di tank verificato allo Step 10 garantisce funzionamento in regione induttiva, ma non basta da solo: la corrente di magnetizzazione deve anche essere *quantitativamente sufficiente* a caricare/scaricare le Coss dei due MOSFET durante il tempo morto disponibile.

```
Im_pk = Vin / (8·Lm·fs)                          (corrente di magnetizzazione di picco, forma triangolare)
Im_pk · t_dead ≥ 2·Coss·Vin                       (criterio di carica sufficiente)
```

Il termine Vin si semplifica in entrambi i membri: il criterio non dipende dalla tensione di ingresso, solo da Lm, tempo morto, Coss e frequenza di switching. Valutato al caso peggiore fs=fr (frequenza più alta nel range operativo boost, dove la magnetizzante è minima):

```
Lm_max = t_dead / (16·Coss·fr)
```

Se Lm (calcolato allo Step 6) supera questo limite, la corrente di magnetizzazione non basta a garantire ZVS nel tempo morto configurato.
```

L'elenco completo di tutte le formule per ciascuno degli 11 step è consultabile direttamente nel tool tramite il pulsante **Metodologia**.

---

## 6. Input richiesti

| Categoria | Parametri |
|---|---|
| Ingresso | Vin nominale, minima, massima |
| Uscita | Potenza, Vout (fissa o variabile con min/max), tipo raddrizzatore (diodo/sincrono) |
| Tank risonante | fr, Qmax, range di ricerca di m (min/max), margine di sicurezza FHA |
| Trasformatore | Bmax, Ae (area efficace nucleo) |
| Condensatori di uscita | I_C_rated (da datasheet), ESR del singolo condensatore (da datasheet) |
| Verifica ZVS (tempo morto) | Tempo morto del driver/microcontrollore, Coss del singolo MOSFET (da datasheet) |

---

## 7. Output prodotti

- Rapporto spire n (teorico e reale dopo arrotondamento a spire intere)
- Componenti risonanti Lr, Cr, Lm
- Range di frequenza di switching risultante (fs_min ↔ fr)
- Parametri trasformatore (Np, Ns, B verificato, correnti RMS avvolgimenti)
- Stress su MOSFET, Cr, diodi raddrizzatori
- Esito e margine della verifica ZVS
- Numero di condensatori di uscita in parallelo e ripple di tensione risultante
- Esito della verifica di capacità di commutazione ZVS (Lm massima ammessa, margine, corrente di magnetizzazione di picco)
- Grafico interattivo della curva di guadagno K(Fx)

---

## 8. Persistenza dei dati

- **Progetti salvati**: legati al browser/PC specifico (localStorage), sopravvivono alla chiusura e riapertura del file.
- **Export/import su file `.json`**: per backup indipendenti dal browser o trasferimento su un'altra macchina.
- **Compatibilità**: caricare un progetto salvato con una versione precedente del tool applica automaticamente i valori di default ai parametri introdotti successivamente (es. range di ricerca m, dati condensatori di uscita).

---

## 9. Limiti del modello

- Il modello FHA considera solo la componente fondamentale della tensione quadra applicata al tank; il guadagno reale sotto-risonanza (fs<fr) è tipicamente superiore del 10–15% — da qui l'opzione di margine di sicurezza.
- Si assume duty cycle 50% simmetrico; sono trascurati tempi morti, non linearità dei semiconduttori e variazioni parametriche dei componenti magnetici.
- La stima di tensione di picco su Cr è approssimata (bias DC + componente AC di picco) — da verificare in simulazione.
- Il criterio di verifica ZVS a tempo morto (Step 12) approssima la corrente disponibile con il solo valore di picco della magnetizzante, trascurando il contributo della corrente risonante: è conservativo, non un calcolo di commutazione esatto.
- Non sostituisce una verifica tramite simulazione circuitale (SPICE/SIMPLIS) prima della realizzazione fisica.

---

## 10. Note di validazione

Durante lo sviluppo sono stati identificati e corretti tre errori nelle formule fornite come specifica iniziale, tutti verificati numericamente prima della correzione:

1. **Formula del guadagno K(Fx,m,Q)**: il termine `m·Fx²−(m−1)` nella bozza iniziale è stato corretto in `m·Fx²−1`, verificato contro l'esempio di progetto Infineon AN 2012-09.
2. **Corrente nel condensatore di uscita**: mancava il valore assoluto su `sin(ωt)` — una corrente raddrizzata non può essere negativa. La correzione ha ridotto l'RMS calcolato di un fattore ~3×.
3. **Ripple di tensione ΔV**: il coefficiente era π invece di π/2 (l'escursione picco-picco della corrente sul condensatore, non Io stesso, è il termine corretto da moltiplicare per l'ESR). La correzione ha dimezzato il ΔV calcolato.

---

## 11. Requisiti tecnici

- Browser moderno (Chrome, Edge, Firefox, Safari)
- Connessione internet alla prima apertura (CDN: React, ReactDOM, Babel Standalone, Tailwind CSS)
- Nessun account, nessuna installazione, nessun dato inviato a server esterni — tutto il calcolo avviene localmente nel browser

---

## 12. Riferimenti

- Infineon AN 2012-09, *"Resonant LLC Converter: Operation and Design"*, Sam Abdel-Rahman
- ON Semiconductor AND90061/D, *"Half-Bridge LLC Resonant Converter Design Using NCP4390/NCV4390"*

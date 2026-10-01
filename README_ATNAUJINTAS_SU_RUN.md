# Dviračių nuomos paklausos prognozavimas

## Projekto paskirtis

Projekto tikslas – prognozuoti **kitos valandos dviračių nuomos paklausą**, kad operatorius galėtų
geriau planuoti dviračių perskirstymą. Naudojamas UCI **Bike Sharing** valandinis duomenų rinkinys.

Sprendime palyginamas paprastas sezoninis atskaitos metodas ir keturi intelektualieji metodai:

- **MLP** – daugiasluoksnis perceptronas;
- **CatBoost** – gradientinio stiprinimo modelis;
- **Voting Ensemble** – optimizuotas GBR ir HGBR ansamblis;
- **N-BEATS tipo modelis** – lengva `backcast` / `forecast` blokų realizacija.

> N-BEATS realizacija šiame projekte yra supaprastinta N-BEATS tipo bendroji architektūra,
> o ne identiška oficialaus autorių kodo kopija.

---

## Eksperimento logika

Laiko eilutė nėra maišoma atsitiktinai.

1. Duomenys paruošiami chronologine tvarka.
2. Paskutinės maždaug 30 dienų atskiriamos galutiniam **TEST**.
3. Likusioje **DEV** dalyje naudojami 3 chronologiniai `rolling-origin` skaidiniai.
4. Hiperparametrai ir Voting svoris parenkami tik DEV dalyje.
5. Galutiniame 06 Notebook užfiksuoti modeliai apmokomi DEV ir vieną kartą vertinami TEST.
6. Pamačius TEST rezultatą modelių hiperparametrai nebekeičiami.

Naudojama fiksuota atsitiktinė sėkla:

```python
SEED = 42
```

### Duomenų nutekėjimo kontrolė

- `casual` ir `registered` nenaudojami kaip požymiai, nes jų suma sudaro `cnt`;
- `lag` ir `rolling` požymiai sudaromi tik iš ankstesnių laiko momentų;
- scaler ir imputer parametrai išmokstami tik mokymo dalyje;
- piko slenkstis nustatomas tik iš mokymo / DEV duomenų;
- TEST nenaudojamas hiperparametrų parinkimui.

---

## Metrikos

**Pagrindinė modelio palyginimo metrika – MAE.**

- **MAE** – vidutinė absoliuti prognozavimo klaida;
- **RMSE** – labiau išryškina didesnes prognozavimo klaidas;
- **Peak MAE** – MAE tik paklausos piko valandomis;
- **PUR (Peak Underestimation Rate)** – papildomas diagnostinis rodiklis, rodantis, kokia
  piko valandų dalis buvo neįvertinta.

PUR **nenaudojamas pagrindiniam modelio pasirinkimui**. Jei pagal MAE geresnis modelis turi
prastesnį PUR, tai aprašoma kaip modelio ribotumas.

---

## Projekto struktūra

Rekomenduojami galutinių failų pavadinimai:

```text
Egzaminui/
│
├── hour.csv
├── README.md
├── requirements.txt
├── run.py
├── AI_naudojimo_zurnalas.txt
├── SALTINIAI.txt
│
├── 01_Duomenu_paruosimas_ir_atskaitos_metodas.ipynb
├── 02_MLP_modelio_eksperimentai.ipynb
├── 03_CatBoost_modelio_eksperimentai.ipynb
├── 04_Optimizuoto_ansamblio_eksperimentai.ipynb
├── 05_NBEATS_tyrimas_ir_tobulinimas.ipynb
├── 06_Bendras_TEST_rezultatu_palyginimas.ipynb
│
├── src/
│   └── common.py
│
└── results/
    ├── best_catboost.json
    ├── best_voting.json
    ├── best_nbeats.json
    ├── test_modeliu_palyginimas.csv
    └── ...
```

`results/` katalogo dalis failų sukuriama paleidžiant Notebook.

---

## Notebook paskirtis

| Notebook | Paskirtis |
|---|---|
| `01_Duomenu_paruosimas_ir_atskaitos_metodas.ipynb` | Duomenų paruošimas, chronologinis skaidymas, leakage patikros, sezoninis Baseline |
| `02_MLP_modelio_eksperimentai.ipynb` | MLP hiperparametrų paieška ir DEV analizė |
| `03_CatBoost_modelio_eksperimentai.ipynb` | CatBoost grid + Optuna DEV optimizavimas |
| `04_Optimizuoto_ansamblio_eksperimentai.ipynb` | GBR + HGBR optimizavimas ir Voting svorio parinkimas |
| `05_NBEATS_tyrimas_ir_tobulinimas.ipynb` | N-BEATS tipo modelis, tobulinimas, abliacija ir atsparumo bandymas |
| `06_Bendras_TEST_rezultatu_palyginimas.ipynb` | Vienkartinis galutinis visų metodų TEST palyginimas ir klaidų analizė |


## Diegimas

Rekomenduojama aplinka:

- Windows 10/11;
- Python 3.11;
- VS Code su Jupyter plėtiniu arba Jupyter Notebook.

Terminale projekto kataloge:

```bash
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

Bibliotekų diegimas:

```bash
python -m pip install -r requirements.txt
```

---

## Duomenų atsisiuntimas

Naudojamas UCI Bike Sharing Dataset:

https://archive.ics.uci.edu/dataset/275/bike+sharing+dataset

Atsisiųskite duomenų rinkinį, išskleiskite `hour.csv` ir įdėkite jį į pagrindinį
`Egzaminui` katalogą.

`hour.csv` kopijos kartu su projektu pateikti nebūtina, jei paliekama ši atkuriama
atsisiuntimo instrukcija.

---

## Paleidimas

### Rekomenduojamas būdas – visas projektas viena komanda

Kad pagrindinį eksperimentą būtų galima pakartoti viena komanda, projekto kataloge pateiktas failas:

```text
run.py
```

Prieš pirmą paleidimą įdiegiamos visos reikalingos bibliotekos:

```bash
python -m pip install -r requirements.txt
```

Tada visas pagrindinis eksperimentas paleidžiamas viena komanda:

```bash
python run.py
```

`run.py` automatiškai:

1. patikrina, ar pagrindiniame kataloge yra `hour.csv`, `src/common.py` ir visi 6 Notebook;
2. paruošia Jupyter kernelį naudodamas tą patį Python interpreterį, kuriuo paleistas `run.py`;
3. nuosekliai vykdo Notebook tokia tvarka:

```text
01 → 02 → 03 → 04 → 05 → 06
```

4. jei kuriame nors Notebook įvyksta klaida, vykdymą sustabdo ir terminale parodo, kuriame faile ji atsirado;
5. sėkmingai įvykdytų Notebook išvestis išsaugo pačiuose `.ipynb` failuose;
6. palieka Notebook sukurtus papildomus rezultatus `results/` kataloge.

Taip viena komanda pakartojama visa pagrindinė eksperimento eiga: duomenų paruošimas ir Baseline, MLP, CatBoost, Voting Ensemble, N-BEATS tyrimas bei galutinis TEST palyginimas.

> **Svarbu:** `run.py` nekeičia eksperimento metodikos. Modelių ir hiperparametrų pasirinkimas atliekamas DEV dalyje, o galutinis TEST naudojamas tik užfiksuotų modelių vienkartiniam palyginimui.

### Reikalingi tikslūs Notebook pavadinimai

Kad `run.py` galėtų automatiškai rasti failus, galutinėje projekto versijoje Notebook turi vadintis:

```text
01_Duomenu_paruosimas_ir_atskaitos_metodas.ipynb
02_MLP_modelio_eksperimentai.ipynb
03_CatBoost_modelio_eksperimentai.ipynb
04_Optimizuoto_ansamblio_eksperimentai.ipynb
05_NBEATS_tyrimas_ir_tobulinimas.ipynb
06_Bendras_TEST_rezultatu_palyginimas.ipynb
```

Jeigu darbo metu naudotos versijos turi papildomus žodžius, pvz. `_GALUTINIS`, prieš galutinį pateikimą jas reikia pervadinti pagal šį sąrašą.

### Alternatyvus rankinis paleidimas

Jei nenorima naudoti `run.py`, Notebook galima atidaryti ir vykdyti rankiniu būdu. Kiekviename pasirenkama **Restart Kernel → Run All**, laikantis tos pačios eilės:

```text
01 → 02 → 03 → 04 → 05 → 06
```

Svarbu nekeisti eiliškumo, nes vėlesni Notebook naudoja ankstesniuose DEV eksperimentuose išsaugotas modelių konfigūracijas.

---

## Paskutinio galutinio TEST paleidimo rezultatai

Bendram palyginimui naudotos **627 TEST valandos**, iš jų **47 piko atvejai**.

| Modelis | MAE ↓ | RMSE ↓ | Peak MAE ↓ | PUR ↓* |
|---|---:|---:|---:|---:|
| **N-BEATS** | **22,233** | **37,546** | 69,643 | **0,404** |
| CatBoost | 25,238 | 40,591 | **60,688** | 0,787 |
| Voting Ensemble | 27,827 | 42,182 | 68,960 | 0,787 |
| MLP | 36,775 | 50,592 | 69,533 | 0,638 |
| Baseline | 63,477 | 103,614 | 97,683 | 0,915 |

\* PUR yra papildomas diagnostinis rodiklis, ne pagrindinis atrankos kriterijus.

### Rezultatų interpretacija

Pagal pagrindinę **MAE** metriką šiame galutiniame TEST paleidime geriausiai pasirodė
**N-BEATS tipo modelis**. Jo MAE buvo apie **65,0 % mažesnė** už sezoninio Baseline MAE.

N-BEATS taip pat pasiekė mažiausią RMSE ir mažesnį PUR, tačiau **CatBoost turėjo mažesnę
Peak MAE**. Tai reiškia, kad skirtingos metrikos išryškina skirtingas modelių savybes:
N-BEATS turėjo mažiausią bendrą klaidą, o CatBoost – mažesnę absoliučią klaidą piko valandomis.

Šis skirtumas nekeičia pagrindinės išvados pagal iš anksto nustatytą MAE kriterijų, bet yra
svarbus modelio ribotumų ir praktinio taikymo vertinimui.

---

## Abliacija ir atsparumo bandymas

**Abliacija:** N-BEATS modeliui palyginamas 24 val. ir 168 val. retrospektyvus istorijos langas.
Taip tikrinama, ar ilgesnė paklausos istorija suteikia papildomos naudos.

**Atsparumo bandymas:** 5 % validavimo sekos įvesties reikšmių pakeičiama trūkstamomis
reikšmėmis ir užpildoma TRAIN medianine reikšme. Modelis vertinamas tomis pačiomis DEV
sąlygomis kaip švarūs duomenys.

Šie eksperimentai atliekami DEV, o ne galutiniame TEST.

---

## Praktinė reikšmė ir ribotumai

Per maža paklausos prognozė piko metu gali lemti nepakankamą dviračių kiekį reikiamoje vietoje,
o per didelė prognozė – nereikalingą perskirstymą. Todėl šalia bendros MAE svarbu analizuoti
piko klaidas ir didžiausių klaidų pavyzdžius.

Pagrindiniai ribotumai:

- duomenys apima 2011–2012 m. vienos dviračių dalijimosi sistemos istoriją;
- modelių rezultatai nebūtinai tiesiogiai perkeliami į kitą miestą ar kitą laikotarpį;
- naudojami istoriniai meteorologiniai duomenys, o realioje sistemoje prognozavimo metu dalis jų
  turėtų būti pakeista realiai prieinama orų prognoze;
- N-BEATS realizacija yra supaprastinta N-BEATS tipo architektūra;
- vienos valandos į priekį N-BEATS vertinime naudojamos iki prognozavimo momento jau stebėtos
  faktinės paklausos reikšmės, todėl tai nėra daugiažingsnė prognozė be naujų stebėjimų.

---

## Šaltiniai ir AI naudojimas

Pilnas šaltinių sąrašas pateiktas faile:

```text
SALTINIAI.txt
```

AI naudojimo, priimtų ir atmestų pasiūlymų, aptiktų klaidų bei jų patikros aprašymas:

```text
AI_naudojimo_zurnalas.txt
```



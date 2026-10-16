# BarSnap – experimentálny plán pre presnosť <= 5 %

## Definícia úspechu

Pre podporované varianty fliaš nechceme hodnotiť iba priemernú chybu.

Pre každý BottleVariant sledujeme:

- MAE v ml,
- Median Absolute Error v ml,
- P95 Absolute Error v ml,
- absolute error ako % nominálneho objemu,
- reject rate,
- chybovosť identifikácie produktu,
- chybovosť identifikácie BottleVariant.

Prvá praktická acceptance hranica:

```text
P95 AE <= 5 % nominálneho objemu
```

Pre 700 ml fľašu je hranica 35 ml.

Pri produktoch s vysokou cenou môžeme nastaviť strict mode, napr. cieľ 3 % alebo povinné dva snímky.

## Fáza A – overiť fyziku bez AI

Vybrať 5–10 transparentných fliaš s rôznou geometriou:

- rovná cylindrická,
- obdĺžniková,
- výrazné ramená,
- široké telo + úzke hrdlo,
- farebné sklo.

Pre každú fľašu vytvoriť kalibračnú sadu:

- prázdna,
- následne pridávať vodu alebo vhodnú testovaciu kvapalinu po 25/50 ml,
- pri každom bode zaznamenať presný objem,
- odfotiť na štandardnom pozadí zpredu.

Cieľ fázy A:

1. ručne označiť hladinu,
2. z fotografie vypočítať normalizované `h`,
3. fitnúť V(h),
4. potvrdiť, že pri dokonale známej hladine dokáže geometrická vrstva dosiahnuť cieľ.

Ak nie, problém je v kalibrácii/geometrii, nie v AI.

## Fáza B – automatická detekcia hladiny

Pre tie isté kalibrované fľaše vytvoriť fotografie pri variáciách:

- 3–5 vzdialeností,
- horizontálne posunutie,
- tilt približne -10° až +10°,
- denné svetlo,
- teplé barové svetlo,
- slabšie svetlo,
- tmavé a svetlé pozadie,
- mierne odlesky,
- rôzne telefóny, ak sú dostupné.

Dataset split musí byť po fyzických fľašiach/scénach, nie náhodne po veľmi podobných frameoch, aby sme nemerali leakage.

## Fáza C – identifikácia variantu

Pre každý produkt, ktorý má viac historických alebo paralelných obalov:

- vytvoriť samostatný BottleVariant,
- zozbierať reference images z viacerých strán,
- vytvoriť embedding index,
- testovať product accuracy a variant accuracy oddelene.

Dôležité: nový obal sa nesmie potichu mapovať na starú kalibráciu iba preto, že má rovnakú značku alebo EAN.

Ak variant confidence < threshold:

```text
UNKNOWN_VARIANT
```

je správnejší výsledok než nesprávny objem.

## Fáza D – barový field test

Pilot v jednej prevádzke:

1. používateľ odfotí fľašu BarSnapom,
2. systém predikuje objem,
3. fľaša sa následne odmeria referenčnou metódou,
4. uloží sa prediction + ground truth + podmienky.

Ground truth možnosti:

- pre experiment odliať a odmerať obsah,
- gravimetricky: presná váha + tara fľaše + hustota kvapaliny,
- pri vode počas vývoja priamo hmotnosť ~= objem.

Pre alkoholický obsah treba pri vážení zohľadniť hustotu konkrétnej kvapaliny alebo merať iba referenčné validačné vzorky presným volumetrickým spôsobom.

## Dataset schema

Každá fotografia má mať minimálne:

```json
{
  "image_id": "...",
  "product_id": "...",
  "bottle_variant_id": "...",
  "nominal_volume_ml": 700,
  "ground_truth_volume_ml": 425,
  "ground_truth_level_normalized": 0.613,
  "phone_model": "...",
  "lighting": "warm_bar",
  "background": "mixed",
  "tilt_deg": 4.2,
  "occlusion": "label_partial",
  "split": "test"
}
```

## Error budget

Celková chyba vzniká minimálne zo štyroch zdrojov:

```text
product error
+ variant error
+ level detection error
+ calibration/geometric error
```

Pre cieľ 5 % je potrebné merať jednotlivé komponenty oddelene.

Príklad interného error budgetu pre 700 ml:

- product/variant: prakticky 0 ml pri správnom accept/reject,
- level detection: P95 ekvivalent <= 20 ml,
- calibration: P95 <= 15 ml,
- výsledný systém: validovať empiricky, nesčítať hranice naivne lineárne.

## Quality gate experiment

Pre každý dôvod zamietnutia vytvoriť samostatnú validačnú množinu:

- bottle cropped,
- heavy blur,
- heavy glare,
- level behind opaque label,
- multiple bottles,
- bottle tilted,
- unknown bottle variant,
- dark opaque glass.

Metrika nie je iba presnosť accepted scanov, ale aj to, či systém dokáže nebezpečné snímky odmietnuť.

## Baseline algoritmy

Porovnávať minimálne:

### Baseline 0
Ručná hladina + kalibračná krivka.

### Baseline 1
Klasický OpenCV threshold/edge detector v známej ROI.

### Baseline 2
Segmentation/keypoint model + klasický level detector.

### Baseline 3
Learned liquid-level model + kalibračná krivka.

### Baseline 4
Ensemble learned + classical CV s reject policy.

Články typu „50 lines of Python“ zaradiť ako Baseline 1, nie ako finálne riešenie.

## Prvý pilotný dataset

Odporúčanie pre prvý týždeň experimentov:

- 10 BottleVariant,
- 12–20 známych hladín na variant,
- 5–10 fotografií na hladinu,
- približne 1 000–2 000 fotografií.

To je dostatočné na rozhodnutie o technickej uskutočniteľnosti a odhalenie hlavných failure modes; nie je to finálny produkčný dataset.

## Logging v produkcii

Každý scan musí ukladať:

- model_version,
- calibration_version,
- confidence,
- quality metrics,
- raw predicted ml,
- final confirmed/corrected ml,
- BottleVariant,
- anonymizované technické metadata fotografie.

Manuálne korekcie sa stanú najcennejším zdrojom dát pre active learning.

## Go / No-Go pre MVP

MVP vision measurement pokračuje, ak pilot ukáže:

- P95 <= 5 % pre vybranú množinu podporovaných transparentných BottleVariant,
- rozumný reject rate,
- opakovateľnosť medzi telefónmi,
- scan je pre barmana výrazne rýchlejší než ručný odhad.

Ak cieľ pri voľnom barovom fotení nevychádza, ďalší krok nie je nekonečne zväčšovať model, ale štandardizovať capture UX (overlay, vzdialenosť, tilt, krátke video, prípadne pomocné svetlo) alebo použiť vision + weight hybrid.

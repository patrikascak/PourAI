# BarSnap / PourAI – návrh architektúry v0

## Cieľ

Vybudovať mobilný systém pre inventúru fliaš v baroch, ktorý z fotografie:

1. identifikuje produkt,
2. identifikuje konkrétny variant/tvar fľaše,
3. nájde hladinu kvapaliny,
4. prepočíta hladinu na objem v ml,
5. uloží výsledok s confidence a diagnostikou,
6. umožní manuálnu opravu,
7. exportuje inventúru do CSV/XLSX/JSON a neskôr cez API.

### Presnostný cieľ

Pre podporované priehľadné alebo dostatočne priesvitné fľaše:

- primárny cieľ: `P95 absolute error <= 5 % nominálneho objemu fľaše`,
- sledovať zároveň `MAE (ml)`, `median AE (ml)` a `P95 AE (ml)`,
- nízko-dôveryhodný scan radšej odmietnuť a vyžiadať nové foto, než uložiť zlý odhad.

Príklad: pre 700 ml fľašu je 5 % = 35 ml.

## Prečo nestačí jeden AI model

Priamy regresný model `foto -> ml` bude krehký voči:

- zmenám tvaru tej istej značky v čase,
- rôznym veľkostiam 500/700/750/1000 ml,
- perspektíve a nakloneniu telefónu,
- odleskom, tmavému sklu a farebným tekutinám,
- etiketám zakrývajúcim hladinu,
- rôznemu pozadiu a osvetleniu,
- novým obalom, ktoré model pri tréningu nevidel.

Odporúčaný systém je hybrid: rozpoznanie + segmentácia + geometria + kalibračná krivka.

## Navrhovaný pipeline

```text
Camera capture
  -> quality gate
  -> barcode/OCR/product retrieval
  -> bottle segmentation + keypoints
  -> bottle variant retrieval
  -> liquid-level / meniscus detection
  -> perspective normalization
  -> fill-height normalization
  -> bottle-specific V(h) calibration
  -> uncertainty/confidence
  -> accept / rescan / manual correction
  -> inventory database
```

## 1. Capture / Quality Gate

Mobilná aplikácia nemá iba otvoriť systémový výber obrázka. Potrebujeme riadený scan flow.

Kontroly pred odoslaním:

- presne jedna fľaša,
- celá fľaša v zábere,
- viditeľné dno a vrch,
- fľaša približne vertikálne,
- minimálny motion blur,
- žiadna extrémna presvetlená plocha,
- dostatočný kontrast v oblasti hladiny,
- primeraná vzdialenosť,
- preferovaný zadný fotoaparát.

Neskôr je vhodný krátky burst/video scan a výber najlepšieho frame-u.

## 2. Product vs Bottle Variant

`Product` a `BottleVariant` musia byť oddelené entity.

Príklad:

```text
Product:
  Jameson Irish Whiskey 700 ml

BottleVariant:
  Jameson-700-v2016
  Jameson-700-v2020
  Jameson-700-v2024
```

EAN nemusí byť dostatočný, pretože výrobca môže zmeniť obal bez zmeny produktu/EAN.

Každý variant má vlastné:

- reference images,
- vizuálny embedding,
- rozmery / shape metadata,
- kalibračnú krivku V(h),
- podporovaný rozsah hladiny,
- confidence thresholds,
- dátum/obdobie používania, ak je známe.

### Odporúčané rozpoznanie variantu

1. barcode, ak je dostupný,
2. OCR brand/label ako pomocný signál,
3. image embedding retrieval medzi kandidátmi produktu,
4. silhouette/keypoint similarity,
5. ak confidence nestačí, používateľ vyberie variant alebo vytvorí nový kandidát.

Nový variant teda nemusí vždy znamenať retraining veľkého klasifikátora.

## 3. Bottle Segmentation

Model má nájsť masku fľaše a ideálne kľúčové body:

- dno,
- horný bod tela,
- ramená,
- hrdlo,
- stredová os.

Maska slúži na:

- normalizáciu perspektívy,
- výpočet výšky hladiny relatívne k telu fľaše,
- porovnanie tvaru s BottleVariant.

## 4. Liquid Level Detection

Výstup nemá byť iba percento. Model/algoritmus má vrátiť:

```json
{
  "level_y_normalized": 0.634,
  "level_confidence": 0.93,
  "visible_segments": [...],
  "occluded_by_label": false
}
```

Možné metódy:

- segmentation model pre liquid/air boundary,
- edge/gradient detector vo vnútri masky,
- classical CV ako pomocný signál,
- ensemble viacerých detektorov.

Ak etiketa zakrýva hladinu alebo je sklo príliš tmavé, scan sa má odmietnuť alebo vyžiadať druhý pohľad.

## 5. Prepočet hladiny na ml

Nepoužívať lineárny vzťah:

```text
volume = bottle_capacity * fill_height
```

Pre každý BottleVariant definovať monotónnu kalibračnú funkciu:

```text
V(h) -> ml
```

kde `h` je normalizovaná výška hladiny.

Kalibráciu možno získať experimentálne pri známych objemoch, napr. po 25/50 ml krokoch, a interpolovať monotónnym spline/PCHIP modelom.

## 6. Confidence a reject policy

Výsledok sa automaticky uloží iba ak:

- product confidence >= threshold,
- variant confidence >= threshold,
- level confidence >= threshold,
- quality gate prejde,
- odhad nie je mimo kalibrovaného rozsahu.

Inak:

- vyžiadať nové foto,
- vyžiadať druhý uhol,
- alebo umožniť manuálnu opravu.

Pre drahé fľaše môže aplikácia používať prísnejší threshold alebo dvojitý scan.

## 7. Dátový model

Minimálne tabuľky:

```text
organizations
venues
users

products
  id
  brand
  name
  nominal_volume_ml
  ean

bottle_variants
  id
  product_id
  variant_code
  geometry_version
  calibration_version
  calibration_curve_json
  reference_embedding
  active_from
  active_to

inventory_sessions
  id
  venue_id
  started_at
  closed_at
  status

inventory_items
  id
  session_id
  product_id
  bottle_variant_id
  image_url
  detected_level
  estimated_volume_ml
  final_volume_ml
  product_confidence
  variant_confidence
  level_confidence
  model_version
  calibration_version
  manually_corrected
  created_at

scan_feedback
  inventory_item_id
  predicted_ml
  corrected_ml
  correction_reason
```

## 8. API response contract

Namiesto súčasného jednoduchého výsledku odporúčame vracať:

```json
{
  "product": {
    "id": "...",
    "brand": "Jameson",
    "name": "Irish Whiskey",
    "nominal_volume_ml": 700,
    "ean": "..."
  },
  "variant": {
    "id": "jameson-700-v2024",
    "confidence": 0.96
  },
  "measurement": {
    "level_normalized": 0.61,
    "estimated_volume_ml": 428,
    "fill_percent": 61.1,
    "confidence": 0.91,
    "estimated_error_ml": 18
  },
  "quality": {
    "blur": 0.04,
    "tilt_deg": 1.8,
    "glare": 0.09,
    "level_visible": true
  },
  "decision": "accept",
  "model_version": "level-v0.3.1",
  "calibration_version": "jameson-700-v2024-c2"
}
```

## 9. Navrhovaný stack

### Mobile
- React Native + Expo + TypeScript
- riadená kamera namiesto obyčajného `<input type=file>`

### Backend
- Python + FastAPI pre CV/ML inference
- Supabase/PostgreSQL pre auth, dáta a storage

### CV/ML
- PyTorch
- OpenCV
- segmentation/keypoint model
- embedding model pre variant retrieval
- ONNX export neskôr pre on-device inference

## 10. Migrácia zo súčasného prototypu

Zo súčasnej aplikácie zachovať ako funkčnú špecifikáciu:

- login/register flow,
- vytváranie a uzatváranie inventúr,
- zoznam skenov,
- história,
- export XLSX/PDF,
- admin koncept,
- duplicate-image guard.

Prepracovať:

- jednosúborový frontend,
- camera flow,
- AI endpoint,
- produkt/variant dátový model,
- confidence/reject flow,
- verziovanie modelov a kalibrácií,
- exporty pre skladové systémy,
- databázové migrácie a RLS politiky uložiť do Git repozitára.

## 11. Bezpečnosť

Frontend sa dnes silno spolieha na Supabase RLS. Všetky RLS policies a DB migrations musia byť vo verziovanom repozitári a testované.

Admin oprávnenie sa nesmie zabezpečovať iba zobrazením/skrytím tlačidla vo frontende. Každá admin operácia musí byť autorizovaná na serverovej/DB vrstve.

## 12. Nepriehľadné a problematické fľaše

Čisto obrazová metóda nebude spoľahlivá pre:

- úplne nepriehľadné fľaše,
- veľmi tmavé sklo bez viditeľnej hladiny,
- keramické/kovové obaly,
- silno obalené/sleeved fľaše.

Fallback:

- manuálne zadanie,
- váženie + známa tara,
- druhý scan / štandardizované osvetlenie.

Dlhodobo môže byť pre premium/dark bottle segment veľmi silná kombinácia vision + jednoduchá Bluetooth váha.

# Projekt 5 — Klasyfikacja Typu Pogody
 
**Autonomiczne Systemy Ekspertyzy i Eksploracji Danych**
 
Karolina Wajszczuk 198009, Sebastian Kuczera 198118
 
---
 
## 1. Cel projektu
 
Celem projektu jest zbudowanie systemu klasyfikacji warunków pogodowych opartego na danych rzeczywistych pobieranych z REST API, z wykorzystaniem infrastruktury chmurowej Amazon Web Services. System przetwarza surowe pomiary temperatury, wilgotności, ciśnienia atmosferycznego, prędkości wiatru, opadów oraz zachmurzenia z czterech stacji pomiarowych.
 
Kluczowym elementem jest transformacja surowych pomiarów w oznakowany zbiór danych: każdy rekord otrzymuje etykietę klasy pogodowej wyznaczoną przez zestaw reguł eksperckich opartych na progach fizycznych. Na tak przygotowanym zbiorze danych trenowany jest model Random Forest, którego zadaniem jest nauczenie się tych samych wzorców klasyfikacyjnych — tak aby model mógł samodzielnie klasyfikować nowe, niewidziane wcześniej pomiary bez konieczności stosowania reguł.
 
---
 
## 2. Architektura
 
Pipeline realizowany jest w całości w środowisku **Amazon SageMaker** jako notebook Jupyter. Dane surowe i przetworzone są wyraźnie rozdzielone i persystowane w **Amazon S3**.
 
```
Weather REST API
        |
Ingestion        --> s3://weather-classification-projekt/raw/raw_weather.json
        |
Processing        --> s3://weather-classification-projekt/processed/labelled_weather.csv
        |
Training          --> s3://weather-classification-projekt/splits/(X_train, X_test, y_train, y_test)
        |
Visualization      --> s3://weather-classification-projekt/reports/visual_summary.png
        |
Prediction          --> predykcja dla pojedynczego przykładu ze zbioru testowego
```
 
**Wykorzystane usługi AWS:**
 
- **Amazon S3** — przechowywanie danych na każdym etapie pipeline'u. Bucket `weather-classification-projekt` zawiera cztery foldery: `raw/`, `processed/`, `splits/`, `reports/`. Zapis i odczyt odbywa się przez bibliotekę `boto3`.
- **Amazon SageMaker** — środowisko wykonawcze notebooka Jupyter. Instancja `ml.t3.medium` zapewnia środowisko Python z dostępem do S3 przez rolę IAM `LabRole`.
---
 
## 3. Metodologia analityczna
 
**Pozyskiwanie danych** — dane pobierane są z REST API przy użyciu biblioteki `requests`. Notebook odpytuje endpoint `/weather/stations`, a następnie pobiera po 500 rekordów z każdej z czterech stacji (`GDN_01`, `GDN_02`, `GDY_01`, `SOP_01`) — łącznie 2000 pomiarów. Surowe dane zapisywane są bez modyfikacji do S3 (warstwa *raw*).
 
**Klasyfikacja regułowa** — funkcja `classify_weather` przypisuje każdemu rekordowi etykietę na podstawie hierarchicznego zestawu reguł opartych na progach fizycznych (ciśnienie, opady, wiatr, temperatura, zachmurzenie). Pierwsza pasująca reguła wygrywa. Wynik tej klasyfikacji stanowi podstawe do trenowania dla modelu ML.
 
**Trening modelu** — dataset dzielony jest na zbiór treningowy i testowy w proporcji 60/40 (`random_state=42` dla powtarzalności). Zastosowano `RandomForestClassifier` ze `scikit-learn` — zespół drzew decyzyjnych trenowanych na losowych podzbiorach danych i cech, gdzie predykcja końcowa wyznaczana jest przez głosowanie większościowe. W porównaniu do pojedynczego drzewa decyzyjnego, las losowy jest odporniejszy na przeuczenie i lepiej generalizuje na klasach rzadkich.
 
**Ewaluacja** — model oceniany jest standardowymi metrykami klasyfikacji wieloklasowej: precision, recall, F1-score i accuracy, z wizualizacją w postaci macierzy pomyłek oraz porównania F1 reguł vs model ML.
 
**Wynik:** accuracy **97.62%** na zbiorze testowym.
 
---
 
## 4. Ograniczenia i możliwe usprawnienia
 
### Ograniczenia
 
- **Dane syntetyczne API** — analiza rozkładów wartości wskazuje że API generuje dane w sposób syntetyczny, a nie na podstawie rzeczywistych pomiarów stacyjnych. Wszystkie cztery stacje wykazują niemal identyczne średnie wartości cech.
- **Klasa `cold` nieobecna w datasecie** — temperatury w badanym okresie nie spadały poniżej progu 12°C. Model nie nadaje się do klasyfikacji warunków zimowych bez dodatkowych przykładów treningowych.
- **Mała liczba przykładów `storm-risk` i `unstable`** — poniżej 100 rekordów to za mało, by model nauczył się granic decyzyjnych dla tych klas, co skutkuje niższym recall.
- **Krótki przedział czasowy** — dane obejmują około 3 dni pomiarów z czterech stacji w jednym regionie, więc model nie może być traktowany jako ogólny klasyfikator warunków pogodowych.
### Możliwe usprawnienia
 
- Pobieranie danych z dłuższego okresu, w celu uzyskania przykładów klasy `cold` oraz zwiększenia liczby przykładów klas rzadkich (`storm-risk`, `unstable`).
- Automatyczne wyznaczanie progów w regułach eksperckich na podstawie analizy rozkładu danych (np. percentyle) zamiast ręcznego dobierania wartości.
---
 
## 5. Wnioski
 
Zbudowany pipeline w pełni realizuje założony cel — przekształca surowe pomiary pogodowe w działający klasyfikator wieloklasowy, osiągając wysoką ogólną skuteczność 97.62%. Dla klas dobrze reprezentowanych w danych, powyżej 100 rekordów, model osiąga F1 ≥ 0.96 praktycznie odtwarzając logikę reguł eksperckich bez ich bezpośredniego stosowania.
 
Jednocześnie wyniki potwierdzają znaczenie liczebności danych treningowych — klasy rzadkie `storm-risk`, `unstable` wypadają zauważalnie słabiej, F1 odpowiednio 0.60 i 0.81, co nie wynika z ograniczeń algorytmu, lecz z niedoboru przykładów. Projekt pokazuje więc nie tylko skuteczność podejścia rule-based-to-ML, ale i jego praktyczne ograniczenie przy nierównomiernym rozkładzie klas.

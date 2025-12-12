# PIDML_task
Repozytorium na zadanie wstępne z ML związane z udziałem w pracach dotyczących CERN


# Przewidywanie Śmiertelności Pacjentów

##  Cel Projektu
Celem projektu było zbudowanie modelu klasyfikacji binarnej z wykorzystaniem algorytmu regresji logistycznej, przewidującego ryzyko zgonu pacjentów szpitalnych na podstawie zbioru danych **Support2**.

Głównym założeniem było stworzenie modelu przewidującego czy pacjent przeżyje, opartego wyłącznie na danych dostępnych w trakcie pobytu pacjenta w szpitalu, z wyeliminowaniem wycieków danych (Data Leakage).

##  Zastosowana Metodologia

### 1. Eksploracja i Czyszczenie Danych
Na etapie wstępnej analizy zidentyfikowano i naprawiono różne problemy z jakością danych:
* **Korekta błędów pomiarowych:** Wartości zerowe w parametrach życiowych (np. `meanbp`, `glucose`, `hrt`), które są niemożliwe z medycznego punktu widzenia, zostały potraktowane jako braki danych.
* **Imputacja Ekspercka:** Zamiast standardowej średniej, braki w wybranych parametrach (np. albumina, kreatynina, bilirubina) uzupełniono wartościami bazowymi z **HBiostat Repository (Prof. Frank Harrell)**. Pozwoliło to na zachowanie spójności logicznej danych.

### 2. Inżynieria Cech (Feature Engineering)
W celu poprawy skuteczności modelu wprowadzono transformacje oparte na wiedzy medycznej:
* **Shock Index (Wskaźnik Wstrząsu):** Wyliczono nową zmienną (`tętno / średnie ciśnienie`). Wartości powyżej 1.0 wskazują na niestabilność hemodynamiczną i podwyższone ryzyko zgonu.
* **Transformacja Logarytmiczna:** Dla zmiennych o silnie skośnym rozkładzie (skewness > 1.0), takich jak `bilirubina`, `kreatynina` czy `leukocyty`, zastosowano funkcję `np.log1p`. Pozwoliło to ograniczyć wpływ wartości odstających na model liniowy.

### 3. Eliminacja Wycieków Danych (Data Leakage)
Kluczowym elementem projektu było usunięcie zmiennych, które nie są dostępne w momencie przyjęcia pacjenta:
* **Wycieki bezpośrednie:** Usunięto czas przeżycia (`d.time`) oraz informację o zgonie w szpitalu (`hospdead`).
* **Ukryty wyciek (`sfdm2`):** W trakcie tworzenia modelu, a końcowym wykresie zidentyfikowano, że kategoria `<2 mo. follow-up` w tej zmiennej jest równoważna z szybkim zgonem pacjenta. Jej usunięcie było krytyczne dla rzetelności modelu.
* **Dane z przyszłości:** Usunięto koszty leczenia i długość pobytu (znane dopiero po wypisie).
* **Subiektywne oceny:** Usunięto prognozy innych modeli, aby model był w pełni niezależny (nie znamy działania tamtego modelu).

### 4. Modelowanie
* **Algorytm:** Regresja Logistyczna (Logistic Regression).
* **Optymalizacja:** Wykorzystano `GridSearchCV` z 10-krotną walidacją krzyżową (CV) do optymalizacji parametrów modelu.
* **Nierównowaga klas:** Zastosowano ważenie klas (`class_weight='balanced'`), aby model skuteczniej wykrywał przypadki zgonu (klasa mniejszościowa).

## Wyniki

Model został przetestowany na powiększonym zbiorze testowym (30% danych, ~2700 pacjentów), aby zapewnić wysoką wiarygodność statystyczną wyników.

* **ROC AUC:** ~0.85 (Bardzo dobra zdolność rozróżniania klas pacjentów)
* **Dokładność (Accuracy):** ~75%
* **Precyzja (Dla klasy 'Zmarł'):** ~0.89 (Mało fałszywych alarmów, że pacjent może umrzeć)

**Wnioski:**
Uzyskana wysoka precyzja oznacza, że model generuje bardzo mało fałszywych alarmów przy identyfikacji pacjentów w stanie zagrożenia życia, najczęściej myli się oznaczając pacjenta jako 1 (zgon), kiedy w rzeczywistości przeżył. Analiza wag modelu daje różne wnioski:
* Sepsa u analizowanych pacjentów wiąże się z większą szansą przeżycia - mimo, że jest to śmiertelna i ciężka choroba, ma większą przeżywalność niż inne dostępne
opcje (np. rak płuc), wynik ten tłumaczy fakt, że analizowano tylko pacjentów w ciężkim stanie
* Decyzja o nieresuscytowaniu (DNR) jasno wiąże się z wiekszą szansą na zgon pacjenta
* Mniejsza zamożność pacjenta wiąże się z większym ryzykiem zgonu

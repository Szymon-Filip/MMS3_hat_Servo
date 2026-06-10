# Hat 4-kanałowy sterownik serw

## Sekcja 1: Dokumentacja Hat'a

### Krótki opis projektu
Hat to 4-kanałowy sterownik serw, bazujący na  generatorze PWM **PCA9685PW** (komunikacja I2C). Zasialnie serw jest z XT60

### Zgodność ze standardem ChainBus

* ✅ Używa złącza ChainBus, nie zmienia jego miejsca ani pinoutu.
* ✅ Używa wyłącznie interfejsów I2C, SPI lub UART i nie inicjuje samodzielnie nowych transmisji (Nie jest master'em I2C albo SPI).
* ✅ Spełnia wymagania mechaniczne standardu (wymiary PCB, rozstaw otworów).
* ✅ Pobiera maksymalny prąd zgodny z ilością na jednego hat'a
* ✅ Obsługuje napięcie wejściowe BRD_VIN do wartości 48V.

### Komunikacja i adresowanie

#### Adresacja I2C
Komunikacja z PCA9685PW odbywa się za pomocą magistrali I2C. Wszystkie linie adresowe układu PCA9685PW (A0, A1, A2, A3, A4, A5) są podłączone do napięcia 5V.

| Układ (IC)    | Funkcja                          | Adres I2C (7-bit) |
| :------------ | :------------------------------- | :---------------: |
| **PCA9685PW** | 16-kanałowy generator PWM 12-bit | `1111111b` (0x7F) |

---

### Przypisanie kanałów generatora PCA9685

Układ PCA9685 steruje bezpośrednio czterema wyjściami serw oraz dwoma debug diodami LED:

| Wyjście PCA9685 | Powiązany element | Typ złącza  | Opis                              |
| :-------------: | :---------------- | :---------- | :-------------------------------- |
|    **LED0**     | Servo 1           | **J10**     | Kanał sterujący serwomechanizmu 1 |
|    **LED1**     | Servo 2           | **J9**      | Kanał sterujący serwomechanizmu 2 |
|    **LED2**     | Servo 3           | **J8**      | Kanał sterujący serwomechanizmu 3 |
|    **LED3**     | Servo 4           | **J7**      | Kanał sterujący serwomechanizmu 4 |
|    **LED14**    | Debug LED         | *Na płycie* | Dioda LED do debugowania          |
|    **LED15**    | Debug LED         | *Na płycie* | Dioda LED do debugowania          |

---

### Pinout złączy serwomechanizmów (J7 – J10)

Każde złącze wyjściowe (J7, J8, J9, J10) posiada 3 piny w następującym układzie:

| Pin   | Nazwa sygnału | Poziom napięcia    | Opis                                       |
| :---- | :------------ | :----------------- | :----------------------------------------- |
| **1** | `GND`         | 0 V                | Wspólna masa układu                        |
| **2** | `SERVO` (PWM) | 5 V                | Sygnał sterujący PWM z układu PCA9685      |
| **3** | `VCC`         | `BRD_VIN` (z XT60) | Napięcie zasilające silnik serwomechanizmu |

> **⚠️ UWAGA DOTYCZĄCA BEZPIECZEŃSTWA (PINOUT):**
> Układ wyprowadzeń złączy na tym module to **[1: GND, 2: Sygnał, 3: VCC]**. Jest on odmienny od klasycznego modelarskiego standardu wtyczek serwomechanizmów (gdzie pin zasilania VCC zazwyczaj znajduje się w środku wtyku).
>
> Przed podłączeniem serwomechanizmu należy bezwzględnie zweryfikować kolejność przewodów w wiązce serwa. Błędne podłączenie może skutkować uszkodzeniem serwa lub obwodów zasilania modułu.

---

### Szczegółowy opis techniczny

#### Zasilanie serwomechanizmów (`VCC`)
Prąd zasilania serwomechanizmów nie obciąża wewnętrznych linii zasilających magistrali ChainBus. Napięcie doprowadzane jest bezpośrednio ze złącza wysokoprądowego **XT60** na pin 3 każdego złącza serwa. Zasilanie z XT-60 od razu idzie na szyne VIN ChainBus'a. Jeśli potrzebujesz zasilić serwa innym napięciem niż takim samym jak VIN ChainBus'a to odlutuj D1 (tą diode po lewej stronie do której idzie taka duża ścieżka)

#### Sterowanie sygnałem PWM
PWM jest na 5V, ale szyna I2C na 3V3 więc wszystko śmiga jak powinno


---

### Gotowe arkusze hierarchiczne
W projekcie wydzielono i zastosowano następujące arkusze hierarchiczne:
* **Servo_controller** – Schemat generatora PWM PCA9685 z konwerterem I2C 3v3 <-> 5V i zworkami do wybierania adresu I2C

---

## Sekcja 2: Specyfikacja standardu ChainBus

### Architektura i łączenie modułów
Standard ChainBus umożliwia modułowe łączenie hatów. Na jednym MMS3 można zamontować pionowo **do 8 hat'ów**. Połączenie realizowane jest poprzez wpięcie złącza męskiego kolejnego hat'a w złącze żeńskie poprzedniego.

### Komunikacja i sterowanie
Magistrala ChainBus jest w pełni cyfrowa. Płyta główna nie steruje bezpośrednio sygnałami ogólnego przeznaczenia (GPIO) na poszczególnych hat'ach. Wszelkie operacje (np. generowanie sygnałów PWM, obsługa diod) muszą być realizowane przez dedykowane układy scalone komunikujące się przez interfejsy systemowe.

Wybór aktywnego modułu realizowany jest przez układ przełącznika magistrali (bus switch) na płycie głównej. Dzięki temu linie I2C, SPI i UART są niezależne dla każdego hat'a (brak konfliktów adresów I2C między różnymi hatami).
* **Identyfikacja:** Każdy moduł powinien posiadać pamięć EEPROM na magistrali I2C w celu identyfikacji płyty przez system - układ M24C64-W skonfigurowany na adres `1010000` przy liniach adresowych A0, A1, A2 zwartych do masy.

### Zasilanie
Złącze ChainBus dostarcza następujące linie zasilania:

| Magistrala zasilania | Napięcie znamionowe | Maksymalny prąd (łączny dla 8 hatów) | Szacowany prąd na jeden hat |
| :------------------- | :-----------------: | :----------------------------------: | :-------------------------: |
| **5V**               |        5.0 V        |                1.0 A                 |           125 mA            |
| **12V stby**         |       12.0 V        |                0.5 A                 |            65 mA            |
| **BRD_VIN**          |   12.0 V – 48.0 V   |                1.5 A                 |           185 mA            |

*   W przypadku zapotrzebowania na wyższą moc (jak w przypadku tego modułu sterowania serwami), stosuje się dodatkowe złącze zasilania XT60 (obciążalność do ok. 60 A).

---

## Sekcja 3: Licencja, linki i tagi

### Licencjonowanie projektu

*   **PCB:** [CERN-OHL-W](https://ohwr.org/project/cernohl/wikis/Documents/CERN-OHL-version-2) - Umożliwia modyfikacje i sprzedaż pod warunkiem zachowania informacji o oryginalnym autorze. Zmiany muszą być open source.
*   **Software:** [MIT License](https://opensource.org/licenses/MIT) - Umożliwia modyfikacje i sprzedaż komercyjną pod warunkiem zachowania informacji o autorze (zmiany nie muszą być udostępniane jako open source).

### Tagi projektu
#chainbus #MMS3 #ModuCard
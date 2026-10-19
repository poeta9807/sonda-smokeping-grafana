# Sonda sieciowa w stylu Smokepinga z wykresami w Grafanie

Mała maszyna wirtualna, która co sekundę pinguje wybrane adresy (router, DNS-y, serwisy, serwery gier) i zapisuje
każdy wynik. W Grafanie widać potem nie tylko średni ping, ale też jak bardzo pingi skaczą i ile pakietów ginie.
Po kilku dniach takiej historii łatwo odpowiedzieć na pytania typu „czy wczoraj wieczorem coś się działo z łączem”
albo „czy lagi w grze to wina sieci, czy serwera”.

Całość stoi na trzech programach uruchamianych w Dockerze:

- **smokeping_prober** ([SuperQ/smokeping_prober](https://github.com/SuperQ/smokeping_prober)) wysyła pingi
  i liczy, ile odpowiedzi przyszło w jakim czasie,
- **Prometheus** co 15 sekund odczytuje te liczniki i przechowuje je (tutaj przez 90 dni),
- **Grafana** rysuje z tego wykresy i może wysyłać alerty.

Opcjonalnie obok działa [Haze](https://github.com/consi/haze), czyli osobny program z własną stroną WWW,
który wygląda podobnie do klasycznego Smokepinga. Haze nie potrafi oddawać danych do Grafany (sprawdzane na
wersji 0.6, październik 2026), więc traktuj go jako dodatkowy podgląd, a nie część głównego zestawu. Jeśli go
nie potrzebujesz, usuń sekcję `haze` z `docker-compose.yml`.

```
 pingowane cele (router, 1.1.1.1, 8.8.8.8, ...)
            ^
            |  ping co 1 s
 smokeping_prober  (port 9374)
            ^
            |  odczyt co 15 s
 Prometheus        (port 9090)
            ^
            |  zapytania
 Grafana           (port 3000)
```

## Gdzie postawić sondę

Sonda mierzy sieć z miejsca, w którym stoi. Jeśli chcesz wiedzieć, jak łącze działa dla zwykłego użytkownika,
postaw ją tam, gdzie stałby jego komputer: za routerem, w zwykłej sieci LAN. Kilka zasad, które oszczędzą
nerwów:

- podłącz ją kablem. Wi-Fi samo z siebie dorzuca skoki pingu i potem nie wiadomo, co jest winne,
- ruch nie może iść przez VPN ani proxy, bo zmierzysz VPN, a nie łącze,
- jeśli to VM na Proxmoxie lub innym hypervisorze, host nie powinien być zajechany pod korek. Przeciążony host
  daje skoki pingu na wszystkich celach naraz, co łatwo pomylić z awarią łącza.

## Wymagania

- Debian 12 albo Ubuntu 24.04 LTS (zwykła instalacja serwerowa, bez środowiska graficznego),
- 2 rdzenie, 2 GB RAM, 20-30 GB dysku. Prometheus z kilkudziesięcioma celami i 90 dniami historii zajmuje
  kilka GB,
- stały adres IP w sieci lokalnej (statyczny albo rezerwacja DHCP na routerze),
- działająca synchronizacja czasu. Sprawdzisz ją poleceniem `timedatectl`; w wyniku powinno być
  `System clock synchronized: yes`. Bez tego wykresy będą przesunięte w czasie.

## Instalacja

### 1. Docker

```bash
sudo apt update
sudo apt -y install curl ca-certificates git
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Po ostatnim poleceniu wyloguj się i zaloguj ponownie, żeby można było używać Dockera bez `sudo`.
Sprawdź, czy działa:

```bash
docker compose version
```

### 2. Pobranie repozytorium

```bash
sudo git clone https://github.com/poeta9807/sonda-smokeping-grafana.git /opt/sonda
sudo chown -R $USER: /opt/sonda
cd /opt/sonda
```

Wszystkie dalsze polecenia wykonuj w katalogu `/opt/sonda`.

### 3. Hasło do Grafany

Hasło administratora Grafany jest w pliku `.env`, którego nie ma w repozytorium (i nie powinno być). Utwórz go
ze wzoru i wpisz losowe hasło:

```bash
cp .env.example .env
sed -i "s/zmien-mnie/$(openssl rand -base64 18 | tr -d '/+=')/" .env
chmod 600 .env
cat .env
```

Zapisz sobie to hasło, przyda się przy pierwszym logowaniu.

### 4. Lista celów

Cele są w pliku `smokeping_prober.yml`. Na start jest tam router `192.168.1.1`, trzy publiczne DNS-y i dwa
portale. Adres routera prawie na pewno trzeba zmienić. Swój znajdziesz poleceniem:

```bash
ip route | grep default
```

Adres po słowie `via` to Twój router.

Przy wyborze celów warto myśleć warstwami, od najbliższego do najdalszego:

1. **router** w sieci lokalnej - powinien mieć ping poniżej 1 ms i zero strat. Jeśli tu są problemy, wina leży
   w LAN-ie albo na samej maszynie, a nie u operatora,
2. **pierwszy router operatora** - znajdziesz go poleceniem `traceroute 1.1.1.1` (zwykle druga lub trzecia
   pozycja na liście). Pokazuje, czy problem jest na łączu do operatora,
3. **publiczne DNS-y** (1.1.1.1, 8.8.8.8, 9.9.9.9) - stoją blisko i prawie nigdy nie padają, więc dobrze
   pokazują stan samego internetu,
4. **serwisy i serwery gier**, z których faktycznie korzystasz.

Dzięki takiemu podziałowi od razu widać, gdzie leży problem. Jeśli pingi giną jednocześnie do wszystkich celów,
problem jest blisko sondy. Jeśli tylko do jednego, problem jest po stronie tego celu albo na trasie do niego.

Każdy dopisany adres pojawi się w Grafanie jako osobna seria. Uwaga na serwery, które nie odpowiadają na ping:
część firm (np. niektóre serwery gier) blokuje ICMP. Zanim dopiszesz adres, sprawdź go z sondy zwykłym
`ping -c 5 adres`.

### 5. Uruchomienie

```bash
docker compose up -d
docker compose ps
```

Wszystkie kontenery powinny mieć status `running` (albo `Up`). Sprawdź, czy prober faktycznie pinguje:

```bash
curl -s localhost:9374/metrics | grep smokeping_requests_total
```

Powinno pojawić się po jednej linii na każdy cel, z liczbą rosnącą przy każdym kolejnym wywołaniu. Jeśli nic nie
ma, zajrzyj do logów: `docker compose logs smokeping`.

Na koniec sprawdź, czy Prometheus zbiera dane. Prometheus nasłuchuje tylko lokalnie, więc z własnego komputera
otwórz tunel SSH:

```bash
ssh -L 9090:localhost:9090 uzytkownik@adres-sondy
```

i wejdź na http://localhost:9090, menu **Status → Targets**. Pozycja `smokeping` ma mieć stan **UP**.

## Pierwsze logowanie

- **Grafana:** `http://adres-sondy:3000`, użytkownik `admin`, hasło z pliku `.env`. Źródło danych Prometheus
  jest już skonfigurowane, nic nie trzeba dodawać ręcznie.
- **Haze** (jeśli zostawiony): `http://adres-sondy:4420`, użytkownik `admin`. Hasło Haze generuje sam przy
  pierwszym starcie i wypisuje w logach:

  ```bash
  docker compose logs haze | grep -i password
  ```

  Cele do Haze dodaje się w jego interfejsie (niezależnie od pliku `smokeping_prober.yml`).

## Wykresy w Grafanie

### Co zbiera prober

Dla każdego celu (etykieta `host`) są dwie metryki:

- `smokeping_requests_total` - ile pingów wysłano,
- `smokeping_response_duration_seconds` - ile odpowiedzi przyszło i w jakim czasie. To histogram, czyli
  prober nie zapisuje każdego pingu osobno, tylko liczy, ile odpowiedzi zmieściło się w przedziałach
  (do 0,5 ms, do 1 ms, do 2 ms i tak dalej). Z tego da się policzyć średnią, medianę, rozrzut i narysować heatmapę.

Brak odpowiedzi nie trafia do histogramu, więc utrata pakietów to po prostu różnica między liczbą wysłanych
pingów a liczbą odpowiedzi.

### Zapytania

Średni ping z ostatniej minuty (jednostka w Grafanie: *Time → seconds (s)*):

```
sum by (host) (rate(smokeping_response_duration_seconds_sum[1m]))
/
sum by (host) (rate(smokeping_response_duration_seconds_count[1m]))
```

Mediana, czyli wartość, poniżej której mieści się połowa pingów. Mniej podatna na pojedyncze skoki niż średnia:

```
histogram_quantile(0.5, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[1m])))
```

Rozrzut pingów (różnica między 90. a 10. percentylem). Im większa liczba, tym bardziej ping skacze, co w grach
i rozmowach głosowych przeszkadza bardziej niż sam wysoki ping:

```
histogram_quantile(0.9, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[5m])))
-
histogram_quantile(0.1, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[5m])))
```

Utrata pakietów (jednostka: *Misc → Percent (0.0-1.0)*):

```
1 - (
  sum by (host) (rate(smokeping_response_duration_seconds_count[1m]))
  /
  sum by (host) (rate(smokeping_requests_total[1m]))
)
```

Przy pingu co sekundę jeden zgubiony pakiet w ciągu minuty to około 1,7% utraty. Pojedyncze takie słupki co
jakiś czas są normalne. Niepokojące są utraty powyżej kilku procent albo trwające dłużej niż kilka minut.

### Budowa dashboardu krok po kroku

1. **Dashboards → New → New dashboard**, potem ikona zębatki (ustawienia dashboardu) → **Variables → Add
   variable**:
   - Type: *Query*, Name: `host`,
   - Query: `label_values(smokeping_requests_total, host)`,
   - zaznacz *Multi-value* i *Include All option*, zapisz.
2. **Add → Visualization**, typ *Time series*. Wpisz zapytanie o średni ping, ale zamiast wszystkich hostów
   użyj zmiennej: w zapytaniu dopisz `{host=~"$host"}` do obu metryk. Ustaw jednostkę *seconds (s)*.
3. W tym samym panelu dodaj drugie zapytanie (przycisk *+ Query*) z utratą pakietów. Następnie w zakładce
   *Overrides* dodaj regułę *Fields returned by query* → B i ustaw:
   - *Axis placement*: Right,
   - *Graph styles → Style*: Bars,
   - *Unit*: Percent (0.0-1.0),
   - *Axis soft max*: 0.05.

   Dzięki temu ping jest linią na lewej osi, a utrata słupkami na prawej, a skala 0-5% sprawia, że nawet małe
   straty są widoczne.
4. W opcjach panelu, w sekcji *Repeat options*, wybierz zmienną `host`. Grafana zrobi wtedy osobny wykres
   dla każdego celu. To dużo czytelniejsze niż kilkanaście linii na jednym wykresie.
5. Pod spodem dodaj heatmapę (opis niżej).
6. Zapisz dashboard. Warto też wyeksportować go do pliku (**Share → Export → Save to file**) i wrzucić JSON
   do repozytorium, żeby nie przepadł przy reinstalacji.

Jeśli wolisz zacząć od gotowca, na stronie
[grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards/) wyszukaj „smokeping_prober”
i zaimportuj jeden z dashboardów, a potem dostosuj go do siebie.

### Heatmapa, czyli „dym”

W klasycznym Smokepingu wokół linii pingu rysowana jest szara chmura, nazywana dymem. Pokazuje, jak bardzo
rozrzucone były pojedyncze pingi. W Grafanie to samo robi panel typu heatmapa:

- oś pozioma to czas, pionowa to ping,
- każdy kwadracik to przedział pingu w danym momencie,
- im ciemniejszy kolor, tym więcej pingów w tym przedziale.

Jak ją zrobić:

1. **Add → Visualization**, typ *Heatmap*.
2. Zapytanie (dla jednego wybranego celu):

   ```
   sum by (le) (increase(smokeping_response_duration_seconds_bucket{host="$host"}[$__rate_interval]))
   ```

3. Pod zapytaniem rozwiń *Options* i ustaw *Format* na **Heatmap**.
4. W opcjach panelu ustaw *Calculate from data* na **No** (dane są już podzielone na przedziały), a jednostkę
   osi Y na *seconds (s)*.

Jak czytać:

- wąski ciemny pas na stałej wysokości oznacza stabilne łącze,
- szeroka, rozmyta chmura oznacza, że ping skacze,
- pas, który nagle przesuwa się wyżej lub niżej i tak zostaje, oznacza zwykle zmianę trasy po stronie operatora,
- pojedyncze kropki wysoko nad pasem to rzadkie, mocno spóźnione pakiety.

Heatmapa pokazuje rzeczy, które średnia ukrywa. Dziewięćdziesiąt dziewięć pingów po 10 ms i jeden po 500 ms
dają średnią około 15 ms, co na zwykłym wykresie wygląda całkiem dobrze. Na heatmapie ten jeden pakiet widać od razu.

## Alerty

Alerty ustawia się w Grafanie w **Alerting → Alert rules**, a miejsce, gdzie mają przychodzić powiadomienia
(e-mail, Discord, Teams, Slack), w **Alerting → Contact points**. Na początek wystarczą trzy:

- **utrata pakietów** powyżej 5% przez 5 minut,
- **ping** wyraźnie wyższy niż zwykle (np. dwa razy) przez 10 minut,
- **sonda nie zbiera danych**, zapytanie `absent(smokeping_requests_total)`. Ten alert jest ważny, bo bez
  niego awaria samej sondy wygląda jak cisza, czyli „wszystko w porządku”.

Adresy webhooków trzymaj tylko w Grafanie, nie wpisuj ich do plików w repozytorium.

## Codzienna obsługa

- **Dodanie celu:** dopisz adres w `smokeping_prober.yml` i uruchom `docker compose restart smokeping`.
- **Aktualizacja:** `docker compose pull && docker compose up -d`. Obrazy są ustawione na `latest`, co jest
  wygodne na start. Gdy wszystko już działa, warto w `docker-compose.yml` wpisać konkretne numery wersji
  (znajdziesz je w zakładkach *Releases* poszczególnych projektów), żeby aktualizacja nie przyszła sama
  w najmniej odpowiednim momencie.
- **Kopia zapasowa:** dane siedzą w wolumenach Dockera (`docker volume ls`). Najcenniejsza jest konfiguracja,
  czyli to repozytorium i wyeksportowany JSON dashboardu. Historię pomiarów zwykle da się odżałować.
- **Miejsce na dysku:** `docker system df`. Jeśli brakuje miejsca, zmniejsz retencję w `docker-compose.yml`
  (`--storage.tsdb.retention.time=90d`, np. na `30d`).

## Bezpieczeństwo

- Grafany i Haze nie wystawiaj do internetu. Dostęp z sieci lokalnej, przez VPN albo przez tunel SSH
  w zupełności wystarczy.
- Docker sam otwiera porty wypisane w `ports:` i omija przy tym zaporę `ufw`. Jeśli coś ma być dostępne tylko
  z samej sondy, wpisz port z adresem `127.0.0.1:` na początku (tak już jest ustawiony Prometheus i prober).
- Zmień hasło admina Grafany, jeśli korzystało z niego więcej osób, i załóż osobne konta zamiast dzielić się
  jednym.
- Plik `.env` nie trafia do repozytorium (jest w `.gitignore`) i niech tak zostanie.

Sonda mierzy tylko czas odpowiedzi i utratę pakietów do wskazanych adresów. Nie widzi ruchu innych urządzeń
w sieci i nie zbiera o nich żadnych danych.

## Gdy coś nie działa

**W logach probera jest `operation not permitted` albo `socket: permission denied`.**
Prober nie ma uprawnień do wysyłania pingów. Sprawdź, czy w `docker-compose.yml` przy usłudze `smokeping` są
linie `user: "0:0"` i `cap_add: [NET_RAW]`.

**Jeden cel cały czas ma 100% utraty.**
Ten serwer najpewniej nie odpowiada na ping. Sprawdź `ping -c 5 adres` z sondy. Jeśli też nie odpowiada,
wybierz inny adres albo mierz go inaczej (np. czas połączenia TCP przez blackbox_exportera, patrz niżej).

**W Prometheusie `smokeping` ma stan DOWN.**
Prometheus nie może połączyć się z proberem. W `prometheus/prometheus.yml` musi być `smokeping:9374`
(nazwa usługi z `docker-compose.yml`, nie `localhost`). Sprawdź też, czy kontener probera w ogóle działa.

**Wykresy są przesunięte o godzinę albo kilka minut.**
Zły czas na maszynie albo przeglądarka i serwer w różnych strefach czasowych. Sprawdź `timedatectl`
i ustawienie strefy czasowej w Grafanie (w profilu użytkownika).

**Skoki pingu lub utrata na wszystkich celach naraz, łącznie z routerem.**
Problem jest na samej sondzie albo w sieci lokalnej: przeciążony host, Wi-Fi, uszkodzony kabel, przeciążony
switch. Łącze do internetu nie ma tu nic do rzeczy, skoro problem widać już na routerze.

**Skoki na wszystkich celach poza routerem.**
To już łącze albo sieć operatora. Porównaj z pingiem do pierwszego routera operatora, żeby zawęzić miejsce.

## Co można dołożyć później

- **Pomiar DNS, stron WWW i portów TCP:** [blackbox_exporter](https://github.com/prometheus/blackbox_exporter)
  dokłada się jako kolejny kontener w `docker-compose.yml` i kolejną sekcję w `prometheus.yml`. Przydaje się,
  gdy ping działa, a strona mimo to ładuje się wolno.
- **Historia tras:** Haze zapisuje trasy do celów, co pomaga zrozumieć, dlaczego ping nagle wzrósł o kilka ms.
- **Klasyczny Smokeping:** jest w repozytoriach Debiana jako pakiet `smokeping`. Ma własne wykresy i stronę WWW,
  ale do Grafany się go nie podepnie, więc ma sens tylko wtedy, gdy Grafana nie jest potrzebna.

## Pliki w repozytorium

| Plik | Do czego służy |
|---|---|
| `docker-compose.yml` | definicja wszystkich kontenerów: prober, Prometheus, Grafana i opcjonalnie Haze |
| `smokeping_prober.yml` | lista pingowanych adresów |
| `prometheus/prometheus.yml` | skąd i jak często Prometheus pobiera dane |
| `grafana/provisioning/datasources/prometheus.yml` | automatyczne podłączenie Prometheusa w Grafanie |
| `.env.example` | wzór pliku z hasłem do Grafany |

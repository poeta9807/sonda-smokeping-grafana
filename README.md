# Sonda sieciowa: Smokeping-style + Prometheus + Grafana

Sonda na maszynie wirtualnej, która co sekundę pinguje wybrane cele i rysuje w Grafanie wykresy w stylu Smokepinga:
średnie opóźnienie, „dym” (rozrzut pomiarów, heatmapa) i utrata pakietów. Opcjonalnie obok działa
[Haze](https://github.com/consi/haze) z własnym interfejsem.

> **Haze a Grafana:** Haze ma własny interfejs WWW (:4420), ale nie eksportuje metryk do Prometheusa/Grafany
> (stan na v0.6.x, 10.2026). Dlatego wykresy w Grafanie robi
> [smokeping_prober](https://github.com/SuperQ/smokeping_prober), a Haze jest tylko dodatkiem.

## Architektura

```
cele (brama, 1.1.1.1, 8.8.8.8, serwisy…)
        ▲  ping ICMP co 1 s
[smokeping_prober :9374]  → histogram opóźnień + licznik wysłanych
        ▲  scrape co 15 s
[Prometheus :9090]        → dane (90 dni)
        ▲  PromQL
[Grafana :3000]           → dashboardy, heatmapa, alerty

[Haze :4420]              → opcjonalny, niezależny podgląd
```

Sondę stawiamy tam, skąd chcemy mierzyć jakość z perspektywy użytkownika – za zwykłym routerem/NAT-em.

## Wymagania

- VM: Debian 12 lub Ubuntu 24.04 LTS, 2 vCPU, 2 GB RAM, 20–30 GB dysku.
- Bezpośredni dostęp do internetu przez badane łącze (bez VPN/proxy), stały IP w LAN, najlepiej kabel, nie Wi-Fi.
- Włączony NTP: `timedatectl` → `System clock synchronized: yes`.

## Instalacja

```bash
# 1. Docker
sudo apt update && sudo apt -y install curl ca-certificates git
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER      # wyloguj się i zaloguj ponownie

# 2. Repozytorium
sudo git clone <adres-tego-repo> /opt/sonda
sudo chown -R $USER: /opt/sonda && cd /opt/sonda

# 3. Hasło do Grafany
cp .env.example .env
sed -i "s/zmien-mnie/$(openssl rand -base64 18 | tr -d '/+=')/" .env
chmod 600 .env

# 4. Cele: edytuj smokeping_prober.yml (podmień 192.168.1.1 na swoją bramę)

# 5. Start
docker compose up -d
docker compose ps                                     # wszystko "running"
curl -s localhost:9374/metrics | grep -c smokeping_   # > 0 = prober działa
```

Dostęp:

- **Grafana:** `http://IP-VM:3000`, login `admin`, hasło z `.env`. Źródło danych Prometheus jest już podpięte.
- **Prometheus:** `ssh -L 9090:localhost:9090 user@vm` → http://localhost:9090 → *Status → Targets* – job `smokeping` ma być **UP**.
- **Haze:** `http://IP-VM:4420`, login `admin`, hasło z `docker compose logs haze | grep -i password`; cele dodaje się w interfejsie.

Nowy cel w Grafanie = dopisz host w `smokeping_prober.yml` → `docker compose restart smokeping`.

Wersje obrazów są ustawione na `latest`. Po pierwszym udanym uruchomieniu przypnij konkretne wersje w `docker-compose.yml`.

## Pliki

| Plik | Co robi |
|---|---|
| `docker-compose.yml` | smokeping_prober, Prometheus, Grafana, Haze (opcjonalny – można usunąć sekcję) |
| `smokeping_prober.yml` | lista pingowanych celów |
| `prometheus/prometheus.yml` | co i jak często zbiera Prometheus |
| `grafana/provisioning/datasources/prometheus.yml` | automatyczne podpięcie Prometheusa w Grafanie |
| `.env.example` | wzór pliku z hasłem (właściwy `.env` nie trafia do gita) |

## Wykresy w Grafanie

Metryki: `smokeping_requests_total` (wysłane pingi) i `smokeping_response_duration_seconds` (histogram: `_bucket`, `_sum`, `_count`), etykieta `host`.

| Wykres | PromQL | Jednostka |
|---|---|---|
| Średnie opóźnienie | `sum by (host) (rate(smokeping_response_duration_seconds_sum[1m])) / sum by (host) (rate(smokeping_response_duration_seconds_count[1m]))` | seconds (s) |
| Mediana | `histogram_quantile(0.5, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[1m])))` | seconds (s) |
| Rozrzut p90–p10 | `histogram_quantile(0.9, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[5m]))) - histogram_quantile(0.1, sum by (le, host) (rate(smokeping_response_duration_seconds_bucket[5m])))` | seconds (s) |
| Utrata pakietów | `1 - sum by (host) (rate(smokeping_response_duration_seconds_count[1m])) / sum by (host) (rate(smokeping_requests_total[1m]))` | Percent (0.0–1.0) |

### Heatmapa „dymu”

1. Nowy panel → typ **Heatmap**.
2. Zapytanie: `sum by (le) (increase(smokeping_response_duration_seconds_bucket{host="$host"}[$__rate_interval]))`
3. Opcje zapytania: **Format = Heatmap**.
4. Panel: *Calculate from data* = **No**, oś Y w sekundach.

Wąski ciemny pas = stabilne łącze, szeroka chmura = jitter.

### Układ dashboardu

- Zmienna `host`: *Variables → Query* `label_values(smokeping_requests_total, host)`, *Multi-value* + *Include All*.
- Panel **Time series**: opóźnienie jako linia, utrata jako słupki na prawej osi
  (*Overrides → Axis placement: Right, Draw style: Bars*).
- **Jeden panel na cel** (*Repeat by variable* = `host`) zamiast wszystkich linii na jednym wykresie.
- Heatmapa dla wybranego celu pod spodem.
- Gotowce: na grafana.com/grafana/dashboards wyszukaj „smokeping_prober”.
- Gotowy dashboard: *Share → Export → Save to file* i commit JSON-a do repo (kopia zapasowa).

## Rozszerzenia

- **Alerty** (*Alerting → Alert rules*): utrata > 5% przez 5 min; opóźnienie > 2× normalne przez 10 min;
  `absent(smokeping_requests_total)` = sonda nie działa. Webhooki (Discord/Teams/Slack) trzymaj tylko w Grafanie.
- **DNS / HTTP / TCP:** dodaj `prom/blackbox-exporter` jako kolejny kontener i job w `prometheus.yml`.
- **Historia tras:** Haze ma wbudowaną historię tras – przydatne, gdy opóźnienie skacze o kilka ms.
- **Klasyczny Smokeping** (pakiet Debiana): własne wykresy RRD, ale bez prostego podpięcia do Grafany.

## Bezpieczeństwo

- Grafany i Haze nie wystawiaj do internetu – dostęp z LAN, VPN lub tunel SSH.
- Prometheus i prober słuchają tylko na `127.0.0.1`.
- Hasła tylko w `.env` (poza gitem) i w Grafanie.
- Sonda mierzy tylko opóźnienie i utratę do publicznych adresów – nie zbiera danych o użytkownikach ani ruchu.

## Typowe problemy

| Objaw | Przyczyna | Rozwiązanie |
|---|---|---|
| `operation not permitted` w logach probera | brak uprawnień do ICMP | sprawdź `cap_add: [NET_RAW]` i `user: "0:0"` |
| Cel ciągle 100% utraty | cel blokuje ping | `ping` z VM; wybierz inny cel lub mierz TCP (blackbox) |
| Job `smokeping` DOWN | zła nazwa/port | w `prometheus.yml` ma być `smokeping:9374` |
| Wykresy przesunięte w czasie | zegar VM | `timedatectl`, włącz NTP |
| Kończy się dysk | retencja | zmniejsz `--storage.tsdb.retention.time` |
| Skoki na wszystkich celach naraz | problem lokalny (VM, host, Wi-Fi, łącze) | sprawdź bramę i obciążenie hosta |

Jak czytać: utrata naraz do wielu różnych celów = problem blisko sondy, nie u celu; trwały skok o kilka ms
tylko do jednego celu = zwykle zmiana trasy; brama powinna mieć < 1 ms i zero utraty.

## Checklista

- [ ] VM z Debianem/Ubuntu, NTP, stały IP
- [ ] Docker + compose działa
- [ ] Repo w `/opt/sonda`, `.env` utworzony
- [ ] `docker compose up -d`, wszystkie kontenery running
- [ ] Target `smokeping` UP w Prometheusie
- [ ] Dashboard: opóźnienie + utrata + heatmapa, jeden panel na cel
- [ ] Alerty „sonda nie działa” i „utrata > 5%”
- [ ] Eksport dashboardu (JSON) w repo
- [ ] Wersje obrazów przypięte

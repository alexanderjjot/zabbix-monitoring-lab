# zabbix-monitoring-lab
# Hybrid Monitoring Lab: Zabbix & Docker

## Opis projektu
Projekt autorskiego laboratorium monitoringu opartego na stosie Zabbix, wdrożonego w architekturze kontenerowej. System nadzoruje infrastrukturę hybrydową składającą się z systemów Linux (Ubuntu) oraz Windows (10 i Server 2022).

## Stack Techniczny
* **Engine:** Docker & Docker Compose
* **Monitoring:** Zabbix (v6.4.21)
* **Database:** PostgreSQL
* **OS & HARDWARE:**
### Wykaz monitorowanych hostów

| Nazwa Hostu | System Operacyjny | Adres IP (zanonimizowany) | Wersja Agenta | Status |
|:---|:---|:---|:---|:---|
| **SRV-WIN-2022** | Windows Server 2022 | 192.168.1.xxx | 7.0.25 | ✅ Aktywny |
| **WRK-WIN-10** | Windows 10 IoT | 192.168.1.xxx | 7.0.26 | ✅ Aktywny |
| **UBUNTU-HOST** | Ubuntu 24.04 LTS | 172.17.0.1 (Docker GW) | 7.0.26 | ✅ Aktywny |       |                                      |                  |                        |                         |                                       |                |     |
### Infrastruktura Sprzętowa (Lab Hardware)

| Hostname | Procesor | RAM | Rola w Laboratorium |
|:---|:---|:---|:---|
| **UBUNTU-HOST** | Intel Core i5-5200U (2.20GHz) | 8GB DDR3 1600MHz | Host Docker, Zabbix Server, DB |
| **SRV-WIN-2022** | Intel Celeron N2840 (2.16GHz) | 4GB DDR3 1333MHz | Monitorowany Serwer (Active Agent) |
| **WRK-WIN-10** | Intel Pentium P6200 (2.13GHz) | 8GB DDR3 1333MHz | Monitorowana Stacja Robocza |

## Konfiguracja sieciowa i Docker

Projekt wykorzystuje architekturę kontenerową oraz hybrydowe środowisko sieciowe (Wi-Fi/LAN), co pozwoliło na przetestowanie przepływu danych w warunkach zbliżonych do rzeczywistych sieci firmowych.

### Kluczowe aspekty infrastruktury:

* **Hybrydowe medium transmisyjne:** Serwer monitoringu (Ubuntu) pracuje w sieci bezprzewodowej (**Wi-Fi**), podczas gdy monitorowane stacje robocze podłączone są kablem (**LAN**). Wymusiło to precyzyjną konfigurację routingu na routerze, aby zapewnić pełną komunikację między różnymi segmentami sieci domowej.
* **Adresacja IP i Docker Gateway:**
    * **Publiczny adres hosta (LAN):** `192.168.1.24` – na tym adresie serwer nasłuchuje raportów od agentów Windows.
    * **Brama Dockera (Docker0):** `172.17.0.1` – adres wykorzystany do monitorowania systemu Ubuntu przez kontener Zabbixa (komunikacja na linii kontener <-> system gospodarza).
* **Zarządzanie portami (10051 / 10052):**
    * W załączonym pliku `docker-compose.yml` zastosowano standardowe mapowanie portów `- "10051:10051"`, co zapewnia bezproblemową komunikację z agentami w trybie Active.
    * W trakcie rozwoju projektu testowano również architekturę z wykorzystaniem portu **10052** jako zewnętrznego punktu styku. Pozwoliło to na zweryfikowanie elastyczności konfiguracji Zabbixa oraz przetestowanie reguł Firewall na stacjach Windows pod kątem niestandardowego ruchu wychodzącego.
* **Bezpieczeństwo i Anonimizacja (OPSEC):** Zrzuty ekranu zamieszczone w dokumentacji zawierają celowo zanonimizowane fragmenty adresów IP (oznaczone jako „XX” lub zakryte). Jest to świadome działanie zgodne z dobrymi praktykami bezpieczeństwa IT, mające na celu ochronę szczegółów technicznych infrastruktury przy publicznej prezentacji projektu.

### Pliki konfiguracyjne
W repozytorium znajduje się plik `docker-compose.yml`. Zgodnie z zasadami bezpieczeństwa, zmienne środowiskowe (hasła, loginy do bazy danych) zostały wydzielone do zewnętrznych plików `.env`, które nie są upubliczniane w systemie kontroli wersji.

W repozytorium znajduje się plik `docker-compose.yml` (z zanonimizowanymi danymi dostępowymi), który definiuje całe środowisko.

## Architektura (Draft)
Serwer Zabbix pracuje wewnątrz izolowanej sieci Dockerowej, komunikując się z agentami na hostach fizycznych i wirtualnych poprzez mapowanie portów i konfigurację reguł firewall. Architektura laboratorium wykorzystuje mieszane medium transmisyjne: serwer monitoringu (Ubuntu) komunikuje się poprzez sieć bezprzewodową (Wi-Fi), natomiast stacje monitorowane podłączone są do infrastruktury kablowej (LAN). Wymusiło to odpowiednią konfigurację reguł routingu na routerze brzegowym, aby zapewnić pełną idoczność między podsieciami.

## Monitorowane parametry (Latest Data)

W ramach laboratorium skonfigurowałem zbieranie kluczowych metryk wydajnościowych:
* **CPU Load / Utilization** (procentowe zużycie oraz obciążenie średnie).
* **Memory usage** (pamięć dostępna vs wykorzystana).
* **ICMP Ping** (dostępność hostów w sieci lokalnej).
* **System information** (wersje OS, uptime).

### Wizualizacja danych
Poniżej znajdują się zrzuty ekranu przedstawiające poprawną komunikację serwera z agentami oraz przykładowe wykresy wydajnościowe.

![Status komunikacji z agentami](zabbix_hosts.png)
![Dashboard Monitoringu](dashboard.png)
![Wykres CPU Windows Server 2022](zabbix_graph_SRV_CPU.png)
![Wykres CPU Windows 10 IoT](zabbix_graph_WRK_CPU.png)

## Dokumentacja problemów (Troubleshooting & Case Studies)

### 1. Windows Defender vs Zabbix Agent
**Problem:** Przerywane wykresy CPU/RAM na Windows 10.
**Analiza:** Heurystyka Windows Defender blokowała pakiety agenta po krótkim czasie aktywności.
**Rozwiązanie:** Skonfigurowano wykluczenia (Exclusions) dla procesu `zabbix_agentd.exe` oraz reguły Inbound w Firewallu.

### 2. Docker Bridge Networking
**Problem:** Brak łączności z agentem na systemie-hoście.
**Rozwiązanie:** Przekierowanie ruchu na dedykowany port 10052 i konfiguracja `ServerActive` na adres IP bramy Docker.

## Rozwiązanie problemu (Troubleshooting)

Podczas wdrażania monitoringu napotkano problem z ciągłością danych dla stacji roboczych z systemem Windows (**WRK-WIN-10**). Poniżej znajduje się analiza i opis techniczny rozwiązania.

### 1. Symptomy
* Wykresy w panelu Zabbix były przerywane (tzw. "dziury" w danych - stacja robocza WRK-WIN-10)
* Dane pojawiały się z dużym opóźnieniem lub były ignorowane przez serwer.

### 2. Diagnoza (Root Cause Analysis)
Podczas inspekcji logów i porównania czasu systemowego wykryto niezgodność stref czasowych (**Timezone Mismatch**):
* **Host/Agenty:** Pracowały w czasie lokalnym (**CEST**, UTC+2).
* **Zabbix Server (Docker):** Pracował w czasie uniwersalnym (**UTC**).
* **Clock Drift:** Wykryto również 5-sekundowe przesunięcie zegara między kontenerem a systemem operacyjnym hosta.

Zabbix Server interpretował dane od agentów jako "dane z przyszłości", co powodowało błędy w indeksowaniu bazy danych i uniemożliwiało poprawne renderowanie wykresów w czasie rzeczywistym.

### 3. Rozwiązanie
Zastosowano wymuszoną synchronizację czasu dla całego stosu technologicznego:
* **Konfiguracja Docker Compose:** Dodano zmienną środowiskową `TZ=Europe/Warsaw` dla usług `zabbix-server` oraz `zabbix-web-apache-php`.
* **Mapowanie wolumenów:** Zmapowano pliki systemowe `/etc/localtime` oraz `/etc/timezone` z hosta do kontenerów w trybie tylko do odczytu (`ro`).
* **Weryfikacja:** Potwierdzono poprawność czasu wewnątrz kontenera:
  ```bash
  sudo docker exec -it zabbix-docker-zabbix-server-1 date
  # Wynik: Wed May 13 12:41:17 CEST 2026


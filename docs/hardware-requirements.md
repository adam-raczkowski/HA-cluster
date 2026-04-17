# Wymagania sprzętowe — HA Cluster

**Projekt:** Dwuwęzłowy klaster wysokiej dostępności (Pacemaker + Corosync + DRBD)
**Data opracowania:** 2026-04-17

## Architektura docelowa

```
                   ┌──────────────┐
                   │   Internet   │
                   └──────┬───────┘
                          │
                   ┌──────▼───────┐
                   │  Router WAN  │  (TP-Link ER605)
                   └──────┬───────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
    │ Switch  │◄────►│ Switch  │      │  RPi 5  │  (arbiter/qdevice)
    │ mgmt/   │      │heartbeat│      └─────────┘
    │ service │      │  (opt.) │
    └──┬───┬──┘      └──┬───┬──┘
       │   │            │   │
    ┌──▼───▼──┐      ┌──▼───▼──┐
    │ Node 1  │◄────►│ Node 2  │
    │         │      │         │
    │  ESP32  │      │  ESP32  │   (reset/power przez relay)
    └─────────┘      └─────────┘
         │                │
         └──────┬─────────┘
                │
            ┌───▼───┐
            │  UPS  │
            └───────┘
```

## 1. Węzły klastra (2 szt.)

Pozyskanie planowane z PWr — poniżej ogólne wymagania minimalne i zalecane.

### Wymagania minimalne (proof-of-concept)
- **CPU:** 4 rdzenie / 8 wątków, x86-64 (Intel 8. gen / AMD Ryzen 3000 lub nowszy)
- **RAM:** 8 GB (DDR4)
- **Dysk systemowy:** SSD SATA/NVMe 120 GB
- **Dysk DRBD:** dodatkowy SSD 240+ GB (osobny wolumen dla replikacji blokowej)
- **Sieć:** min. 2× NIC 1 GbE (uplink + heartbeat)

### Wymagania zalecane (docelowe)
- **CPU:** 6–8 rdzeni, wsparcie wirtualizacji (VT-x/AMD-V), najlepiej ECC
- **RAM:** 16–32 GB ECC DDR4/DDR5
- **Dysk systemowy:** NVMe 256 GB
- **Dysk DRBD:** NVMe/SSD 500 GB — **identyczny model i pojemność na obu nodach** (krytyczne dla DRBD)
- **Sieć:** 3× NIC 1 GbE (mgmt, service, heartbeat) lub 2× 1 GbE + 1× 2.5/10 GbE dla DRBD
- **Zasilacz:** redundancja niekonieczna (zewnętrzny UPS)
- **Identyczność:** oba nody powinny być maksymalnie zbliżone sprzętowo (upraszcza failover i testy)

**Uwaga:** jeśli PWr nie udostępni sprzętu, realistyczna opcja "kupna nowego" to mini PC (Minisforum/Beelink z Ryzen 7 lub Intel Core Ultra) — ok. **3000–4000 PLN/szt.** Poniższy kosztorys zakłada scenariusz pozyskania z uczelni (koszt 0).

## 2. Sprzęt sieciowy (TP-Link)

### Switch zarządzalny — główny
**TP-Link TL-SG2210MP** (Omada, 8× PoE+ 1 GbE + 2× SFP)
- Zarządzanie przez Omada Controller / web
- Obsługa VLAN (separacja heartbeat / mgmt / service)
- Link Aggregation (LACP) — przydatne do bondowania NIC
- PoE+ zasili Raspberry Pi (przez HAT) i ew. AP

**Alternatywa budżetowa:** TL-SG2008P (8× 1 GbE, 4× PoE+) — ok. 500 PLN

### Switch heartbeat (opcjonalnie, dla redundancji)
Można użyć niezarządzanego taniego switcha **TL-SG105** (5× 1 GbE) do dedykowanej sieci heartbeat, lub zrobić bezpośrednie połączenie kablem DAC/Cat6 między nodami (bez switcha) — **zalecane dla 2-node setup**.

### Router brzegowy
**TP-Link ER605 (TL-R605)** — VPN router, 1× WAN + 3× LAN/WAN gigabit
- Integracja z Omadą (jedna konsola zarządzania)
- VPN (IPsec/OpenVPN/WireGuard) — zdalny dostęp do labu
- Load balancing / failover WAN

### Kable
- 5–8× patchcord Cat6 1–2 m
- Ew. kabel crossover/DAC dla bezpośredniego heartbeat

## 3. Arbiter — Raspberry Pi 5

Rola: **qdevice** w Corosync (trzeci głos dla quorum, zapobiega split-brain w setupie 2-node). Dodatkowo może hostować Omada Controller (software), monitoring (Prometheus/Grafana).

**Zestaw:**
- Raspberry Pi 5 **4 GB** (wystarczy dla qdevice + lekki monitoring; 8 GB jeśli Omada + Grafana)
- Oficjalny zasilacz 27 W USB-C
- Aktywne chłodzenie (Active Cooler)
- Obudowa (oficjalna lub aluminiowa pasywna)
- Karta microSD **64 GB A2** (Samsung Pro Endurance / SanDisk High Endurance) — alternatywnie NVMe HAT + SSD 256 GB (~150 PLN extra, znacznie lepsza żywotność)

## 4. ESP32 — zdalny reset/power (IP-KVM pinowy)

Rola: zwieranie pinów PWR_SW / RST_SW na płycie głównej przez przekaźniki, sterowane z ESP32 po WiFi/Ethernet (REST API / MQTT). Substytut IPMI dla sprzętu consumer-grade.

**Zestaw na 1 node (×2):**
- **ESP32-DevKitC** (lub ESP32-S3 dla USB native / ETH: ESP32 z PHY LAN8720)
- **Moduł przekaźników 2-kanałowy 5V** z optoizolacją (kanały: PWR + RESET)
  - Alternatywa: moduł z przekaźnikami SSR (bardziej niezawodne, droższe)
- Zasilacz 5V 2A (USB)
- Zworki Dupont F-F (do pinów na płycie głównej)
- Mała obudowa / drukowana 3D
- Opcjonalnie: czujnik stanu LED (fotorezystor + ADC) do odczytu, czy komputer działa — pozwala ESP32 wykrywać stan zasilania bez polegania wyłącznie na pingu

**Firmware:** ESPHome lub własny (Arduino / ESP-IDF) z REST/MQTT. Integracja z Home Assistant lub skryptem fencingowym w Pacemaker (custom STONITH agent wywołujący HTTP na ESP32).

## 5. UPS (zasilanie awaryjne)

**Krytyczne dla HA** — chroni przed false-failover przy krótkich zanikach zasilania i pozwala na graceful shutdown.

**Rekomendacja:** APC Back-UPS BX950U-GR lub CyberPower Value 1000EI
- ~600–1000 VA
- Komunikacja USB → NUT (Network UPS Tools) na jednym z nodów → sygnał do drugiego przez sieć
- Zasila: oba nody + switch + router + RPi (nie zasilać ESP32 przez UPS — ma być niezależny!)

## 6. Opcjonalne / nice-to-have

- Szafka rack 6U lub półka techniczna
- PDU z monitoringiem zużycia (pomiar jeden z eksperymentów?)
- KVM sprzętowy (klawiatura/monitor) do bootstrapu — może być rozwiązane przez ESP32 z UART

---

## Kosztorys (szacunkowy, PLN, stan rynku kwiecień 2026)

### Scenariusz A — nody z PWr

| Kategoria | Pozycja | Cena [PLN] |
|-----------|---------|-----------:|
| **Sieć** | TP-Link TL-SG2210MP (switch zarządzalny PoE) | 1050 |
|          | TP-Link ER605 (router brzegowy) | 350 |
|          | Patchcordy Cat6 ×6 | 60 |
| **Arbiter** | Raspberry Pi 5 4GB | 330 |
|             | Zasilacz oficjalny 27W | 60 |
|             | Active Cooler | 50 |
|             | Obudowa | 70 |
|             | microSD 64GB A2 (Pro Endurance) | 60 |
| **ESP32 KVM** (×2) | ESP32-DevKitC | 2× 45 = 90 |
|                    | Moduł przekaźników 2-ch z optoizolacją | 2× 20 = 40 |
|                    | Zasilacz 5V + okablowanie + obudowa | 2× 40 = 80 |
| **UPS** | APC Back-UPS ~900VA | 650 |
| **Inne** | Kable, zworki, drobnica | 100 |
| **SUMA (A)** | | **~2990 PLN** |

### Scenariusz B — wszystko nowe (nody też)

| Kategoria | Pozycja | Cena [PLN] |
|-----------|---------|-----------:|
| **Nody** | 2× Mini PC (Ryzen 7 / Core Ultra, 16 GB, NVMe 512 GB) | 2× 3500 = 7000 |
|          | 2× dodatkowy SSD 500 GB (DRBD) | 2× 220 = 440 |
|          | 2× karta USB→Ethernet 1 GbE (jeśli mini PC ma 1 NIC) | 2× 80 = 160 |
| **Sieć + arbiter + ESP32 + UPS + drobnica** | j.w. | 2990 |
| **SUMA (B)** | | **~10 590 PLN** |

### Scenariusz C — minimum budżetowe (nody z PWr, switche niezarządzalne)

| Kategoria | Pozycja | Cena [PLN] |
|-----------|---------|-----------:|
| Switch TL-SG108 niezarządzalny | | 90 |
| Router TP-Link Archer (home) | | 200 |
| Raspberry Pi 5 4GB + osprzęt | | 570 |
| ESP32 ×2 + relays | | 210 |
| UPS podstawowy | | 450 |
| Drobnica | | 80 |
| **SUMA (C)** | | **~1600 PLN** |

---

## Uwagi końcowe

- **VLAN-y** na switchu zarządzalnym pozwolą zrealizować separację sieci (service / mgmt / heartbeat / storage) bez dokupowania fizycznych switchy — wart rozważenia przy wyborze TL-SG2210MP zamiast tańszego niezarządzalnego.
- **DRBD** bardzo zyskuje na łączu ≥ 2.5 GbE lub 10 GbE dla dysków NVMe — jeśli budżet pozwala, warto rozważyć TL-SG3210XHP-M2 (z portami 2.5G) zamiast SG2210MP. Koszt: ~2000 PLN.
- **STONITH/fencing:** ESP32 pełni tę rolę w tym projekcie. Warto od razu zaprojektować protokół REST API (np. `POST /reset`, `POST /power`, `GET /status`) i napisać custom fence agent dla Pacemakera.
- **Raspberry Pi jako qdevice** — konfiguracja `corosync-qnetd` + `corosync-qdevice` na nodach. RPi nie jest pełnoprawnym trzecim nodem, tylko "arbitrem" rozstrzygającym quorum.
- Wszystkie ceny są orientacyjne — przed zakupem zweryfikować na Morele/x-kom/AliExpress (dla ESP32 i akcesoriów elektronicznych AliExpress ~50% taniej).

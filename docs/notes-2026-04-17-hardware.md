# Notatka — rozkminy sprzętowe

**Data:** 2026-04-17
**Kontekst:** Dobór sprzętu do pracy inżynierskiej "Projektowanie i wdrożenie dwuwęzłowego klastra wysokiej dostępności"

## Wyjściowe założenia (z rozmowy)

- Sprzęt sieciowy — sam kupię, preferowany **TP-Link** (Omada).
- Arbiter (qdevice Corosync) — **Raspberry Pi 5**.
- Fencing / zdalny reset — **ESP32** zwierający piny power/reset przez przekaźniki.
- Nody — początkowo plan: pozyskać z PWr. Zrewidowane — patrz niżej.
- Serwery będą stały **w mieszkaniu** podczas pisania pracy → wymóg: **cicho**, niska moc.
- Po obronie chcę dalej korzystać z rozwiązania → sensowniej kupić samemu.
- Budżet studencki — **zdecydowanie < 3000 PLN/serwer**, najlepiej dużo mniej.

## Rozpatrywane opcje nodów

| Opcja | Plusy | Minusy | Koszt/szt. |
|-------|-------|--------|-----------:|
| Refurb enterprise (Dell R630, HPE DL360 Gen9) | IPMI, ECC, enterprise | **Głośne**, prądożerne, duże | 2000–4000 PLN |
| Minisforum MS-01 (nowy) | 2× 10GbE, 3× NVMe, Xeon/Core Ultra | Drogi | 4500–6000 PLN |
| **Używane mini PC biz (OLX)** — HP EliteDesk Mini, Lenovo ThinkCentre Tiny, Dell OptiPlex Micro | Cicho (~20 dB), tanio, polska gwarancja OLX, szeroka dostępność | 1 NIC (USB adapter do dokupienia), brak ECC | **600–900 PLN** |
| Chińskie mini PC N100/N305 4×i226 2.5GbE (CWWK/Topton, AliExpress) | 4 NIC-i 2.5GbE out-of-the-box, fanless, nowe | Wysyłka 2–4 tyg., brak PL gwarancji, słabsze CPU | 600–1500 PLN |
| Thin client (HP t620/t630, Dell Wyse) | Bardzo tanio, fanless | Za słabe do DRBD+VM, limitowany RAM | 200–400 PLN |

### Ustalenie wstępne (do potwierdzenia)

**Preferowany kierunek:** 2× używane mini PC enterprise z OLX (HP EliteDesk 800 G3/G4 Mini, Lenovo M720q/M920q, Dell OptiPlex 7060/7070 Micro).

Przykładowy budżet:
- 2× mini PC (i5 8-gen, 16 GB, 256 NVMe) — 2× 800 PLN
- 2× USB 3.0 → 2.5GbE (Realtek RTL8156) — 2× 90 PLN
- 2× SSD SATA 500 GB dla DRBD — 2× 180 PLN
- **Razem ~2100 PLN za oba nody**

## Temat fencingu — AMT vs ESP32

**Problem:** HP EliteDesk Mini nie ma standardowego headera PWR/RST na płycie głównej; przycisk power to microswitch na front I/O board. Reset button nie istnieje fizycznie.

### Wnioski techniczne

- ESP32 + przekaźnik **da się wpiąć** — lutowanie równoległe do microswitcha power button (~30 min roboty per node).
- Brak fizycznego reset = hard reset realizowany programowo: `force_off (5s) → pauza → power_on`. Dla fencingu Pacemaker w 100% wystarczy.
- Wiele modeli EliteDesk 800 z CPU vPro ma **Intel AMT** = pełny out-of-band management (power, KVM, boot redirect) — **faktyczny zamiennik IPMI**.

### Decyzja narracyjna dla pracy

Fencing to **jeden z wielu komponentów** w temacie ogólnym, nie główny wkład. Główny ciężar pracy:
- DRBD (protokoły, tuning)
- Pacemaker/Corosync (zasoby, constrainty, stickiness)
- Quorum + qdevice (RPi)
- Scenariusze failover + pomiary
- Split-brain handling
- Monitoring (Prometheus/Grafana)
- Automatyzacja (Ansible)

**Strategia doboru sprzętu niezależna od vPro:**
- Kupuję co tanie, nie filtruję po vPro.
- Jeśli trafi się vPro → AMT jako *primary* fence agent (`fence_amt_ws`), ESP32 jako *alternatywa porównawcza* w pracy ("fencing enterprise vs commodity").
- Jeśli nie → ESP32 jedyny fencing, uzasadnienie: "wdrożenie na commodity hardware bez BMC".

W obu przypadkach **ESP32-KVM budujemy** — to konkretny artefakt inżynierski, zostaje w homelab.

## Status ustaleń

- [x] Plik `hardware-requirements.md` — utworzony w pierwotnej wersji (scenariusze A/B/C, nody "z PWr" vs "nowe")
- [ ] **Do aktualizacji:** rewizja pod używki z OLX, rozdział o AMT vs ESP32, realny budżet ~2100 PLN za oba nody
- [ ] Potwierdzić kierunek: używane EliteDeski/ThinkCentre vs chińskie N100 z 4× 2.5GbE
- [ ] Po potwierdzeniu — przepisać sekcje w `hardware-requirements.md`

## Otwarte pytania

1. Czy użyć chińskich N100 (4× 2.5GbE per node = luksus sieciowy dla DRBD) mimo braku PL gwarancji?
2. Czy w pracy uwzględniać comparative study AMT vs ESP32, czy trzymać się jednej ścieżki?
3. Czy Raspberry Pi 5 ma hostować tylko qdevice, czy też Omada Controller + Prometheus/Grafana? (wpływa na wybór 4 vs 8 GB RAM)

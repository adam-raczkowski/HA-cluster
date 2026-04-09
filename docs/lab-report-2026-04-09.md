# Lab 01 — Konfiguracja środowiska klastra HA

**Data:** 2026-04-09  
**Temat pracy inżynierskiej:** Projektowanie i implementacja dwuwęzłowego klastra wysokiej dostępności  
**Środowisko:** MacBook Air M4, UTM (Apple Virtualization Framework)

## Cel

Postawienie dwóch maszyn wirtualnych i skonfigurowanie podstawowego klastra Pacemaker + Corosync z zasobem VIP (wirtualny adres IP) oraz weryfikacja mechanizmu failover.

> **Uwaga:** Środowisko wirtualne (UTM) służy wyłącznie do celów testowych i prototypowania konfiguracji. Docelowe wdrożenie klastra zostanie przeprowadzone na rzeczywistych urządzeniach fizycznych.

## Środowisko wirtualne

- **Hypervisor:** UTM w trybie Virtualize (natywna wirtualizacja ARM-on-ARM)
- **System operacyjny VM:** Ubuntu Server 24.04.4 LTS (arm64)
- **Parametry każdej VM:** 2 vCPU, 2 GB RAM, 20 GB VirtIO disk

### Konfiguracja sieciowa

Każda VM posiada dwa interfejsy sieciowe:

| Interfejs | Tryb | Przeznaczenie |
|-----------|------|---------------|
| `enp0s1` | Shared Network (NAT) | Dostęp do internetu, aktualizacje pakietów |
| `enp0s2` | Host Only | Komunikacja klastrowa (heartbeat) |

Adresy statyczne interfejsu heartbeat (netplan `/etc/netplan/99-cluster.yaml`):

| Node | Adres IP |
|------|----------|
| ha-node1 | 10.10.10.1/24 |
| ha-node2 | 10.10.10.2/24 |

## Przebieg prac

### 1. Instalacja maszyn wirtualnych

- Pobranie ISO Ubuntu Server 24.04.4 LTS w wersji **arm64** (początkowy błąd: pobrano wersję amd64, co skutkowało wylądowaniem w UEFI Shell zamiast uruchomienia instalatora).
- Instalacja ha-node1 z włączonym OpenSSH Server.
- Klonowanie ha-node1 → ha-node2 za pomocą funkcji Clone w UTM.
- Zmiana hostname na klonie: `sudo hostnamectl set-hostname ha-node2`.

### 2. Konfiguracja sieci heartbeat

- Dodanie drugiego interfejsu sieciowego (Host Only) do obu VM.
- Konfiguracja statycznych adresów IP przez netplan.
- Wpisy w `/etc/hosts` na obu nodach:

```
10.10.10.1  ha-node1
10.10.10.2  ha-node2
```

**Problem napotkany:** Domyślny wpis `127.0.1.1 ha-nodeX` w `/etc/hosts` powodował, że Corosync bindował się na localhost zamiast na adres heartbeat. Rozwiązanie: zakomentowanie linii `127.0.1.1`.

### 3. Instalacja i konfiguracja klastra

- Instalacja pakietów: `pacemaker`, `corosync`, `pcs`.
- Ustawienie hasła użytkownika `hacluster` na obu nodach.
- Uwierzytelnienie nodów: `pcs host auth`.
- Utworzenie klastra: `pcs cluster setup ha-cluster ha-node1 ha-node2`.
- Wyłączenie STONITH na potrzeby labbingu: `pcs property set stonith-enabled=false`.

### 4. Zasób VIP i test failover

Utworzenie zasobu wirtualnego IP:

```bash
sudo pcs resource create cluster_vip ocf:heartbeat:IPaddr2 \
  ip=10.10.10.100 cidr_netmask=24 op monitor interval=30s
```

**Test failover:**

1. Stan początkowy: `cluster_vip` uruchomiony na ha-node1.
2. Przełączenie ha-node1 w tryb standby: `pcs node standby ha-node1`.
3. Wynik: VIP automatycznie przemigrował na ha-node2.
4. Po przywróceniu ha-node1 (`pcs node unstandby`) VIP pozostał na ha-node2 (prawidłowe zachowanie — resource stickiness).

## Status końcowy

```
Cluster name: ha-cluster
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ ha-node1 ha-node2 ]

Full List of Resources:
  * cluster_vip (ocf:heartbeat:IPaddr2):    Started ha-node2
```

## Napotkane problemy

| Problem | Przyczyna | Rozwiązanie |
|---------|-----------|-------------|
| UEFI Shell zamiast instalatora | ISO w architekturze amd64 zamiast arm64 | Pobranie prawidłowego ISO arm64 |
| Nody nie widzą się w klastrze (partition WITHOUT quorum) | Corosync bindował na 127.0.1.1 (wpis `/etc/hosts`) | Zakomentowanie `127.0.1.1` w `/etc/hosts` |
| Node2 wisi na starcie (~2 min) | `systemd-networkd-wait-online` czeka na niekonfigurowany interfejs | Dodanie konfiguracji netplan dla enp0s2 |

## Następne kroki

- [ ] Konfiguracja DRBD (replikacja storage na poziomie bloków)
- [ ] Postawienie serwisu HA (Nginx lub PostgreSQL) na wolumenie DRBD
- [ ] Konfiguracja STONITH/fencing (softdog lub fence_virsh)
- [ ] Testy scenariuszy awaryjnych (kill node, split-brain, network partition)

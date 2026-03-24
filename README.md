# Optymalizacja połączeń SSH na Ubuntu (Eliminacja Input Laga)

Repozytorium zawiera zbiór sprawdzonych rozwiązań problemu wysokiego opóźnienia (input lag) oraz wolnego logowania podczas pracy przez terminal z lokalnego systemu linux na serwery VPS i kontenery LXC (np. Mikr.us).

## 1. Niestabilny ping i opóźnienia w sieci Wi-Fi (Lokalnie)

**Problem:** Wysoki ping (np. 130-150 ms) z maszyny lokalnej na systemie linux, podczas gdy na tym samym sprzęcie pod systemem Windows wynosi on znacznie mniej (~50 ms). Przyczyną jest domyślne, agresywne oszczędzanie energii karty Wi-Fi przez moduł `NetworkManager`.

**Rozwiązanie:** Wyłączenie usypiania karty sieciowej.

1. Edytuj plik konfiguracyjny z uprawnieniami root:

   ```bash
   sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
   ```

2. Zmień wartość z 3 (włączone) na 2 (wyłączone):

   ```ini
   [connection]
   wifi.powersave = 2
   ```

3. Zrestartuj usługę zarządzającą siecią:

   ```bash
   sudo systemctl restart NetworkManager
   ```

## 2. Długi czas logowania do serwera (VPS)

**Problem:** Terminal zawiesza się na kilkanaście sekund przed poproszeniem o hasło lub wpuszczeniem użytkownika autoryzowanego kluczem RSA. Wynika to z faktu, że serwer domyślnie próbuje wykonać wsteczne wyszukiwanie DNS (Reverse DNS lookup) dla adresu IP klienta.

**Rozwiązanie:** Wyłączenie dyrektywy UseDNS na docelowym serwerze.

1. Zaloguj się na serwer i edytuj konfigurację demona SSH:

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. Zlokalizuj linię `UseDNS yes` i zmień ją na `UseDNS no` (usuń znak `#`, jeśli jest zakomentowana).

3. Zrestartuj usługę:

   ```bash
   sudo systemctl restart ssh
   ```

> **Uwaga:** W dystrybucjach takich jak CentOS/Arch użyj `sshd` zamiast `ssh`.

## 3. Zastąpienie SSH protokołem Mosh (Mobile Shell)

**Problem:** Standardowy protokół SSH działa synchronicznie, czeka na potwierdzenie od serwera przed wyświetleniem znaku na ekranie. Przy wysokim pingu powoduje to uciążliwy "input lag" podczas pisania komend.

**Rozwiązanie:** Użycie narzędzia Mosh. Działa ono w oparciu o protokół UDP i oferuje lokalne echo (natychmiastowe wyświetlanie wpisywanych znaków), co całkowicie eliminuje odczuwalne opóźnienia w pisaniu.

1. Instalacja (lokalnie oraz na serwerze):

   ```bash
   sudo apt update
   sudo apt install mosh
   ```

2. Zamiast `ssh user@ip`, użyj:

   ```bash
   mosh user@ip
   ```

## 4. Konfiguracja Mosh dla niestandardowych portów (Kontenery LXC / Mikr.us)

**Problem:** W środowiskach współdzielonych domyślny port SSH (22) oraz domyślne porty UDP dla Mosh (60000-61000) są niedostępne. Użytkownik posiada jedynie wąską, losowo przydzieloną pulę portów.

**Rozwiązanie:** Wymuszenie na Mosh użycia konkretnego portu TCP do autoryzacji SSH oraz konkretnego portu UDP z przypisanej puli do właściwej komunikacji.

```bash
mosh --ssh="ssh -p [PORT_SSH]" -p [PORT_UDP] user@adres_serwera
```

Przykład:
Połączenie dla serwera `bogdan312.mikrus.xyz`, przypisanego portu SSH `10312` oraz wolnego portu UDP z puli użytkownika `30251`:

```bash
mosh --ssh="ssh -p 10312" -p 30312 root@bogdan312.mikrus.xyz
```

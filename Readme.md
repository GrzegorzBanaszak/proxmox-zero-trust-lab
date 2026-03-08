# Case Study: Bezpieczna Publikacja Usług (Zero Trust, Cloudflare Tunnels & SSO)

## 📌 Cel projektu

Celem projektu było zaprojektowanie i wdrożenie bezpiecznej architektury dla usług utrzymywanych w środowisku Home Lab (Proxmox VE), umożliwiającej dostęp z dowolnego miejsca na świecie.

Głównym wyzwaniem technologicznym był brak publicznego adresu IP (połączenie z internetem przez operatora stosującego CGNAT/internet radiowy). Dodatkowo, architektura musiała spełniać założenia modelu **Zero Trust**, wymuszając uwierzytelnianie wieloskładnikowe (MFA) dla każdej usługi – nawet dla tych, które natywnie nie posiadają panelu logowania.

## 🛠 Wykorzystany Tech Stack

- **Infrastruktura:** Proxmox VE -> Ubuntu Server 24.04 LTS (VM dedykowana jako Gateway)
- **Kompilacja i Orkiestracja:** Docker & Docker Compose
- **Zarządzanie ruchem brzegowym:** Cloudflare Tunnels (`cloudflared`)
- **Reverse Proxy / Ingress:** Nginx Proxy Manager (NPM)
- **Identity Provider (IdP) & SSO:** Authentik (korzystający z PostgreSQL i Redis)
- **Aplikacja testowa:** Homer (Dashboard)

---

## 🏗 Architektura i Przepływ Ruchu

Zrezygnowano z tradycyjnego podejścia (przekierowanie portów na routerze i wystawienie usług na portach 80/443) na rzecz bezpiecznego, szyfrowanego tunelu wychodzącego.

**Przepływ żądania HTTP/HTTPS:**

1. **Request:** Użytkownik łączy się z adresem `https://homer.gbanaszak.pl`.
2. **Edge Network:** Ruch trafia na serwery brzegowe Cloudflare (gdzie aplikowany jest certyfikat SSL i podstawowy WAF).
3. **Szyfrowany Tunel:** Żądanie przesyłane jest przez Cloudflare Tunnel do demona `cloudflared` działającego w mojej sieci lokalnej. **Żadne porty wejściowe (Inbound) nie są otwarte na firewallu domowym.**
4. **Reverse Proxy:** `cloudflared` przekazuje ruch wewnętrznie do Nginx Proxy Managera (NPM).
5. **Forward Authentication (MFA Check):** NPM zatrzymuje żądanie i weryfikuje u usługi Authentik, czy użytkownik posiada ważną sesję.
   - _Brak sesji:_ Użytkownik jest przekierowywany na stronę logowania `auth.gbanaszak.pl` i musi podać login, hasło oraz kod TOTP.
   - _Sesja ważna:_ Authentik wydaje polecenie przepuszczenia ruchu.
6. **Dostęp do usługi:** Nginx przekazuje ruch do właściwego kontenera (Homer), który natywnie nie posiada żadnych zabezpieczeń.

---

## 🛡 Kluczowe Rozwiązania i Rozwiązywanie Problemów

Podczas wdrożenia zmierzyłem się z kilkoma rzeczywistymi problemami inżynieryjnymi, które z powodzeniem rozwiązałem:

### 1. Ominięcie CGNAT (Carrier-grade NAT)

Z powodu braku publicznego adresu IP, standardowa usługa DDNS i Port Forwarding były niemożliwe do zastosowania. Wdrożyłem **Cloudflare Zero Trust Tunnels**, łącząc moją sieć z chmurą za pomocą agenta, co całkowicie zamaskowało moje publiczne IP (ochrona przed skanowaniem portów i atakami DDoS).

### 2. Forward Authentication dla aplikacji "Legacy"

Wdrożyłem aplikację _Homer_, która domyślnie jest całkowicie otwarta. Dzięki zastosowaniu oficjalnych skryptów integrujących Authentik z blokami `location` w Nginx Proxy Managerze, nałożyłem globalną warstwę SSO i MFA. Żaden pakiet nie dotrze do kontenera docelowego bez weryfikacji tożsamości.

### 3. Debugowanie środowisk Linux / Docker

- Zdiagnozowałem i naprawiłem problem z brakiem łączności zewnętrznej maszyny wirtualnej, modyfikując pliki konfiguracyjne **Netplan** (dodanie serwerów DNS dla statycznego IP).
- Rozwiązałem problem z błędami uprawnień (`Permission denied`) podczas inicjalizacji kontenera Authentik Worker. Kontener (ze względów bezpieczeństwa) działał jako użytkownik niebędący rootem. Użyłem polecenia `chown -R 1000:1000` na zamontowanych wolumenach (Volumes) na hoście, dostosowując uprawnienia do wymogów bezpieczeństwa Dockera.

---

## 📸 Dokumentacja Wizualna (Proof of Work)

Poniżej przedstawiam kluczowe etapy zrealizowanego wdrożenia.

**Tunelu Cloudflare Tunnels**

![Utworzenie tunelu](./docs/screenshots/img-1.png)

**Kontener Cloudeflare wraz z kontenerem testowym Whoami**

![Pierwsze kontenery](./docs/screenshots/img-3.png)

**Test połaczenia z Whoami**

![Test połaczenia z Whoami](./docs/screenshots/img-4.png)

**Utworzenie Nginx Proxy Manager**

![Utworzenie Nginx Proxy Manager](./docs/screenshots/img-5.png)

**Konfiguracja DNS dla Authentik**

![Konfiguracja DNS dla Authentik](./docs/screenshots/img-7.png)

![Konfiguracja DNS dla Authentik 2](./docs/screenshots/img-8.png)

**Logowanie i konfiguracja Authentik**

![Logowanie i konfiguracja Authentik](./docs/screenshots/img-9.png)

![Logowanie i konfiguracja Authentik](./docs/screenshots/img-10.png)

**Konfiguracja weryfikacji dwuetapowej**

![Logowanie i konfiguracja Authentik](./docs/screenshots/img-11.png)
![Logowanie i konfiguracja Authentik](./docs/screenshots/img-12.png)
![Logowanie i konfiguracja Authentik](./docs/screenshots/img-13.png)

**Konfiguracja dla Home Dashboard**

![Logowanie i konfiguracja Authentik](./docs/screenshots/img-14.png)
![Logowanie i konfiguracja Authentik](./docs/screenshots/img-15.png)
![Logowanie i konfiguracja Authentik](./docs/screenshots/img-16.png)

**Kontenery Docker**

![Logowanie i konfiguracja Authentik](./docs/screenshots/img-17.png)

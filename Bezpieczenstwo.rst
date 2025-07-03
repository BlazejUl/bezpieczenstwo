Bezpieczeństwo
===============
:Autorzy: - Katarzyna Tarasek
	  - Błażej Uliasz
         

1. pg_hba.conf — opis pliku konfiguracyjnego PostgreSQL
---------------------------------------------------------

Plik ``pg_hba.conf`` (skrót od *PostgreSQL Host-Based Authentication*) kontroluje, kto może się połączyć z bazą danych PostgreSQL, skąd, i w jaki sposób ma zostać uwierzytelniony.

Format pliku
~~~~~~~~~~~~~~~~~~~~
Każdy wiersz odpowiada jednej regule dostępu:
::
    <typ>  <baza danych>  <użytkownik>  <adres>  <metoda>  [opcje]

Opis elementów:

Znaczenie Elementów
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``<typ>`` — Typ połączenia – np. ``local``, ``host``, ``hostssl``, ``hostnossl``
- ``<baza>`` — Nazwa bazy danych, do której ma być dostęp – konkretna lub ``all``
- ``<użytkownik>`` — Nazwa użytkownika PostgreSQL lub ``all``
- ``<adres>`` — Adres IP lub zakres CIDR klienta (np. ``192.168.1.0/24``); pomijany dla ``local``
- ``<metoda>`` — Metoda uwierzytelnienia – np. ``md5``, ``trust``, ``scram-sha-256``
- ``[opcje]`` — Opcjonalne dodatkowe parametry (np. ``clientcert=1``)

Typy połączeń
~~~~~~~~~~~~~~~~~~~~
- ``local`` — Umożliwia połączenia **lokalne przez Unix socket** (pliki specjalne w systemie plików, np. ``/var/run/postgresql/.s.PGSQL.5432``).  
  Ten tryb jest dostępny **tylko na systemach Unix/Linux** i ignoruje pole ``<adres>``.

- ``host`` — Oznacza połączenia **przez TCP/IP**, niezależnie od tego, czy klient znajduje się na tym samym hoście, czy w sieci.  
  Wymaga podania adresu IP lub zakresu IP (w polu ``<adres>``).

- ``hostssl`` — Jak ``host``, ale **wymusza użycie SSL/TLS**. Połączenia bez szyfrowania będą odrzucone.  
  Wymaga, aby serwer PostgreSQL był poprawnie skonfigurowany do obsługi SSL (np. pliki ``server.crt``, ``server.key``).

- ``hostnossl`` — Jak ``host``, ale **odrzuca połączenia przez SSL/TLS**. Działa tylko dla połączeń nieszyfrowanych.  
  Może być używane do rozróżnienia reguł dla klientów z/do SSL i bez SSL.

Metody uwierzytelniania
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- ``trust`` — brak uwierzytelnienia (niezalecane!)

- ``md5`` — Klient musi podać hasło, które jest przesyłane jako skrót MD5.  
  To popularna metoda w starszych wersjach PostgreSQL, ale obecnie uznawana za przestarzałą (choć nadal obsługiwana).

- ``scram-sha-256`` — Nowoczesna, bezpieczna metoda uwierzytelniania oparta na protokole SCRAM i algorytmie SHA-256.  
  Zalecana w produkcji od PostgreSQL 10 wzwyż. Wymaga, aby hasła w systemie były zapisane jako SCRAM, a nie MD5.

- ``peer`` — Tylko dla połączeń ``local``. Sprawdza, czy nazwa użytkownika systemowego (OS) pasuje do użytkownika PostgreSQL.  
  Stosowane w systemach Unix/Linux.

- ``ident`` — Tylko dla połączeń TCP/IP. Wymaga usługi ident (lub pliku mapowania ident), aby ustalić, kto próbuje się połączyć.  
  Bardziej złożona i rzadziej używana niż ``peer``.

- ``reject`` — Zawsze odrzuca połączenie. Może być użyte do celowego blokowania określonych adresów lub użytkowników.  
  

Przykładowy wpis
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    # 1. Lokalny dostęp bez hasła
    local   all             postgres                                peer



Zmiany i przeładowanie
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Po zmianach w pliku należy przeładować konfigurację PostgreSQL:

::

    pg_ctl reload
    -- lub:
    SELECT pg_reload_conf();


2. Uprawnienia użytkownika
-----------------------------

PostgreSQL pozwala na bardzo precyzyjne zarządzanie uprawnieniami użytkowników lub roli poprzez wiele poziomów dostępu — od globalnych uprawnień systemowych, przez bazy danych, aż po pojedyncze kolumny w tabelach.

Poziom systemowy
~~~~~~~~~~~~~~~~~~~~

To najwyższy poziom uprawnień, nadawany roli jako atrybut. Dotyczy całego klastra PostgreSQL:

- `SUPERUSER` — Pełna kontrola nad serwerem, obejmuje wszystkie uprawnienia

- `CREATEDB` — Możliwość tworzenia nowych baz danych

- `CREATEROLE` — Tworzenie i zarządzanie rolami/użytkownikami

- `REPLICATION` — Umożliwia replikację danych (logiczna/strumieniowa)

- `BYPASSRLS` — Omija polityki RLS (Row-Level Security)



Poziom bazy danych
~~~~~~~~~~~~~~~~~~~~

Uprawnienia do konkretnej bazy danych:

- `CONNECT` — Pozwala na połączenie z bazą danych

- `CREATE` — Pozwala na tworzenie schematów w tej bazie

- `TEMP` — Możliwość tworzenia tymczasowych tabel



Poziom schematu
~~~~~~~~~~~~~~~~~~~~

Schemat (np. `public`) to kontener na tabele, funkcje, typy. Uprawnienia:

- `USAGE` — Umożliwia dostęp do schematu (bez tego SELECT/INSERT nie zadziała)

- `CREATE` — Pozwala tworzyć obiekty (np. tabele) w schemacie



Poziom tabeli
~~~~~~~~~~~~~~~~~~~~

Uprawnienia do całej tabeli :

- `SELECT` — Odczyt danych

- `INSERT` — Wstawianie danych

- `UPDATE` — Modyfikacja danych

- `DELETE` — Usuwanie danych

Przykład
~~~~~~~~~~~~~~
::

    GRANT SELECT, UPDATE ON employees TO hr_team;
    REVOKE DELETE ON employees FROM kontraktorzy;


3. Zarządzanie użytkownikami a dane wprowadzone
--------------------------------------------------

Zarządzanie użytkownikami w PostgreSQL dotyczy tworzenia, usuwania i modyfikowania użytkowników. Sytuacja na którą trzeba tutaj zwrócić uwagę jest usuwanie użytkonika ale pozostawienie danych, które wprowadził. 

Tworzenie i modyfikacja użytkowników
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Do tworzenia nowych użytkowników używamy polecenia ``CREATE USER``. Do modyfikowania użytkowników, którzy już istnieją, używamy polecenia ``ALETER USER``:

::

	CREATE USER username WITH PASSWORD 'password';
	ALTER USER username WITH PASSWORD 'new_password';

Usuwanie użtkowników
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Do usuwania użytkowników, używamy polecenia ``DROP USER`:

::

	DROP USER username;

Dane wprowadzone przez uśytkownika np. za pomocą polecenia ``INSERT`` pozostają, nawet jeśli jego konto zostało usunięte.

Usunięcie użytkownika, a dane które posiadał
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Po usunięciu używtkonika dane, które posiadał nie są automatycznie usuwane. Dane te pozostają w bazie danych ale stają się "niedostępne" dla tego użytkownika. Aby się ich pozbyć, musi to zrobić użytkownik który ma do nich uprawnienia, korzystając z plecenia ``DROP``.

Usunięcie użytkowników, a obietky
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Usuniecie użytkownika, który jest właścicielem obiektów, wygląda inaczej niż przy wcześniejszych danych. Jeżeli użytkownik jest właścicielem jakiegoś obiektu, to jego usunięcie skutkuje błędem:
::
	ERROR: role "username" cannot be droped becouse some objects depend on it

Aby zapobiec takim błędom stosujemy poniższe rozwiazanie:
::
	REASSIGN OWNED BY username TO nowa_rola;
	DROP OWNER BY username;
	DROP ROLE username;

4. Zabezpieczenie połączenia przez SSL/TLS
--------------------------------------------

TLS (Transport Layer Security) i jego poprzednik SSL (Secure Sockets Layer) to kryptograficzne protokoły służące do zabezpieczania połączeń sieciowych. W PostgreSQL służą one do szyfrowania transmisji danych pomiędzy klientem a serwerem, uniemożliwiając podsłuch, modyfikację lub podszywanie się pod jedną ze stron.

Konfiguracja SSL/TLS w PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Konfiguracja serwera: musimy edytować dwa pliki i zrestartować serwer PostgreSQL. Plik ``postgresql.conf``:
::

	ssl = on
	ssl_cert_file = 'server.crt'
	ssl_key_file = 'server.key'
	ssl_ca_file = 'root.crt'    
	ssl_min_protocol_version = 'TLSv1.3'  

oraz ''pg_hba.conf'':

::

	hostssl all all 0.0.0.0/0 cert

Generowanie certyfikatów: jeśli nie używamy komercyjnego CA, możemy sami go wygerenować, a pomocą poniższych komend:
::

	openssl genrsa -out server.key 2048
	openssl req -new -key server.key -out server.csr
	openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

Konfiguracja klienta: parametry SSL, których możemy użyć.

- ``sslmode`` - kontroluje wymuszanie i weryfikację SSL (``require``, ``verify-ca``, ``verify-full``)

- ``sslcert`` - ścieżka do certyfikatu klienta (jeśli wymagane uwierzytelnienie certyfikatem)

- ``sslkey`` -	klucz prywatny klienta

- ``sslrootcert`` - certyfikat CA do weryfikacji certyfikatu serwera

Monitorowanie i testowanie SSL/TLS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sprawdzenie czy połączenie jest szyfrowanie w PostgreSQL wystarczy użyć prostego polecenia ``SELECT ssl_is_used();``. Jeśli jednak chcemy dostać więcej informacji, musimy wpisać poniższe polecenia:
::

	SELECT datname, usename, ssl, client_addr, application_name, backend_type
	FROM pg_stat_ssl
	JOIN pg_stat_activity ON pg_stat_ssl.pid = pg_stat_activity.pid
	ORDER BY ssl;

Testowanie z poziomu terminala pozwala podejrzeć szczegóły TLS takie jak certyfikaty, wesję protokołu czy użyty szyft. Wpisujemy poniższą komendę:
::
	openssl s_client -starttls postgres -connect example.com:5432 -showcerts


5. Szyfrowanie danych
-----------------------

Szyfrowanie danych w PostgreSQL odgrywa kluczową rolę w zapewnianiu poufności, integralności i ochrony danych przed nieautoryzowanym dostępem. Można je realizować na różnych poziomach: transmisji (in-transit), przechowywania (at-rest) oraz aplikacyjnym.

Szyfrowanie transmisji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Korzystając z technologi SSL/TLS chroni dane przesyłane pomiędzy klientem, a serwerem przed podsłuchiwaniem lub modyfikacją. Wymaga konfiguracji serwera PostgreSQL do obsługi SSL oraz klienci muszą łączyć się przez SSL. 

Szyfrowanie całego dysku
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dane są szyfrowane na poziomie systemu operacyjnego lub warstwy przechowywania. Stosowanymi roziazaniami jest LUKS, BitLocker, szyfrowanie oferowane przez chmury. Zaletami tego szyfrowania jest transparentność dla PostgrSQL i łatwość w implementacji. Wadami za to jest brak selektywnego szyfrowania oraz fakt, że jeśli system jest aktywny to dane są odszyfrowane i dostępne. 

Szyfrowanie na poziomie kolumn z użyciem pgcrypto
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pozwala na szyfrowanie konkretnych kolumn danych. Rozszerzenie to ``pgcrypto``. Funkcje takiego szyfrowania to:

- symetryczne szyfrowanie

::

	SELECT pgp_sym_encrypt('tajne dane', 'haslo');
	SELECT pgp_sym_decrypt(kolumna::bytea, 'haslo');


- asymetryczne szyfrowanie (z uśyciem kluczy publicznych/prywatnych)

- haszowanie

::

	SELECT digest('haslo', 'sha256');

Zaletami tego szyfrowania jest duża elastyczność i selektywne szyfrowanie. Wadami zaś wydajność i konieczność zarządzania kluczami w aplikacji. 

Szyfrowanie na poziomie aplikacji
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dane są szyfrowane przed zapisaniem do bazy danych i odszyfrowywane po odczycie. Używane biblioteki:

- Python – cryptography, pycryptodome,

- Java – javax.crypto, Bouncy Castle,

- JavaScript – crypto, sjcl.

Zaletami jest pełna kontrola nad szyfrowaniem oraz fakt, że dane są chronione nawet w razie włamania do bazy. Wadami zaś trudniejsze wyszukiwanie i indeksowanie, konieczność przeniesienia odpowiedzialności za bezpieczeństwo do aplikacji oraz problemy ze zgodnością przy migracjach danych.

Zarządzanie kluczami szyfrującymi
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Niezależnie od rodzaju szyfrowania, bezpieczne zarządzanie kluczami jest kluczowe dla ochrony danych. Klucze powinny być generowane, przechowywane, dystrybuowane i niszczone w sposób bezpieczny. Potrzebne są do tego odpowiednie narzędzia. Rekomendowanymi narzędziami do bezpiecznego zarządzania kluczami są:

- Sprzętowe moduły bezpieczeństwa (HSM) - Urządzenia te oferują bezpieczne środowisko do generowania, przechowywania i zarządzania kluczami. HSM-y są odporne na fizyczne ataki i zapewniają wysoki poziom bezpieczeństwa. 

- Systemy zarządzania kluczami (KMS) - KMS to oprogramowanie, które centralizuje zarządzanie kluczami, umożliwiając ich bezpieczne przechowywanie, rotację i dystrybucję. 



- Narzędzia do bezpiecznej komunikacji - Narzędzia takie jak Signal czy WhatsApp oferują szyfrowanie end-to-end, które chroni komunikację przed nieautoryzowanym dostępem. 

- Narzędzia do szyfrowania dysków - Takie jak BitLocker czy FileVault, które pozwalają na zaszyfrowanie całego dysku twardego lub jego partycji. 

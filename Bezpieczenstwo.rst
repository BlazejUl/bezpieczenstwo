Bezpieczeństwo
=====
:Autorzy: - Kasia Tarasek
	  - Błażej Uliasz
         

pg_hba.conf — opis pliku konfiguracyjnego PostgreSQL
--------------------

Plik ``pg_hba.conf`` (skrót od *PostgreSQL Host-Based Authentication*) kontroluje, kto może się połączyć z bazą danych PostgreSQL, skąd, i w jaki sposób ma zostać uwierzytelniony.

Format pliku
~~~~~~~~~~
Każdy wiersz odpowiada jednej regule dostępu:
::
    <typ>  <baza danych>  <użytkownik>  <adres>  <metoda>  [opcje]

Opis elementów:

Znaczenie Elementów
~~~~~~~~

- ``<typ>`` — Typ połączenia – np. ``local``, ``host``, ``hostssl``, ``hostnossl``
- ``<baza>`` — Nazwa bazy danych, do której ma być dostęp – konkretna lub ``all``
- ``<użytkownik>`` — Nazwa użytkownika PostgreSQL lub ``all``
- ``<adres>`` — Adres IP lub zakres CIDR klienta (np. ``192.168.1.0/24``); pomijany dla ``local``
- ``<metoda>`` — Metoda uwierzytelnienia – np. ``md5``, ``trust``, ``scram-sha-256``
- ``[opcje]`` — Opcjonalne dodatkowe parametry (np. ``clientcert=1``)

Typy połączeń
~~~~~~~~~~
- ``local`` — Umożliwia połączenia **lokalne przez Unix socket** (pliki specjalne w systemie plików, np. ``/var/run/postgresql/.s.PGSQL.5432``).  
  Ten tryb jest dostępny **tylko na systemach Unix/Linux** i ignoruje pole ``<adres>``.

- ``host`` — Oznacza połączenia **przez TCP/IP**, niezależnie od tego, czy klient znajduje się na tym samym hoście, czy w sieci.  
  Wymaga podania adresu IP lub zakresu IP (w polu ``<adres>``).

- ``hostssl`` — Jak ``host``, ale **wymusza użycie SSL/TLS**. Połączenia bez szyfrowania będą odrzucone.  
  Wymaga, aby serwer PostgreSQL był poprawnie skonfigurowany do obsługi SSL (np. pliki ``server.crt``, ``server.key``).

- ``hostnossl`` — Jak ``host``, ale **odrzuca połączenia przez SSL/TLS**. Działa tylko dla połączeń nieszyfrowanych.  
  Może być używane do rozróżnienia reguł dla klientów z/do SSL i bez SSL.

Metody uwierzytelniania
~~~~~~~~~~
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
~~~~~~~~~~

::

    # 1. Lokalny dostęp bez hasła
    local   all             postgres                                peer



Zmiany i przeładowanie
~~~~~~~~~~

Po zmianach w pliku należy przeładować konfigurację PostgreSQL:

::

    pg_ctl reload
    -- lub:
    SELECT pg_reload_conf();


Uprawnienia użytkownika
---------

PostgreSQL pozwala na bardzo precyzyjne zarządzanie uprawnieniami użytkowników lub roli poprzez wiele poziomów dostępu — od globalnych uprawnień systemowych, przez bazy danych, aż po pojedyncze kolumny w tabelach.

Poziom systemowy
~~~~~~~~~~

To najwyższy poziom uprawnień, nadawany roli jako atrybut. Dotyczy całego klastra PostgreSQL:

- `SUPERUSER` — Pełna kontrola nad serwerem, obejmuje wszystkie uprawnienia

- `CREATEDB` — Możliwość tworzenia nowych baz danych

- `CREATEROLE` — Tworzenie i zarządzanie rolami/użytkownikami

- `REPLICATION` — Umożliwia replikację danych (logiczna/strumieniowa)

- `BYPASSRLS` — Omija polityki RLS (Row-Level Security)



Poziom bazy danych
~~~~~~~~~~

Uprawnienia do konkretnej bazy danych:

- `CONNECT` — Pozwala na połączenie z bazą danych

- `CREATE` — Pozwala na tworzenie schematów w tej bazie

- `TEMP` — Możliwość tworzenia tymczasowych tabel



Poziom schematu
~~~~~~~~~~

Schemat (np. `public`) to kontener na tabele, funkcje, typy. Uprawnienia:

- `USAGE` — Umożliwia dostęp do schematu (bez tego SELECT/INSERT nie zadziała)

- `CREATE` — Pozwala tworzyć obiekty (np. tabele) w schemacie



Poziom tabeli
~~~~~~~~~~

Uprawnienia do całej tabeli :

- `SELECT` — Odczyt danych

- `INSERT` — Wstawianie danych

- `UPDATE` — Modyfikacja danych

- `DELETE` — Usuwanie danych

Przykład
~~~~
::

    GRANT SELECT, UPDATE ON employees TO hr_team;
    REVOKE DELETE ON employees FROM kontraktorzy;

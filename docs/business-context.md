# Kontekst biznesowy — Metalpol Sp. z o.o.

## O firmie

Metalpol Sp. z o.o. to polski producent komponentów metalowych dla branży automotive. Firma zatrudnia 180 pracowników i prowadzi działalność w trzech halach produkcyjnych. Posiada własny dział serwisu posprzedażowego odpowiedzialny za obsługę reklamacji klientów.

## Problemy zgłoszone przez CEO

| # | Problem | Skutek |
|---|---|---|
| 1 | ~40% maili trafia do spamu lub jest czytane z opóźnieniem | Klienci nie otrzymują potwierdzenia przyjęcia reklamacji |
| 2 | Jeden specjalista obsługuje 30 reklamacji/dzień (szczyt: 80/dzień) | Backlog 2–3 dni, ryzyko przekroczenia SLA |
| 3 | Niespójna kategoryzacja wad między specjalistami | Ten sam typ wady raz klasyfikowany jako "wizualna", raz jako "materiał" |
| 4 | Brak metryk procesowych | Brak wiedzy: ile reklamacji, jakie typy, które linie produkcyjne generują najwięcej problemów |
| 5 | SAP i JIRA nie komunikują się ze sobą | Wszystkie dane przechodzą ręcznie przez Excela — błędy, czas, duplikaty |

## Aktorzy systemu

### Aktorzy ludzcy

| Aktor | Rola |
|---|---|
| **Klient** | Zgłasza reklamację e-mailem (PL lub EN), dołącza zdjęcia wady i numer zamówienia |
| **Specjalista serwisu** | Czyta maile, przepisuje dane do Excela, kategoryzuje wadę, tworzy tickety w JIRA, odpowiada klientowi |
| **Dział jakości** | Odbiera tickety korygujące z JIRA, prowadzi działania naprawcze |

### Systemy zewnętrzne

| System | Rola |
|---|---|
| **Microsoft 365 / Exchange** | Skrzynka reklamacje@metalpol.pl, Microsoft Graph API, webhook na nowe maile |
| **SAP ERP (PP/QM)** | Źródło danych o zamówieniach i batchach produkcyjnych, REST API |
| **JIRA Cloud** | Tickety reklamacyjne (Complaint) i korygujące (Correction), projekt REK |
| **PostgreSQL** | Wewnętrzna baza klientów (read-only), ~15k rekordów |
| **Azure Blob Storage** | Archiwum zdjęć wad, dostęp przez SAS tokens |

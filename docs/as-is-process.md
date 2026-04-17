# Opis procesu AS-IS — Obsługa reklamacji Metalpol

Pełny diagram: [Event Storming AS-IS](../diagrams/as-is/event-storming.md)

---

## Legenda

| Element | Kolor | Opis |
|---|---|---|
| **Event** | Pomarańczowy | Fakt, który się wydarzył — zmiana stanu systemu |
| **Command** | Niebieski | Intencja zmiany — akcja wywołana przez aktora |
| **UI Action** | Żółty | Akcja wykonywana przez aktora w interfejsie |
| **System** | Fioletowy | System zewnętrzny zaangażowany w proces |
| **Hot Spot** | Różowy/czerwony | Problem zgłoszony przez CEO |

---

## Aktorzy

- **Klient** — inicjuje reklamację e-mailem
- **Specjalista serwisu** — główny aktor procesu, obsługuje reklamację ręcznie end-to-end
- **Systemy:** Exchange / Graph API, Excel (Rejestr Reklamacji), JIRA Cloud (REK), SAP ERP (PP/QM), Outlook

---

## Przebieg procesu

### 1. Zgłoszenie reklamacji
- Klient wysyła e-mail na `reklamacje@metalpol.pl` ze zdjęciem wady, numerem zamówienia i opisem (PL lub EN).
- Mail wpada do skrzynki Exchange.

### 2. Odbiór i rejestracja
- Specjalista serwisu odbiera maila i ręcznie przepisuje dane (klient, numer zamówienia, opis wady) do pliku `Rejestr Reklamacji 2026.xlsx`.

### 3. Kategoryzacja wady
- Specjalista subiektywnie ocenia i przypisuje kategorię wady:
  - wizualna / wymiary / materiał / logistyka

### 4. Tworzenie ticketu w JIRA
- Specjalista ręcznie tworzy ticket w JIRA Cloud (projekt REK, issue type: **Complaint**).

### 5. Weryfikacja w SAP
- Specjalista przechodzi do SAP ERP i sprawdza zamówienie oraz batch produkcyjny (`GET /api/v1/orders/{id}`, `GET /api/v1/batches/{id}`).
- Dane przepisuje ręcznie — brak integracji między SAP a JIRA.

### 6. Odpowiedź do klienta
- Specjalista pisze odpowiedź e-mailową przez Outlook.
- Średni czas odpowiedzi: **2 dni od zgłoszenia**.

### 7. Ticket korygujący
- Jeśli wada potwierdzona — specjalista tworzy drugi ticket w JIRA (issue type: **Correction**) dla działu jakości.

---

## Hot Spoty — problemy CEO

| # | Problem | Konsekwencja |
|---|---|---|
| 1 | ~40% maili trafia do spamu | Reklamacje bez odpowiedzi, ryzyko utraty klientów |
| 2 | Backlog 2–3 dni | Przekroczenie SLA, niezadowolenie klientów |
| 3 | Brak metryk | Zero KPI, brak danych do decyzji strategicznych |
| 4 | Niespójna kategoryzacja | Błędne dane analityczne, trudność we wzorcach jakościowych |
| 5 | SAP i JIRA nie są zintegrowane | Ręczne przepisywanie danych — błędy, czas, duplikaty |
| 6 | Ręczne przepisywanie danych | Wysoki koszt operacyjny, błędy ludzkie |
| 7 | Brak wykorzystania bazy klientów | Specjalista nie widzi historii klienta podczas obsługi |
| 8 | Brak danych do decyzji strategicznych | Zero KPI — management działa w ciemno |
| 9 | Ryzyko utraty klientów | Wynikające ze wszystkich powyższych problemów |

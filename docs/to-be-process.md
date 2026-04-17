# Opis procesu TO-BE — Obsługa reklamacji po automatyzacji AI

Pełny diagram: [Event Storming TO-BE](../diagrams/to-be/event-storming.md)

---

## Legenda

| Element | Kolor | Opis |
|---|---|---|
| **Event** | Pomarańczowy | Fakt, który się wydarzył — zmiana stanu systemu |
| **Command** | Niebieski | Intencja zmiany — akcja wywołana przez aktora |
| **UI Action** | Żółty | Akcja wykonywana przez aktora w interfejsie |
| **System** | Fioletowy | System zewnętrzny zaangażowany w proces |
| **AI Action** | Zielony | Akcja wykonywana autonomicznie przez warstwę AI |
| **Polityka biznesowa** | Pomarańczowy (ciemny) | Reguła decyzyjna określająca zachowanie systemu |

---

## Aktorzy

- **Klient** — inicjuje reklamację e-mailem, otrzymuje odpowiedź
- **Specjalista serwisu** — weryfikuje wyniki AI, zatwierdza lub koryguje przed wysłaniem (human-in-the-loop)
- **AI/Automatyzacja** — nowa warstwa: przetwarza maile, klasyfikuje wady, odpytuje SAP, generuje draft odpowiedzi i tworzy tickety
- **Systemy:** Exchange / Graph API, Azure Blob Storage, SAP ERP (PP/QM), JIRA Cloud (REK), PostgreSQL

---

## Przebieg procesu

### 1. Zgłoszenie reklamacji
- Klient wysyła e-mail na `reklamacje@metalpol.pl` ze zdjęciem wady, numerem zamówienia i opisem (PL lub EN).
- Exchange odbiera wiadomość i natychmiast triggeruje webhook przez Microsoft Graph API.

### 2. Automatyczne przetwarzanie przez AI
- Orchestrator odbiera webhook i uruchamia pipeline:
  - **Ekstrakcja danych (LLM)** — wyodrębnienie numeru zamówienia, opisu wady, języka maila, metadanych zdjęć.
  - **Upload zdjęć** — załączniki zapisywane do Azure Blob Storage z SAS token.
  - **Klasyfikacja wady (LLM)** — przypisanie kategorii: wizualna / wymiary / materiał / logistyka na podstawie opisu i zdjęć.
  - **Lookup w SAP** — automatyczne pobranie danych zamówienia i batcha (`GET /api/v1/orders/{id}`, `GET /api/v1/batches/{id}`).
  - **Lookup w PostgreSQL** — pobranie historii klienta i danych kontaktowych.

### 3. Automatyczne tworzenie ticketu w JIRA
- AI tworzy ticket (issue type: **Complaint**) w projekcie REK z uzupełnionymi polami: klient, zamówienie, batch, kategoria wady, opis, linki do zdjęć.
- Brak ręcznego przepisywania danych.

### 4. Generowanie draftu odpowiedzi
- LLM generuje draft e-maila do klienta w języku wykrytym z maila (PL lub EN), uwzględniając dane z SAP i klasyfikację wady.

### 5. Human-in-the-loop — weryfikacja przez specjalistę
- Specjalista serwisu otrzymuje powiadomienie z gotowym draftem odpowiedzi i podglądem ticketu JIRA.
- Zatwierdza lub koryguje klasyfikację i treść odpowiedzi.
- Czas zaangażowania specjalisty: minuty zamiast godzin.

### 6. Wysłanie odpowiedzi i zamknięcie pętli
- Zaakceptowany draft wysyłany do klienta przez Exchange/Outlook.
- Jeśli wada potwierdzona — AI automatycznie tworzy ticket korygujący (issue type: **Correction**) dla działu jakości.

---

## Problemy AS-IS → rozwiązania TO-BE

| Problem AS-IS | Rozwiązanie TO-BE |
|---|---|
| ~40% maili trafia do spamu | Webhook Graph API — mail przechwytywany natychmiast po wpłynięciu |
| Backlog 2–3 dni | Czas obsługi skrócony do minut — AI przetwarza równolegle |
| Niespójna kategoryzacja | LLM klasyfikuje według stałych reguł — spójność 100% |
| SAP i JIRA nie są zintegrowane | Orchestrator integruje oba systemy automatycznie |
| Ręczne przepisywanie danych | Ekstrakcja i zapis danych w pełni automatyczna |
| Brak metryk i KPI | Wszystkie zdarzenia logowane — dashboard z pełnymi metrykami |
| Brak wykorzystania bazy klientów | PostgreSQL odpytywana automatycznie przy każdej reklamacji |
| Brak danych do decyzji strategicznych | Spójne dane w JIRA umożliwiają analizę trendów i linii produkcyjnych |

---

## Korzyści CEO

- **0% maili pominiętych** — webhook zastępuje ręczne sprawdzanie skrzynki
- **Czas obsługi: 2 dni → &lt; 1 h** (czas specjalisty ograniczony do weryfikacji)
- **Spójna kategoryzacja** — podstawa rzetelnych KPI jakościowych
- **Automatyczne przepisywanie danych** — eliminacja błędów ludzkich
- **Pełne metryki procesowe** — widoczność dla management

# Przegląd systemu — komponenty i odpowiedzialności

## Diagram komponentów

```mermaid
graph TB
    subgraph Klient
        CL([Klient])
    end

    subgraph Microsoft_365
        EX[Exchange\nreklamacje@metalpol.pl]
        GR[Microsoft Graph API\nwebhook]
        OL[Outlook\nwysyłka odpowiedzi]
    end

    subgraph Azure
        SB[Azure Service Bus\nkolejka zdarzeń]
        CA[Orchestrator\nFastAPI / Container Apps]
        BS[Azure Blob Storage\nzdjęcia wad]
    end

    subgraph AI
        LLM[Claude Sonnet\nAnthropic API]
    end

    subgraph Systemy_wewnętrzne
        SAP[SAP ERP PP/QM\nREST API]
        PG[PostgreSQL\nbaza klientów]
        JI[JIRA Cloud\nprojekt REK]
    end

    subgraph Specjalista
        SP([Specjalista serwisu])
    end

    CL -->|e-mail z reklamacją| EX
    EX -->|webhook| GR
    GR -->|zdarzenie| SB
    SB -->|kolejka| CA
    CA -->|tekst maila + załączniki| LLM
    CA -->|upload zdjęć| BS
    CA -->|GET order + batch| SAP
    CA -->|GET historia klienta| PG
    LLM -->|JSON ekstrakcja + klasyfikacja + draft| CA
    CA -->|utwórz Complaint ticket| JI
    CA -->|powiadomienie z draftem| SP
    SP -->|zatwierdź / koryguj| CA
    CA -->|wyślij odpowiedź| OL
    OL -->|e-mail odpowiedź| CL
    CA -->|utwórz Correction ticket| JI
```

---

## Opis komponentów

### Exchange — `reklamacje@metalpol.pl`
Punkt wejścia dla wszystkich reklamacji. Każdy przychodzący mail triggeruje webhook przez Microsoft Graph API. Eliminuje problem spamu — zdarzenie jest rejestrowane natychmiast po wpłynięciu maila, niezależnie od folderu docelowego.

### Microsoft Graph API (webhook)
Subskrypcja na nowe wiadomości w skrzynce. Wysyła zdarzenie do Azure Service Bus w ciągu sekund od wpłynięcia maila. Autoryzacja przez OAuth2 (service account Metalpol).

### Azure Service Bus
Kolejka zdarzeń między webhookiem a orchestratorem. Gwarantuje dostarczenie zdarzenia nawet przy chwilowej niedostępności orchestratora. Kluczowy komponent przy obsłudze szczytu sezonowego (2000 reklamacji/miesiąc) — buforuje ruch zamiast przeciążać orchestrator.

### Orchestrator — FastAPI / Azure Container Apps
Centralny komponent systemu. Koordynuje cały pipeline: odbiera zdarzenia z Service Bus, wywołuje kolejne kroki (LLM, SAP, PostgreSQL, JIRA, Blob), obsługuje błędy i edge cases, wysyła powiadomienia do specjalisty. Skaluje automatycznie przy wzroście wolumenu.

### Claude Sonnet — Anthropic API
Warstwa AI odpowiedzialna za:
- **Ekstrakcję danych** z treści maila (numer zamówienia, opis wady, język, dane kontaktowe)
- **Klasyfikację wady** (wizualna / wymiary / materiał / logistyka)
- **Generowanie draftu odpowiedzi** w języku klienta (PL / EN)
- **Wykrywanie intencji** (czy mail to reklamacja, czy np. zapytanie ofertowe)

### Azure Blob Storage
Przechowywanie zdjęć wad przesłanych przez klientów. Orchestrator pobiera załączniki przez Graph API i uploaduje do Blob. Generowane SAS URL-e dołączane do ticketu JIRA i dostępne dla działu jakości.

### SAP ERP (PP/QM)
Źródło danych produkcyjnych. Orchestrator odpytuje dwa endpointy:
- `GET /api/v1/orders/{id}` — dane zamówienia
- `GET /api/v1/batches/{id}` — parametry batcha, operator, linia produkcyjna

Dane z SAP przekazywane do LLM jako kontekst przy klasyfikacji i drafcie odpowiedzi.

### PostgreSQL — baza klientów
Read-only. Dostarcza historię reklamacji klienta i dane kontaktowe. Kontekst używany przez LLM do personalizacji draftu odpowiedzi (~15k rekordów).

### JIRA Cloud — projekt REK
Orchestrator automatycznie tworzy:
- **Complaint** — przy każdej nowej reklamacji
- **Correction** — jeśli wada potwierdzona, dla działu jakości

Eliminuje ręczne tworzenie ticketów i przepisywanie danych z Excela.

### Specjalista serwisu
Pozostaje w pętli jako **human-in-the-loop**. Otrzymuje powiadomienie z gotowym draftem, klasyfikacją i podglądem ticketu. Jego rola zmienia się z operatora przepisującego dane na weryfikatora decyzji AI (~3 min na reklamację zamiast ~30 min).

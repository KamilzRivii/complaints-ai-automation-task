# Przepływ danych między systemami

## Diagram sekwencji

```mermaid
sequenceDiagram
    actor Klient
    participant EX as Exchange<br/>Graph API
    participant SB as Azure<br/>Service Bus
    participant OR as Orchestrator<br/>FastAPI
    participant LLM as Claude Sonnet<br/>Anthropic API
    participant BS as Azure<br/>Blob Storage
    participant SAP as SAP ERP<br/>PP/QM
    participant PG as PostgreSQL
    participant JI as JIRA Cloud
    actor SP as Specjalista

    Klient->>EX: E-mail z reklamacją (PL/EN)<br/>+ zdjęcia (1–3 pliki, 2–5 MB)

    EX->>OR: Webhook POST /webhook/complaint<br/>OAuth2 Bearer token

    OR->>SB: Publish zdarzenie<br/>(message_id, mail_id)
    OR-->>EX: 200 OK (natychmiastowo)

    SB->>OR: Consume zdarzenie z kolejki

    OR->>EX: GET /messages/{mail_id}<br/>pobierz treść + załączniki
    EX-->>OR: mail body + base64 attachments

    OR->>LLM: Prompt: ekstrakcja danych<br/>(treść maila, few-shot examples)
    LLM-->>OR: JSON {order_number, defect_description,<br/>language, customer_email}

    alt Brak numeru zamówienia
        OR->>SP: Powiadomienie: brak nr zamówienia
        OR->>EX: Wyślij e-mail do klienta<br/>z prośbą o numer zamówienia
    end

    OR->>BS: PUT załączniki → Blob Storage<br/>SAS token auth
    BS-->>OR: SAS URL[] dla każdego zdjęcia

    OR->>SAP: GET /api/v1/orders/{order_number}<br/>rate limit: 100 req/min
    SAP-->>OR: dane zamówienia (klient, data, produkt)

    OR->>SAP: GET /api/v1/batches/{batch_id}
    SAP-->>OR: dane batcha (operator, linia, parametry)

    OR->>PG: SELECT historia klienta<br/>WHERE email = {customer_email}
    PG-->>OR: historia reklamacji (read-only)

    OR->>LLM: Prompt: klasyfikacja wady<br/>(opis + dane SAP + few-shot examples)
    LLM-->>OR: {category: wizualna|wymiary|materiał|logistyka,<br/>confidence: float}

    OR->>LLM: Prompt: generuj draft odpowiedzi<br/>(język: PL|EN, dane zamówienia, klasyfikacja)
    LLM-->>OR: draft e-mail do klienta

    OR->>JI: POST /rest/api/3/issue<br/>issue type: Complaint<br/>fields: klient, zamówienie, batch,<br/>kategoria, opis, SAS URLs
    JI-->>OR: issue_key (REK-XXXX)

    OR->>SP: Powiadomienie (e-mail / Teams)<br/>draft odpowiedzi + klasyfikacja + link JIRA

    SP->>OR: Zatwierdź lub edytuj draft<br/>POST /review/{complaint_id}

    OR->>EX: Wyślij odpowiedź do klienta<br/>przez Graph API (send mail)
    EX->>Klient: E-mail odpowiedź

    alt Wada potwierdzona
        OR->>JI: POST /rest/api/3/issue<br/>issue type: Correction<br/>linked to REK-XXXX
        JI-->>OR: issue_key (REK-XXXX-C)
    end
```

---

## Kluczowe aspekty przepływu

### Asynchroniczność i niezawodność
Orchestrator odpowiada webhookowi natychmiast (`200 OK`), zanim rozpocznie jakiekolwiek przetwarzanie. Właściwy pipeline uruchamiany jest przez Azure Service Bus — gwarantuje dostarczenie zdarzenia nawet przy chwilowej awarii orchestratora (dead-letter queue po 3 nieudanych próbach).

### Rate limiting SAP
SAP ERP dopuszcza 100 req/min. Przy szczycie sezonowym (2000 reklamacji/miesiąc ≈ ~3 req/min średnio, ale lokalnie więcej) Service Bus naturalnie throttluje ruch. Orchestrator implementuje exponential backoff przy odpowiedzi `429 Too Many Requests`.

### OAuth2 — Microsoft Graph API
Orchestrator używa service account z zakresem `Mail.Read` + `Mail.Send`. Token odświeżany automatycznie przed wygaśnięciem (TTL: 1h). Brak interakcji użytkownika przy uwierzytelnianiu.

### Azure Blob Storage — SAS tokens
Zdjęcia uploadowane z unikalnym SAS URL ważnym 30 dni. URL dołączany do ticketu JIRA — dział jakości ma bezpośredni dostęp bez logowania do Blob Storage.

### PostgreSQL — read-only
Orchestrator łączy się jako użytkownik tylko do odczytu. Baza klientów nie jest modyfikowana przez pipeline — zapis danych reklamacji odbywa się wyłącznie w JIRA.

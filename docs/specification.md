# Specyfikacja rozwiązania — Automatyzacja obsługi reklamacji Metalpol

## 1. Problem statement

Metalpol przetwarza ~600 reklamacji miesięcznie (szczyt: 2000/miesiąc) w pełni ręcznie. Jeden specjalista obsługuje do 80 zgłoszeń dziennie, przepisując dane między Exchange, Excelem, SAP i JIRA. Skutkuje to backlogiem 2–3 dni, niespójną kategoryzacją wad i brakiem jakichkolwiek metryk procesowych. Szczegóły: [Kontekst biznesowy](business-context.md), [Opis procesu AS-IS](as-is-process.md).

---

## 2. Proponowane rozwiązanie

Event-driven pipeline AI, który przechwytuje każdy przychodzący mail reklamacyjny przez webhook, przetwarza go automatycznie (ekstrakcja, klasyfikacja, lookup w SAP, draft odpowiedzi, ticket JIRA) i przekazuje specjaliście wyłącznie do zatwierdzenia (**human-in-the-loop**).

---

## 3. Stos technologiczny

| Warstwa | Technologia | Uzasadnienie |
|---|---|---|
| Trigger | Microsoft Graph API (webhook) | Natychmiastowe przechwycenie maila — eliminacja problemu spamu |
| Kolejka | Azure Service Bus | Buforowanie przy szczytach (2000/mies.), retry logic, dead-letter queue |
| Orchestrator | Python + FastAPI na Azure Container Apps | Lekki, skalowalny, łatwa integracja z Azure ekosystemem |
| LLM | Claude claude-sonnet-4-6 (Anthropic API) | Najlepsza jakość ekstrakcji ustrukturyzowanych danych, wsparcie PL/EN, niezawodny JSON output |
| Storage | Azure Blob Storage (istniejący) | Zdjęcia wad — SAS tokens już skonfigurowane |
| Baza klientów | PostgreSQL (istniejący, read-only) | Historia klienta przy każdej reklamacji |
| ERP | SAP REST API (PP/QM) | Dane zamówień i batchów produkcyjnych |
| Ticketing | JIRA Cloud REST API | Automatyczne tworzenie Complaint i Correction |
| Powiadomienia | Microsoft Graph API (mail/Teams) | Notyfikacja specjalisty z draftem do zatwierdzenia |

---

## 4. Flow automatyzacji

```
[Exchange] → webhook → [Azure Service Bus]
                              ↓
                      [Orchestrator / FastAPI]
                              ↓
              ┌───────────────────────────────┐
              │        AI Pipeline            │
              │  1. Ekstrakcja danych (LLM)   │
              │  2. Upload zdjęć → Blob       │
              │  3. Lookup SAP (order+batch)  │
              │  4. Lookup PostgreSQL         │
              │  5. Klasyfikacja wady (LLM)   │
              │  6. Draft odpowiedzi (LLM)    │
              │  7. Utwórz ticket JIRA        │
              └───────────────────────────────┘
                              ↓
              [Powiadomienie → Specjalista serwisu]
                              ↓
                    [Human-in-the-loop]
                    Zatwierdź / Koryguj
                              ↓
              ┌───────────────────────────────┐
              │  Wyślij odpowiedź → Klient    │
              │  (opcjonalnie) Correction     │
              │  ticket → Dział jakości       │
              └───────────────────────────────┘
```

### Krok 1 — Ekstrakcja danych (LLM)
LLM otrzymuje treść maila i wyodrębnia ustrukturyzowany JSON:
```json
{
  "order_number": "string | null",
  "defect_description": "string",
  "language": "PL | EN",
  "customer_email": "string",
  "attachment_count": "int"
}
```
Prompt zawiera instrukcję zwrotu `null` gdy pole nie jest dostępne — orchestrator reaguje na brakujące dane zgodnie z polityką edge-case.

### Krok 2 — Upload zdjęć
Załączniki pobierane przez Graph API i uploadowane do Azure Blob Storage. Generowane SAS URL-e dołączane do ticketu JIRA.

### Krok 3 — Lookup SAP
```
GET /api/v1/orders/{order_number}
GET /api/v1/batches/{batch_id}
```
Rate limit: 100 req/min — przy szczytach kolejkowanie przez Azure Service Bus zapobiega przeciążeniu.

### Krok 4 — Lookup PostgreSQL
Pobranie historii reklamacji klienta i danych kontaktowych (read-only). Kontekst przekazywany do LLM przy generowaniu draftu.

### Krok 5 — Klasyfikacja wady (LLM)
LLM klasyfikuje wadę na podstawie opisu i metadanych SAP (parametry batcha, operator, linia produkcyjna):
- `wizualna` / `wymiary` / `materiał` / `logistyka`

Klasyfikacja spójna — wywołanie API z `temperature=0` minimalizuje losowość modelu. Ten sam opis wady → ta sama kategoria. Eliminuje niespójność między specjalistami.

### Krok 6 — Draft odpowiedzi (LLM)
LLM generuje draft e-maila do klienta w wykrytym języku (PL/EN), uwzględniając:
- dane zamówienia i batcha z SAP
- kategorię wady
- historię klienta z PostgreSQL
- ton formalny zgodny z branżą automotive

### Krok 7 — Ticket JIRA
Orchestrator tworzy ticket (issue type: **Complaint**) z uzupełnionymi polami: klient, zamówienie, batch, kategoria, opis, SAS URL zdjęć.

---

## 5. Human-in-the-loop

Specjalista otrzymuje powiadomienie (e-mail lub Teams) z:
- podglądem danych wyekstrahowanych przez AI
- proponowaną klasyfikacją wady
- draftem odpowiedzi do klienta
- linkiem do ticketu JIRA

Może zatwierdzić jednym kliknięciem lub edytować dowolne pole. Po zatwierdzeniu — odpowiedź wysyłana automatycznie, ticket aktualizowany.

**Dlaczego human-in-the-loop?** Decyzja o uznaniu reklamacji ma skutki prawne i finansowe. AI może się mylić przy niestandardowych przypadkach. Specjalista pozostaje odpowiedzialny — AI redukuje jego czas z ~30 min do ~3 min na reklamację.

---

## 6. Edge cases

| Przypadek | Obsługa |
|---|---|
| Brak numeru zamówienia w mailu | AI flaguje → specjalista dostaje powiadomienie z prośbą o uzupełnienie; mail do klienta z prośbą o numer |
| Nieczytelne / brak zdjęć | Pipeline kontynuuje bez analizy obrazu; klasyfikacja na podstawie opisu tekstowego |
| SAP nie zwraca danych dla zamówienia | Orchestrator flaguje pole jako `not_found`; specjalista weryfikuje ręcznie |
| Duplikat reklamacji (ten sam numer zamówienia) | Deduplication na podstawie `order_number` + okno 24h — scalenie z istniejącym ticketem |
| Wykroczenie poza rate limit SAP (100 req/min) | Exponential backoff + retry; przy szczycie — throttling przez Service Bus |
| Nieznany język maila | Fallback do PL; specjalista może zmienić język draftu |
| Mail nie jest reklamacją (spam, zapytanie ofertowe) | LLM klasyfikuje intent → jeśli nie-reklamacja: brak ticketu, powiadomienie do specjalisty |
| Bardzo długi mail (> token limit) | Chunking + summarization przed właściwą ekstrakcją |

---

## 7. Kluczowe decyzje i trade-offy

### LLM: Claude vs GPT-4o vs open-source
**Decyzja:** Claude claude-sonnet-4-6
- Najlepsza jakość ustrukturyzowanego JSON output (function calling / tool use)
- Dobre wsparcie języka polskiego
- Koszt akceptowalny przy 600–2000 reklamacji/miesiąc (szacunek: ~$50–200/mies.)
- **Trade-off:** vendor lock-in Anthropic; alternatywnie GPT-4o (podobna jakość, większy ekosystem)

### Serverless vs dedykowany serwis
**Decyzja:** Azure Container Apps (zawsze włączony, min. 1 instancja)
- Azure Functions (serverless) tańsze przy niskim wolumenie, ale cold start ~2s nieakceptowalny dla webhooków
- Container Apps skaluje automatycznie przy szczytach sezonowych
- **Trade-off:** wyższy koszt bazowy vs niezawodność

### Pełna automatyzacja vs human-in-the-loop
**Decyzja:** Human-in-the-loop (weryfikacja przed wysłaniem)
- Reklamacje mają skutki prawne — błąd AI może kosztować więcej niż oszczędność
- 85% reklamacji AI rozstrzygnie poprawnie na podstawie SAP; specjalista zatwierdza w ~3 min
- **Trade-off:** nie eliminujemy specjalisty całkowicie — redukujemy jego obciążenie o ~90%

### Prompt engineering vs fine-tuning
**Decyzja:** Prompt engineering (MVP)
- Fine-tuning wymaga dużego zbioru danych historycznych i znaczącego budżetu
- Prompt engineering z kilkoma przykładami (few-shot) daje >90% accuracy przy klasyfikacji wad
- **Trade-off:** fine-tuning warto rozważyć po 6 miesiącach zebrania danych produkcyjnych

### Synchroniczne vs asynchroniczne przetwarzanie
**Decyzja:** Asynchroniczne (Azure Service Bus)
- Webhook odpowiada natychmiast (200 OK) — Exchange nie retryuje
- Przetwarzanie AI trwa 5–15s — nie może blokować webhooka
- Service Bus gwarantuje dostarczenie przy awarii orchestratora
- **Trade-off:** złożoność infrastruktury vs niezawodność

---

## 8. Metryki sukcesu

| Metryka | AS-IS | Cel TO-BE |
|---|---|---|
| % maili przechwyconych | ~60% | 100% |
| Średni czas odpowiedzi | 2 dni | < 2 h |
| Backlog | 2–3 dni | 0 |
| Spójność kategoryzacji | ~60% | > 95% |
| Czas specjalisty na reklamację | ~30 min | < 5 min |
| Reklamacje obsługiwane przez 1 specjalistę/dzień | 30 (szczyt 80) | 200+ |
| KPI dostępne dla management | brak | pełny dashboard |

---

## 9. Interfejs specjalisty (human-in-the-loop)

Specjalista nie korzysta z żadnej nowej aplikacji — powiadomienie trafia na jego skrzynkę Exchange lub kanał Teams i zawiera:

- podsumowanie reklamacji (klient, numer zamówienia, kategoria wady, dane batcha z SAP)
- draft odpowiedzi do klienta gotowy do wysłania
- dwa linki akcji: **[Zatwierdź i wyślij]** / **[Edytuj w panelu]**

**Zatwierdzenie** — kliknięcie w link wywołuje endpoint orchestratora (`POST /review/{complaint_id}?action=approve`) z tokenem jednorazowym. Orchestrator wysyła odpowiedź do klienta i aktualizuje ticket JIRA. Zero nowego UI.

**Edycja** — link otwiera prosty panel webowy (hostowany przez orchestrator, FastAPI + Jinja2) z formularzem: pole draftu odpowiedzi, dropdown kategorii wady, przycisk "Wyślij". Minimalne UI — jedna strona, brak frameworku frontendowego.

**Dlaczego nie JIRA jako interfejs?** JIRA nie obsługuje wysyłki maili do klientów bezpośrednio. Dodanie akcji w JIRA wymagałoby Forge app — zbędna złożoność na MVP.

---

## 10. Observability i obsługa błędów

### Logowanie
Każde zdarzenie w pipeline zapisywane do Azure Application Insights:
- czas przetwarzania każdego kroku (ekstrakcja, SAP lookup, klasyfikacja, JIRA)
- wynik klasyfikacji wady + confidence score
- czy specjalista zatwierdził bez zmian czy edytował draft

### Alerty
- dead-letter queue w Service Bus niepusta → alert do zespołu technicznego
- błąd wywołania LLM (5xx Anthropic API) → retry x3, potem fallback do ręcznej obsługi
- SAP niedostępny > 5 min → alert + kolejkowanie requestów

### Dashboardy (Azure Monitor)
- liczba reklamacji dziennie / tygodniowo
- rozkład kategorii wad (wizualna / wymiary / materiał / logistyka)
- % przypadków gdzie specjalista edytował draft AI (wskaźnik jakości modelu)
- średni czas od wpłynięcia maila do wysłania odpowiedzi

### Dead-letter queue
Wiadomości z Service Bus po 3 nieudanych próbach trafiają do dead-letter queue. Orchestrator wysyła alert i tworzy w JIRA ticket z flagą `requires_manual_processing` — specjalista obsługuje ręcznie jak w AS-IS.

---

## 11. Bezpieczeństwo

### Zarządzanie secretami
Wszystkie klucze API (Anthropic, JIRA, SAP, Graph API) przechowywane w **Azure Key Vault**. Orchestrator pobiera je przy starcie przez Managed Identity — brak secretów w kodzie, w zmiennych środowiskowych ani w repozytorium.

### Autoryzacja Microsoft Graph API
Service account z zakresem minimalnym: `Mail.Read` + `Mail.Send` na skrzynce `reklamacje@metalpol.pl`. Token OAuth2 odświeżany automatycznie. Brak dostępu do innych skrzynek Exchange.

### Linki akcji specjalisty (approve/reject)
Tokeny jednorazowe z TTL 48h, podpisane HMAC. Po użyciu lub wygaśnięciu — nieważne. Zapobiega wielokrotnemu zatwierdzeniu tej samej reklamacji.

### GDPR
- Dane klientów z PostgreSQL używane wyłącznie jako kontekst dla LLM — nie są logowane ani przechowywane poza pipelinemem
- Zdjęcia wad w Azure Blob Storage z retencją 90 dni (konfigurowalna przez lifecycle policy)
- Treści maili nie są przechowywane przez orchestrator — przetwarzane w pamięci i odrzucane po zakończeniu pipeline'u

### Komunikacja z SAP
Wywołania SAP REST API przez sieć wewnętrzną (VNet peering między Azure Container Apps a SAP) — nie przez publiczny internet.

---

## 12. Co byłoby zmienione przy ograniczonym budżecie

1. **Rezygnacja z Azure Service Bus** → synchroniczne przetwarzanie z retry w orchestratorze (ryzyko utraty zdarzeń przy awarii)
2. **GPT-3.5-turbo zamiast Claude claude-sonnet-4-6** → niższy koszt, gorsza jakość ekstrakcji PL
3. **Azure Functions zamiast Container Apps** → cold start ~2s, ale znacząco tańsze
4. **Brak PostgreSQL lookup** → mniejszy kontekst dla LLM, prostszy pipeline
5. **E-mail zamiast Teams do powiadomień specjalisty** → brak dodatkowych integracji

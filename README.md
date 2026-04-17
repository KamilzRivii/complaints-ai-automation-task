# Metalpol — Automatyzacja Obsługi Reklamacji AI

> Analiza i projekt automatyzacji procesu reklamacyjnego dla Metalpol Sp. z o.o.

## Nawigacja

| Dokument | Opis |
|---|---|
| [Kontekst biznesowy](docs/business-context.md) | Opis firmy, problemy CEO, aktorzy systemu |
| [Event Storming AS-IS](diagrams/as-is/event-storming.md) | Obecny proces obsługi reklamacji |
| [Opis procesu AS-IS](docs/as-is-process.md) | Szczegółowy opis diagramu AS-IS — aktorzy, kroki, hot spoty |
| [Event Storming TO-BE](diagrams/to-be/event-storming.md) | Proces po wprowadzeniu automatyzacji AI |
| [Opis procesu TO-BE](docs/to-be-process.md) | Szczegółowy opis diagramu TO-BE — AI pipeline, human-in-the-loop, korzyści |
| [Specyfikacja rozwiązania](docs/specification.md) | Architektura, integracje, stos technologiczny, trade-offy |
| [Przegląd systemu](architecture/system-overview.md) | Komponenty systemu i ich odpowiedzialności |
| [Przepływ danych](architecture/data-flow.md) | Sekwencja danych między systemami |

## Struktura repozytorium

```
complaints-ai-automation-Metalpol/
├── README.md
├── diagrams/
│   ├── as-is/          # Event Storming — stan obecny
│   └── to-be/          # Event Storming — stan docelowy
├── docs/               # Specyfikacja rozwiązania
└── architecture/       # Diagramy architektury i przepływu danych
```

# Attraction Service – Dokumentacja

> Mikroserwis do zarządzania atrakcjami turystycznymi, zbudowany zgodnie z zasadami **Domain-Driven Design (DDD)**.  
> Spring Boot 3.2 · Java 17 · In-Memory repositories · REST API

---

## Spis treści

1. [Opis systemu](#1-opis-systemu)
2. [Architektura – moduły](#2-architektura--moduły)
3. [Diagram modułów](#3-diagram-modułów)
4. [Moduł: attraction](#4-moduł-attraction)
5. [Moduł: catalog](#5-moduł-catalog)
6. [Moduł: availability](#6-moduł-availability)
7. [Moduł: plan](#7-moduł-plan)
8. [Wzorce projektowe](#8-wzorce-projektowe)
9. [Domain Events](#9-domain-events)
10. [REST API – przegląd endpointów](#10-rest-api--przegląd-endpointów)
11. [Uruchomienie](#11-uruchomienie)
12. [Przykładowe dane startowe](#12-przykładowe-dane-startowe)

---

## 1. Opis systemu

System udostępnia atrakcje turystyczne (muzea, parki, rejsy, eventy, wycieczki i dowolne inne typy) przez REST API. Kluczowe założenia:

- **Nowe atrakcje bez redeploymentu** – typ atrakcji to zwykły `String`, dodanie nowego typu nie wymaga zmian w kodzie
- **Generyczność** – jeden byt `Attraction` obsługuje wszystkie typy, brak hierarchii dziedziczenia po typie
- **Separacja definicji od rezerwacji** – moduł `attraction` nie wie nic o biletach; moduł `availability` nie wie nic o opisach
- **Sezonowość** – katalog letni różni się od zimowego; plan może zawierać tylko atrakcje z katalogu odpowiedniego sezonu
- **Brak bazy danych** – prototyp używa repozytoriów in-memory

---

## 2. Architektura – moduły

System składa się z **jednego mikroserwisu** podzielonego na **4 moduły domenowe**. Każdy moduł ma wewnętrzną strukturę trzech warstw:

```
moduł/
├── domain/       ← agregaty, value objects, interfejsy repozytoriów, domain events, specyfikacje
├── application/  ← serwisy, implementacje repozytoriów, listenery eventów
└── api/          ← kontrolery REST, DTO
```

### Zależności między modułami

```
attraction  ──►  catalog
attraction  ──►  availability
attraction  ──►  plan
catalog     ──►  plan      (plan może zawierać tylko atrakcje z katalogu)
availability ──► plan      (sprawdzanie sezonu przy dodawaniu do planu)
```

Moduły komunikują się wyłącznie **przez identyfikatory** (String UUID) i **domain events** – nigdy nie przekazują referencji do agregatów innych modułów.

---

## 3. Diagram modułów

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          attraction module                               │
│                                                                          │
│  ┌─────────────────────────────────────────────┐  ┌──────────────────┐  │
│  │                   domain                    │  │   application    │  │
│  │                                             │  │                  │  │
│  │  ┌────────────┐  ┌──────────────────────┐  │  │  AttractionSvc   │  │
│  │  │ Attraction │  │  AttractionArchetype  │  │  │  EventGroupSvc   │  │
│  │  │ (aggregate)│  │  (aggregate)          │  │  │  ArchetypeSvc    │  │
│  │  └────────────┘  └──────────────────────┘  │  │  DomainEvent     │  │
│  │                                             │  │  Publisher       │  │
│  │  ┌──────────────┐  ┌──────────────────┐    │  │  @EventListener  │  │
│  │  │  Attraction  │  │  AttractionGroup │    │  └──────────────────┘  │
│  │  │   Instance   │  │  (aggregate)     │    │                        │
│  │  └──────────────┘  └──────────────────┘    │  ┌──────────────────┐  │
│  │                                             │  │       api        │  │
│  │  ┌────────────────────────────────────┐     │  │  Controllers     │  │
│  │  │  EventGroup (aggregate)            │     │  │  DTOs            │  │
│  │  │  tylko type=EVENT, daty, harmonogram│    │  └──────────────────┘  │
│  │  └────────────────────────────────────┘     │                        │
│  │                                             │                        │
│  │  ┌────────────────────────────────────┐     │                        │
│  │  │  Specification<T>  (interface)     │     │                        │
│  │  │  IsPublishedSpec                   │     │                        │
│  │  │  HasOptionsInSeasonSpec            │     │                        │
│  │  │  IsInCatalogSpec                   │     │                        │
│  │  │  NotIncompatibleWithSpec           │     │                        │
│  │  │  PlanConfirmationSpecification     │     │                        │
│  │  └────────────────────────────────────┘     │                        │
│  └─────────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
            │ domain events           │ AttractionRepository
            ▼                         ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│   catalog module  │    │availability module │    │    plan module    │
│                   │    │                   │    │                   │
│  domain:          │    │  domain:          │    │  domain:          │
│  ┌─────────────┐  │    │  ┌─────────────┐  │    │  ┌─────────────┐  │
│  │   Catalog   │  │    │  │ Availability│  │    │  │    Plan     │  │
│  │  (aggregate)│  │    │  │    Slot     │  │    │  │ (aggregate) │  │
│  └─────────────┘  │    │  │ (aggregate) │  │    │  └─────────────┘  │
│                   │    │  └─────────────┘  │    │                   │
│  season: Season   │    │  HasCapacitySpec  │    │  tylko powiększ.  │
│  ACTIVE/ARCHIVED  │    │  IsOpenAtTimeSpec │    │  PlanEntry[]      │
│                   │    │  OpeningHours     │    │                   │
│  application:     │    │  NO OVERBOOKING   │    │  application:     │
│  CatalogService   │    │                   │    │  PlanService      │
│  CatalogEvent     │    │  application:     │    │  używa Specyfikacji│
│  Listeners        │    │  AvailabilitySvc  │    │                   │
│  auto-remove on   │    │  @EventListeners  │    │  api:             │
│  WITHDRAWN event  │    │                   │    │  PlanController   │
│                   │    │  api:             │    │                   │
│  api:             │    │  AvailCtrl        │    └───────────────────┘
│  CatalogCtrl      │    └───────────────────┘
└───────────────────┘
```

---

## 4. Moduł: attraction

Centralny moduł systemu. Zarządza definicjami, tożsamością i cyklem życia atrakcji.

### Agregaty i encje

#### `Attraction` – Aggregate Root

Główny byt domenowy. Przechodzi przez **state machine**:

```
DRAFT
  │  describe(description, tags)          → opis i tagi zamrożone
  ▼
DESCRIBED
  │  setRequirements(requirements)        → wymagania zamrożone
  ▼
WITH_REQUIREMENTS
  │  markReady()                          → wymaga min. 1 opcji
  ▼
READY
  │  publish()
  ▼
PUBLISHED ──── withdraw(reason) ──► WITHDRAWN
```

**Niezmienność pól:**

| Pole | Kiedy ustawiane | Mechanizm niezmienności |
|------|----------------|------------------------|
| `id` | konstruktor | `final` |
| `name` | konstruktor | `final` |
| `type` | konstruktor | `final` |
| `description`, `tags` | `describe()` | rzuca wyjątek poza DRAFT |
| `requirements` | `setRequirements()` | rzuca wyjątek poza DESCRIBED, zwraca `unmodifiableSet` |

**Relacje między atrakcjami:**

| Typ relacji | Znaczenie | Egzekwowanie |
|-------------|-----------|--------------|
| `REQUIRES` | A wymaga B – nie można potwierdzić planu bez B | przy `plan.confirm()` |
| `RECOMMENDED` | zalecane razem | informacyjnie |
| `INCOMPATIBLE` | A i B nie mogą być razem | przy `addAttraction()` do planu |

**Opcje atrakcji (`AttractionOption`)** – value object:
- każda atrakcja może mieć wiele opcji (standard, VIP, dla niewidomych, dla dzieci...)
- każda opcja ma własne: `price`, `maxCapacity`, `availableSeasons: Set<Season>`, `openingHours: OpeningHours`

#### `AttractionArchetype` – Aggregate Root

Szablon/przepis atrakcji. Wzorzec Archetype:

```
DRAFT  →  seal()  →  SEALED
             │
             └──► instantiate(label, mandatory) → AttractionInstance
```

Po `seal()` tożsamość (nazwa, typ, opis, wymagania) jest **trwale niezmieniona**. Z jednego archetypu można tworzyć wiele instancji np. `"Muzeum Narodowe – Lato 2025"` i `"Muzeum Narodowe – Zima 2025"`.

#### `AttractionInstance` – Entity

Konkretna instancja stworzona z archetypu. Ma pełną konfigurację: opcje, sezonowość, godziny, ceny. Przechodzi przez własny cykl: `DRAFT → CONFIGURED → PUBLISHED → WITHDRAWN`.

#### `AttractionGroup` – Aggregate Root

Pakiet atrakcji które muszą lub mogą występować razem:

```java
group.addMember(attractionId, MemberRole.REQUIRED);   // obowiązkowy
group.addMember(attractionId, MemberRole.OPTIONAL);   // opcjonalny
group.publish();
```

Dodanie grupy do planu jednym requestem: `POST /api/plans/{id}/groups`.  
Przy `confirm()` sprawdzane jest czy wszyscy `REQUIRED` członkowie są w planie.

#### `EventGroup` – Aggregate Root

Specjalizacja grupy wyłącznie dla atrakcji typu `EVENT`. Reprezentuje festiwal, serię koncertów itp.

```
EventGroup: "Jazz Summer Festival 2025"  (1–7 lipca)
  ├── slot 1: "Koncert Otwarcia"    – 1 lipca, required=true,  order=1
  └── slot 2: "Wielki Finał"        – 7 lipca, required=true,  order=2
```

Walidacje: data eventu musi mieścić się w zakresie grupy, tylko `type=EVENT`, min. 1 REQUIRED przy publikacji.

### Value Objects

| Klasa | Opis |
|-------|------|
| `AttractionOption` | Opcja atrakcji z ceną, pojemnością, sezonem, godzinami |
| `OpeningHours` | `opens: LocalTime`, `closes: LocalTime` – konfigurowalne per opcja |
| `Season` | `SPRING`, `SUMMER`, `AUTUMN`, `WINTER`, `ALL_YEAR` |
| `RelationType` | `REQUIRES`, `RECOMMENDED`, `INCOMPATIBLE` |

### Specyfikacje domenowe (`Specification<T>`)

Generyczny interfejs z kompozycją `and()`, `or()`, `not()`:

```java
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);
    String describe();
    default Specification<T> and(Specification<T> other) { ... }
    default Specification<T> or(Specification<T> other)  { ... }
    default Specification<T> not()                        { ... }
}
```

| Specyfikacja | Typ parametru | Zastosowanie |
|--------------|---------------|--------------|
| `IsPublishedSpec` | `Attraction` | CatalogService – tylko opublikowane do katalogu |
| `HasOptionsInSeasonSpec(season)` | `Attraction` | Katalog i plan – atrakcja musi mieć opcje w sezonie |
| `IsInCatalogSpec(ids)` | `Attraction` | PlanService – atrakcja musi być w katalogu planu |
| `NotIncompatibleWithSpec(ids, repo)` | `Attraction` | PlanService – sprawdza obie strony relacji INCOMPATIBLE |
| `PlanNotEmptySpec` | `PlanConfirmationContext` | Plan nie może być pusty |
| `RequiresRelationsSatisfiedSpec` | `PlanConfirmationContext` | Wszystkie REQUIRES spełnione |
| `GroupsCompleteSpec` | `PlanConfirmationContext` | Wszyscy REQUIRED z grup w planie |
| `PlanConfirmationSpecification` | fasada | Zbiera wszystkie naruszenia naraz |

---

## 5. Moduł: catalog

Zarządza sezonowymi katalogami ofert.

### Agregat `Catalog`

```
ACTIVE  →  archive()  →  ARCHIVED
```

Katalog przechowuje:
- `season: Season` – do której pory roku katalog należy (SUMMER, WINTER itd.)
- `attractionIds: Set<String>` – referencje do opublikowanych atrakcji
- `groupIds: Set<String>` – referencje do opublikowanych grup

**Reguły:**
- Do katalogu można dodać tylko atrakcje ze statusem `PUBLISHED`
- Atrakcja musi mieć opcje dostępne w sezonie katalogu (`HasOptionsInSeasonSpec`)
- Dodanie grupy auto-dodaje jej wymaganych (`REQUIRED`) członków do katalogu
- Po archiwizacji katalog nie może być modyfikowany

### CatalogEventListeners

Reaguje na domain events z modułu `attraction`:

```
AttractionWithdrawnEvent  →  auto-usuwa atrakcję ze wszystkich ACTIVE katalogów
AttractionPublishedEvent  →  loguje ile aktywnych katalogów jest dostępnych
```

---

## 6. Moduł: availability

Zarządza dostępnością i rezerwacjami. **Oddzielony od definicji atrakcji.**

### Agregat `AvailabilitySlot`

Reprezentuje pulę biletów dla konkretnej **atrakcji + opcji + daty**:

```
createSlot(attractionId, optionId, date, capacity)
    └── kopiuje OpeningHours z AttractionOption (lub override w request)

reserve(count, visitTime)
    ├── HasCapacitySpec     → brak overbookingu
    └── IsOpenAtTimeSpec    → wizyta musi być w godzinach otwarcia

cancel(count)
    └── zwalnia miejsca
```

**Godziny otwarcia są konfigurowalne** na dwóch poziomach:
1. `AttractionOption.openingHours` – domyślne godziny opcji
2. `CreateSlotRequest.opensAt/closesAt` – nadpisanie dla konkretnego dnia (np. Noc Muzeów)

### Domain Events availability

| Event | Kiedy | Dane |
|-------|-------|------|
| `SlotCreatedEvent` | nowy slot | attractionId, optionId, date, capacity |
| `SlotReservedEvent` | rezerwacja | count, remainingCapacity |
| `SlotFullyBookedEvent` | ostatnie miejsce zajęte | slotId, attractionId |
| `SlotCancelledEvent` | anulowanie | count, remainingCapacity |

### AvailabilityEventListeners

| Event | Reakcja |
|-------|---------|
| `SlotReservedEvent` | WARNING gdy pozostało ≤ 3 miejsca |
| `SlotFullyBookedEvent` | alert – brak wolnych miejsc |

---

## 7. Moduł: plan

Zarządza planami wycieczek klientów.

### Agregat `Plan`

```
DRAFT  →  confirm()  →  CONFIRMED (niemodyfikowalny)
```

**Plan może być tylko powiększany** – brak metody `removeAttraction()`. Po `confirm()` żadna zmiana nie jest możliwa.

`PlanEntry` przechowuje: `attractionId`, `optionId`, `mandatory`, `groupId` (null jeśli dodano indywidualnie).

### Walidacje przy `addAttraction()`

Sprawdzane przez specyfikacje w kolejności:

```
1. IsInCatalogSpec           → atrakcja musi być w katalogu planu
2. HasOptionsInSeasonSpec    → atrakcja musi mieć opcje w sezonie katalogu
3. NotIncompatibleWithSpec   → brak relacji INCOMPATIBLE (obie strony)
```

### Walidacje przy `confirm()`

`PlanConfirmationSpecification` zbiera **wszystkie naruszenia naraz**:

```
PlanNotEmptySpec                  → plan nie może być pusty
RequiresRelationsSatisfiedSpec    → jeśli A REQUIRES B, to B musi być w planie
GroupsCompleteSpec                → wszyscy REQUIRED z grup muszą być w planie
```

---

## 8. Wzorce projektowe

| Wzorzec | Gdzie | Opis |
|---------|-------|------|
| **Aggregate Root** | `Attraction`, `Catalog`, `AvailabilitySlot`, `Plan`, `AttractionGroup`, `EventGroup` | Kontroluje dostęp do wewnętrznych obiektów, emituje domain events |
| **State Machine** | `Attraction.Status` | DRAFT → DESCRIBED → WITH_REQUIREMENTS → READY → PUBLISHED → WITHDRAWN |
| **Specification Pattern** | `Specifications.java`, `AvailabilitySpecifications.java` | `Specification<T>` z `and()`, `or()`, `not()` – reguły biznesowe jako obiekty |
| **Repository Pattern** | `AttractionRepository`, `AvailabilitySlotRepository` | Interfejsy w domenie, implementacje in-memory w warstwie application |
| **Domain Events** | `AttractionEvents`, `AvailabilityEvents`, `EventGroupEvents` | Zdarzenia domenowe publikowane przez Spring `ApplicationEventPublisher` |
| **Archetype Pattern** | `AttractionArchetype` → `AttractionInstance` | Szablon (przepis) zamrożony przez `seal()`, wiele instancji z różną konfiguracją |
| **Value Object** | `AttractionOption`, `OpeningHours`, `Season`, `PlanEntry` | Niemutowalne obiekty bez własnej tożsamości |
| **Observer / @EventListener** | `AttractionEventListeners`, `CatalogEventListeners`, `AvailabilityEventListeners`, `EventGroupListeners` | Luźne sprzężenie między modułami przez eventy |

---

## 9. Domain Events

Przepływ eventów przez system:

```
Attraction.publish()
    └──► AttractionPublishedEvent
              ├──► AttractionEventListeners.onAttractionPublished()
              │      → loguje że atrakcja gotowa do katalogu
              └──► CatalogEventListeners.onAttractionPublished()
                     → loguje liczbę aktywnych katalogów

Attraction.withdraw(reason)
    └──► AttractionWithdrawnEvent
              ├──► AttractionEventListeners.onAttractionWithdrawn()
              │      → sprawdza czy atrakcja była w grupach
              ├──► CatalogEventListeners.onAttractionWithdrawn()
              │      → AUTO-USUWA z wszystkich ACTIVE katalogów
              └──► EventGroupListeners.onAttractionWithdrawn()
                     → ostrzega o niekompletnych EventGroup

AvailabilitySlot.reserve(count, visitTime)
    └──► SlotReservedEvent
              └──► AvailabilityEventListeners.onSlotReserved()
                     → WARNING jeśli remaining <= 3
         SlotFullyBookedEvent (jeśli brak miejsc)
              └──► AvailabilityEventListeners.onSlotFullyBooked()
                     → ALERT brak wolnych miejsc

AttractionArchetype.seal()
    └──► ArchetypeSealedEvent
              └──► AttractionEventListeners.onArchetypeSealed()

AttractionArchetype.instantiate()
    └──► ArchetypeInstantiatedEvent
              └──► AttractionEventListeners.onArchetypeInstantiated()

EventGroup.publish()
    └──► EventGroupPublishedEvent
              └──► EventGroupListeners.onEventGroupPublished()

EventGroup.cancel()
    └──► EventGroupCancelledEvent
              └──► EventGroupListeners.onEventGroupCancelled()
```

---

## 10. REST API – przegląd endpointów

### Atrakcje `/api/attractions`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/attractions` | Utwórz nową atrakcję |
| GET | `/api/attractions` | Lista wszystkich |
| GET | `/api/attractions?status=PUBLISHED` | Filtruj po statusie |
| GET | `/api/attractions?season=SUMMER` | Filtruj po sezonie |
| GET | `/api/attractions/{id}` | Pobierz jedną |
| PATCH | `/api/attractions/{id}/describe` | Dodaj opis i tagi (DRAFT) |
| PATCH | `/api/attractions/{id}/requirements` | Ustaw wymagania (DESCRIBED) |
| PATCH | `/api/attractions/{id}/ready` | Oznacz jako gotową |
| PATCH | `/api/attractions/{id}/publish` | Opublikuj |
| PATCH | `/api/attractions/{id}/withdraw` | Wycofaj |
| POST | `/api/attractions/{id}/options` | Dodaj opcję |
| POST | `/api/attractions/{id}/relations` | Dodaj relację |

### Archetypy i instancje `/api/archetypes`, `/api/instances`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/archetypes` | Utwórz archetype |
| PATCH | `/api/archetypes/{id}/describe` | Dodaj opis i wymagania |
| PATCH | `/api/archetypes/{id}/seal` | Zapieczętuj (zamroź tożsamość) |
| POST | `/api/archetypes/instantiate` | Stwórz instancję z archetypu |
| GET | `/api/archetypes/{id}/instances` | Lista instancji archetypu |
| GET | `/api/instances?season=SUMMER` | Instancje dostępne w sezonie |
| POST | `/api/instances/{id}/options` | Dodaj opcję do instancji |
| PATCH | `/api/instances/{id}/publish` | Opublikuj instancję |

### Grupy `/api/groups`, `/api/event-groups`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/groups` | Utwórz grupę atrakcji |
| POST | `/api/groups/{id}/members` | Dodaj atrakcję do grupy (REQUIRED/OPTIONAL) |
| PATCH | `/api/groups/{id}/publish` | Opublikuj grupę |
| POST | `/api/event-groups` | Utwórz grupę eventów (festiwal) |
| POST | `/api/event-groups/{id}/events` | Dodaj event do grupy (tylko type=EVENT) |
| GET | `/api/event-groups?from=2025-07-01&to=2025-07-31` | Szukaj po datach |
| PATCH | `/api/event-groups/{id}/publish` | Opublikuj |
| PATCH | `/api/event-groups/{id}/cancel` | Anuluj |

### Katalogi `/api/catalogs`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/catalogs` | Utwórz katalog sezonowy |
| GET | `/api/catalogs?season=SUMMER` | Katalogi danego sezonu |
| POST | `/api/catalogs/{id}/attractions` | Dodaj atrakcję do katalogu |
| POST | `/api/catalogs/{id}/groups` | Dodaj grupę do katalogu |
| DELETE | `/api/catalogs/{id}/attractions/{aid}` | Usuń atrakcję z katalogu |
| PATCH | `/api/catalogs/{id}/archive` | Zarchiwizuj katalog |

### Dostępność `/api/availability`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/availability/slots` | Utwórz slot (pula biletów na datę) |
| GET | `/api/availability/slots?attractionId=X&date=2025-07-01` | Sloty atrakcji |
| GET | `/api/availability/slots/{id}/check?count=5&visitTime=14:30` | Sprawdź dostępność |
| POST | `/api/availability/slots/{id}/reserve` | Zarezerwuj `{ count, visitTime }` |
| POST | `/api/availability/slots/{id}/cancel` | Anuluj `{ count }` |

### Plany `/api/plans`

| Metoda | URL | Opis |
|--------|-----|------|
| POST | `/api/plans` | Utwórz plan (wymaga catalogId) |
| GET | `/api/plans` | Lista planów |
| GET | `/api/plans/{id}` | Pobierz plan |
| POST | `/api/plans/{id}/attractions` | Dodaj atrakcję do planu |
| POST | `/api/plans/{id}/groups` | Dodaj całą grupę do planu |
| PATCH | `/api/plans/{id}/confirm` | Potwierdź plan (walidacja wszystkich reguł) |

---

## 11. Uruchomienie

### Wymagania

- Java 17+
- Maven 3.8+

### Uruchomienie

```bash
git clone <repo>
cd attraction-service
mvn spring-boot:run
```

Serwis startuje na `http://localhost:8080`.

Po starcie `DataInitializer` automatycznie wgrywa przykładowe dane (patrz sekcja 12).

### Struktura projektu

```
src/main/java/com/attractions/
├── AttractionServiceApplication.java
├── DataInitializer.java
├── attraction/
│   ├── domain/          ← Attraction, Archetype, Instance, Group, EventGroup,
│   │                       AttractionOption, OpeningHours, Season, RelationType,
│   │                       Specifications, DomainEvent, *Events, *Repository (interfejsy)
│   ├── application/     ← AttractionService, ArchetypeService, EventGroupService,
│   │                       DomainEventPublisher, InMemory*Repository, *EventListeners
│   └── api/             ← AttractionController, DTOs, GlobalExceptionHandler
├── catalog/
│   ├── domain/          ← Catalog
│   ├── application/     ← CatalogService, CatalogEventListeners, InMemoryCatalogRepository
│   └── api/             ← CatalogController
├── availability/
│   ├── domain/          ← AvailabilitySlot, AvailabilitySpecifications, AvailabilityEvents,
│   │                       AvailabilitySlotRepository (interfejs)
│   ├── application/     ← AvailabilityService, AvailabilityEventListeners,
│   │                       InMemoryAvailabilitySlotRepository
│   └── api/             ← AvailabilityController
└── plan/
    ├── domain/          ← (Plan, PlanEntry w application – do wydzielenia)
    ├── application/     ← PlanService, InMemoryPlanRepository
    └── api/             ← PlanController
```

---

## 12. Przykładowe dane startowe

`DataInitializer` tworzy przy każdym starcie:

### Muzeum Narodowe w Krakowie (`type=MUSEUM`)
- Opcja 1: Zwiedzanie standardowe – 25 zł, max 200 osób, ALL_YEAR, 9:00–17:00
- Opcja 2: Zwiedzanie z przewodnikiem – 45 zł, max 20 osób, ALL_YEAR, 10:00–16:00
- Opcja 3: Tour dla niewidomych – 15 zł, max 10 osób, ALL_YEAR, 11:00–14:00
- Wymagania: `booking_recommended`, `no_flash_photography`

### Rejs po Wiśle (`type=CRUISE`, mandatory=true)
- Opcja: Rejs standardowy (1h) – 35 zł, max 50 osób, SPRING/SUMMER/AUTUMN, 10:00–18:00
- Wymagania: `life_jacket_provided`, `min_age:3`

### Zwiedzanie wyspy na Wiśle (`type=TOUR`)
- Opcja: Wycieczka piesza (2h) – 20 zł, max 30 osób, SPRING/SUMMER, 9:00–17:00
- Relacja: `REQUIRES` → Rejs po Wiśle (muszą być razem)
- Wymagania: `comfortable_shoes`, `weather_dependent`

### Grupa: Rejs + Wyspa
- Rejs: `REQUIRED`
- Wyspa: `REQUIRED`

### Park Krakowski (`type=PARK`)
- Opcja letnia: Letni piknik – 50 zł, SUMMER, 10:00–20:00
- Opcja zimowa: Zimowy spacer – 30 zł, WINTER, 14:00–19:00

### Archetype: Degustacja wina (`type=EVENT`)
- Instancja Lato 2025: 120 zł, SUMMER, 17:00–21:00, max 15 osób
- Instancja Zima 2025: 150 zł, WINTER, 18:00–22:00, max 10 osób

### EventGroup: Jazz Summer Festival 2025
- Zakres: 1–7 lipca 2025
- Slot 1: Koncert Otwarcia – 1 lipca, REQUIRED
- Slot 2: Wielki Finał – 7 lipca, REQUIRED

---

## Przykład: kompletny przepływ rezerwacji

```bash
# 1. Sprawdź opublikowane atrakcje
GET /api/attractions?status=PUBLISHED

# 2. Utwórz katalog letni
POST /api/catalogs
{ "name": "Katalog Lato 2025", "season": "SUMMER" }

# 3. Dodaj atrakcję do katalogu (waliduje sezon automatycznie)
POST /api/catalogs/{catalogId}/attractions
{ "attractionId": "uuid-muzeum" }

# 4. Utwórz slot dostępności (godziny pobrane z opcji automatycznie)
POST /api/availability/slots
{ "attractionId": "uuid-muzeum", "optionId": "uuid-opcji", "date": "2025-07-15", "totalCapacity": 200 }

# 5. Sprawdź dostępność o konkretnej godzinie
GET /api/availability/slots/{slotId}/check?count=4&visitTime=14:30

# 6. Zarezerwuj
POST /api/availability/slots/{slotId}/reserve
{ "count": 4, "visitTime": "14:30" }

# 7. Utwórz plan wycieczki
POST /api/plans
{ "name": "Wycieczka do Krakowa", "catalogId": "uuid-katalogu" }

# 8. Dodaj atrakcje do planu
POST /api/plans/{planId}/attractions
{ "attractionId": "uuid-muzeum", "optionId": "uuid-opcji", "mandatory": true }

# 9. Potwierdź plan (sprawdza REQUIRES, INCOMPATIBLE, grupy)
PATCH /api/plans/{planId}/confirm
```

---

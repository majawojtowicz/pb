# Mikroserwis Atrakcji Turystycznych

> Jeden mikroserwis · Cztery moduły

| | |
|---|---|
| **Wersja** | 1.0 – Prototyp |
| **Stack** | Java 21 + Spring Boot 3.2 |

---

## Spis treści

1. [Cel i założenia](#1-cel-i-założenia)
2. [Przegląd czterech modułów](#2-przegląd-czterech-modułów)
3. [Model domenowy](#3-model-domenowy)
4. [Katalog i sezonowość](#4-katalog-i-sezonowość)
5. [Plan atrakcji i pula biletów](#5-plan-atrakcji-i-pula-biletów)
6. [Endpointy REST API](#6-endpointy-rest-api)
7. [Struktura projektu](#7-struktura-projektu)
8. [Dane startowe (hardkodowane)](#8-dane-startowe-hardkodowane)
9. [Jak dodać nowy typ atrakcji](#9-jak-dodać-nowy-typ-atrakcji)
10. [Podsumowanie – wzorce i decyzje](#10-podsumowanie--wzorce-i-decyzje)

---

## 1. Cel i założenia

System udostępnia atrakcje turystyczne różnego rodzaju (muzea, parki, rejsy, wycieczki, eventy i inne) w sposób generyczny – tak, aby **dodanie nowego rodzaju atrakcji nie wymagało ponownego deploymentu** całego systemu.

### Kluczowe założenia

- Jeden mikroserwis podzielony na **4 logiczne moduły** (osobne pakiety, wspólny deployment)
- Architektura warstwowa w każdym module: `domain → application → api`
- **Wzorzec Archetypu Produktu** – definicja atrakcji jest niezmienną tożsamością
- **Wzorzec Specyfikacji** – każdy typ atrakcji opisany osobną klasą, bez modyfikacji reszty kodu
- **Katalog sezonowy** – inne atrakcje latem, inne zimą (konfiguracja, nie kod)
- **Plan tylko rośnie** – pozycje planu można dodawać, nie usuwać
- Pula biletów z zabezpieczeniem przed overbookingiem
- Okna dostępności (godziny otwarcia) konfigurowalne per atrakcja
- Relacje między atrakcjami: obligatoryjne (razem lub wcale) i opcjonalne (można osobno)

### Co NIE jest w zakresie prototypu

- Frontend (pominięty celowo)
- Baza danych (dane hardkodowane in-memory)
- Autentykacja i autoryzacja
- Zewnętrzne integracje płatnicze

---

## 2. Przegląd czterech modułów

> **Jeden mikroserwis = jeden deployment.**
> Moduły to pakiety Java (`com.attractions.core`, `.spec`, `.catalog`, `.plan`), nie osobne usługi.
> Komunikacja wewnątrz JVM przez serwisy i interfejsy – bez HTTP między modułami.

### Moduł 1 – `attraction-core` (Archetyp & Tożsamość)

- Przechowuje **niezmienną tożsamość** atrakcji: id, nazwa, kategoria, tagi, opis
- Zarządza stanami: `DRAFT → DEFINED → PUBLISHED → ARCHIVED`
- Obsługuje grupy atrakcji (`AttractionGroup`) – np. rejs i wyspa razem
- Definiuje relacje: `MANDATORY_TOGETHER`, `OPTIONAL_TOGETHER`, `STANDALONE`
- Relacje są niezmienne po przejściu do stanu `PUBLISHED`
- Punkt wejścia dla wszystkich pozostałych modułów

### Moduł 2 – `attraction-spec` (Specyfikacja & Wzorzec)

- Definiuje abstrakcyjną klasę `AttractionSpec` z kontraktem dla każdego typu
- Konkretne specyfikacje: `MuseumSpec`, `EventSpec`, `TourSpec`, `ParkSpec`, `BoatTourSpec`...
- Rejestr specyfikacji (`SpecificationRegistry`): `Map<String, AttractionSpec>`
- **Nowy typ atrakcji = nowa klasa implements AttractionSpec. Zero zmian w reszcie kodu.**
- Każda spec zawiera: `maxCapacity()`, `isAvailableAt(time)`, `getType()`, `validate()`

### Moduł 3 – `attraction-catalog` (Katalog & Sezonowość)

- `CatalogEntry` = archetyp + spec + `SeasonPolicy` (SUMMER / WINTER / ALL_YEAR)
- `AvailabilityWindow`: dni tygodnia + godziny otwarcia – konfigurowalne per wpis
- Ten sam archetyp może mieć dwa wpisy w katalogu: letni i zimowy z różnymi godzinami
- `CatalogService.getActiveEntries(season, date)` – filtruje dostępne atrakcje
- Katalog jest append-only: wpisy można dodawać i dezaktywować, nie usuwać

### Moduł 4 – `attraction-plan` (Plan & Bilety)

- `AttractionPlan` = uporządkowana lista `PlanEntry` z wpisów katalogowych
- **Plan tylko rośnie**: metoda `addEntry()`, brak `removeEntry()`. Immutable po zamknięciu.
- Każda `PlanEntry` ma flagę: `MANDATORY` (musi wystąpić) / `OPTIONAL` (może wystąpić)
- `TicketPool` per wpis: `totalCapacity`, `reserved` – blokada przed overbookingiem
- `PlanService.reserve()` sprawdza: czy wpis aktywny, czy jest wolne miejsce, czy relacje OK
- Relacje są walidowane: grup nie można rezerwować częściowo (rejs bez wyspy = błąd)

---

## 3. Model domenowy

### 3.1 Stany atrakcji

| Stan | Opis | Przejście do | Kto zmienia |
|---|---|---|---|
| `DRAFT` | Atrakcja w trakcie tworzenia, dane niekompletne | `DEFINED` | Redaktor |
| `DEFINED` | Opis i wymagania uzupełnione, tożsamość ustalona | `PUBLISHED` | Redaktor |
| `PUBLISHED` | Aktywna, widoczna w katalogu, można rezerwować | `ARCHIVED` | Admin |
| `ARCHIVED` | Wycofana, niewidoczna, dane zachowane dla historii | *(koniec)* | Admin |

### 3.2 Typy relacji między atrakcjami

| Typ relacji | Znaczenie | Przykład | Efekt w planie |
|---|---|---|---|
| `MANDATORY_TOGETHER` | Obie lub żadna – nie można zarezerwować osobno | Rejs + wyspa | System blokuje rezerwację jednej bez drugiej |
| `OPTIONAL_TOGETHER` | Mogą wystąpić razem, ale nie muszą | Muzeum + kawiarnia | Sugestia przy rezerwacji, bez blokady |
| `STANDALONE` | Brak powiązania z innymi | Park narodowy | Rezerwacja niezależna od innych |

### 3.3 Wzorzec Specyfikacji – kluczowy kod

> Wzorzec Specyfikacji pozwala definiować nowe typy atrakcji bez modyfikacji istniejącego kodu.
> Nowa atrakcja = nowa klasa Java. System ją automatycznie rozpoznaje przez rejestr.

**Kontrakt bazowy:**

```java
public abstract class AttractionSpec {
    public abstract String getType();
    public abstract boolean isAvailableAt(LocalDateTime dateTime);
    public abstract int getMaxCapacity();
    public abstract void validate(); // rzuca SpecificationException
}
```

**Przykładowa specyfikacja muzeum:**

```java
public class MuseumSpec extends AttractionSpec {
    private LocalTime openFrom;     // konfigurowalne
    private LocalTime openTo;       // konfigurowalne
    private int numberOfRooms;
    private int visitorsPerRoom;

    @Override
    public boolean isAvailableAt(LocalDateTime dt) {
        LocalTime t = dt.toLocalTime();
        return !t.isBefore(openFrom) && !t.isAfter(openTo);
    }

    @Override
    public int getMaxCapacity() { return numberOfRooms * visitorsPerRoom; }
}
```

**Rejestr specyfikacji (automatyczna rejestracja nowych typów):**

```java
@Component
public class SpecificationRegistry {
    private final Map<String, AttractionSpec> specs = new HashMap<>();

    public void register(AttractionSpec spec) {
        specs.put(spec.getType(), spec);
    }

    public AttractionSpec getSpec(String type) {
        return Optional.ofNullable(specs.get(type))
            .orElseThrow(() -> new UnknownAttractionTypeException(type));
    }
}
```

### 3.4 Archetyp produktu

Archetyp to niezmienna czesc atrakcji. Raz opublikowany archetyp nie może być modyfikowany. Zmiany wymagają stworzenia nowego archetypu (np. nowa wersja trasy zwiedzania).

| Pole | Typ | Czy zmienne? | Opis |
|---|---|---|---|
| `id` | UUID | NIE | Unikalne ID nadawane przy tworzeniu |
| `name` | String | NIE (po DEFINED) | Oficjalna nazwa atrakcji |
| `category` | Enum | NIE (po DEFINED) | MUSEUM, PARK, EVENT, TOUR, BOAT... |
| `tags` | Set\<String\> | NIE (po DEFINED) | Tagi wyszukiwania: "historia", "dla dzieci" |
| `description` | String | NIE (po DEFINED) | Trwały opis – nie reklama, ale tożsamość |
| `status` | Enum | TAK | DRAFT → DEFINED → PUBLISHED → ARCHIVED |
| `groupId` | UUID? | NIE (po DEFINED) | ID grupy, jeśli atrakcja jest częścią grupy |
| `relationshipType` | Enum | NIE (po DEFINED) | STANDALONE / MANDATORY_TOGETHER / OPTIONAL |

---

## 4. Katalog i sezonowość

Katalog to kolekcja aktywnych wpisów (`CatalogEntry`), z których każdy łączy archetyp z konkretną konfiguracją sezonową. Ten sam archetyp (np. "Muzeum Narodowe") może mieć dwa wpisy: letni (dłuższe godziny) i zimowy (krótsze).

### 4.1 Struktura CatalogEntry

| Pole | Typ | Opis |
|---|---|---|
| `id` | UUID | ID wpisu katalogowego |
| `archetypeId` | UUID | Referencja do `AttractionArchetype` |
| `spec` | AttractionSpec | Specyfikacja z konfiguracją (godziny, pojemność) |
| `season` | SeasonPolicy | SUMMER / WINTER / ALL_YEAR |
| `availabilityWindow` | AvailabilityWindow | Dni tygodnia + godziny otwarcia |
| `ticketPool` | TicketPool | Pula biletów dla tej konkretnej instancji |
| `active` | boolean | Czy wpis aktywny (dezaktywacja zamiast usunięcia) |
| `validFrom` | LocalDate | Data wejścia w życie |
| `validTo` | LocalDate? | Data wygaśnięcia (null = bezterminowy) |

### 4.2 SeasonPolicy i AvailabilityWindow

`SeasonPolicy` decyduje, kiedy wpis jest widoczny w systemie:

- `SUMMER` – widoczny od 1 czerwca do 31 sierpnia
- `WINTER` – widoczny od 1 grudnia do 28/29 lutego
- `ALL_YEAR` – zawsze widoczny

`AvailabilityWindow` konfiguruje konkretne okna czasowe:

```java
public class AvailabilityWindow {
    private Set<DayOfWeek> openDays;       // np. {MON, TUE, WED, THU, FRI}
    private LocalTime openFrom;             // np. 09:00
    private LocalTime openTo;               // np. 17:00
    private LocalTime lunchBreakFrom;       // opcjonalne: 12:00
    private LocalTime lunchBreakTo;         // opcjonalne: 13:00

    public boolean isOpen(LocalDateTime dateTime) {
        DayOfWeek day = dateTime.getDayOfWeek();
        LocalTime time = dateTime.toLocalTime();
        if (!openDays.contains(day)) return false;
        if (time.isBefore(openFrom) || time.isAfter(openTo)) return false;
        if (lunchBreakFrom != null &&
            !time.isBefore(lunchBreakFrom) && !time.isAfter(lunchBreakTo)) return false;
        return true;
    }
}
```

---

## 5. Plan atrakcji i pula biletów

### 5.1 AttractionPlan

Plan to uporządkowana, rosnąca kolekcja wpisów z aktualnego katalogu. Raz dodana pozycja nie może być usunięta. Plan może być w stanie `OPEN` (można dodawać) lub `LOCKED` (zamknięty).

> **Reguła biznesowa:** Plan może zawierać tylko wpisy z aktualnie aktywnego katalogu.
> Nie można dodać do planu atrakcji oznaczonej jako nieaktywna lub spoza sezonu.

### 5.2 PlanEntry – pozycja planu

| Pole | Typ | Opis |
|---|---|---|
| `catalogEntryId` | UUID | Referencja do `CatalogEntry` |
| `presence` | Enum | `MANDATORY` (musi być) / `OPTIONAL` (może być) |
| `plannedDate` | LocalDate | Planowana data realizacji |
| `addedAt` | LocalDateTime | Znacznik czasu dodania (immutable) |
| `reservationIds` | List\<UUID\> | Powiązane rezerwacje biletów |

### 5.3 TicketPool – mechanizm anty-overbooking

```java
public class TicketPool {
    private final int totalCapacity;
    private int reserved; // atomowo inkrementowany

    public synchronized ReservationResult reserve(int count) {
        int available = totalCapacity - reserved;
        if (available < count) {
            return ReservationResult.failed(
                "Brak miejsc. Dostepne: " + available + ", wymagane: " + count
            );
        }
        reserved += count;
        return ReservationResult.success(UUID.randomUUID());
    }

    public synchronized void release(int count) {
        reserved = Math.max(0, reserved - count);
    }

    public int getAvailable() { return totalCapacity - reserved; }
}
```

### 5.4 Walidacja relacji przy rezerwacji

`PlanService` sprawdza relacje między atrakcjami przed zatwierdzeniem rezerwacji:

1. Pobierz `AttractionGroup` dla rezerwowanej atrakcji
2. Jeśli relacja = `MANDATORY_TOGETHER`: sprawdź, czy wszystkie atrakcje z grupy są w planie
3. Jeśli którejkolwiek brakuje: rzuć `MissingMandatoryAttractionException`
4. Jeśli wszystkie są: zarezerwuj bilety atomowo dla całej grupy
5. Jeśli `OPTIONAL_TOGETHER`: zaloguj sugestię, kontynuuj rezerwację
6. Przy błędzie rezerwacji jednej z grupy: rollback wszystkich w grupie

---

## 6. Endpointy REST API

### 6.1 Moduł `attraction-core`

| Metoda | Endpoint | Opis |
|---|---|---|
| `POST` | `/api/attractions` | Tworzy nowy archetyp (stan: DRAFT) |
| `GET` | `/api/attractions` | Lista archety pów (filtr: `status`, `category`) |
| `GET` | `/api/attractions/{id}` | Szczegóły archetypu |
| `PUT` | `/api/attractions/{id}/define` | DRAFT → DEFINED |
| `PUT` | `/api/attractions/{id}/publish` | DEFINED → PUBLISHED |
| `PUT` | `/api/attractions/{id}/archive` | PUBLISHED → ARCHIVED |
| `POST` | `/api/attraction-groups` | Tworzy grupę (lista ID + typ relacji) |
| `GET` | `/api/attraction-groups/{id}` | Szczegóły grupy z listą atrakcji |

### 6.2 Moduł `attraction-spec`

| Metoda | Endpoint | Opis |
|---|---|---|
| `GET` | `/api/spec/types` | Lista zarejestrowanych typów specyfikacji |
| `POST` | `/api/attractions/{id}/spec` | Przypisuje specyfikację do archetypu |
| `GET` | `/api/attractions/{id}/spec` | Pobiera aktualną specyfikację |
| `GET` | `/api/attractions/{id}/availability?dateTime=...` | Czy atrakcja dostępna w danej chwili? |

### 6.3 Moduł `attraction-catalog`

| Metoda | Endpoint | Opis |
|---|---|---|
| `POST` | `/api/catalog/entries` | Dodaje wpis do katalogu (archetyp + spec + sezon) |
| `GET` | `/api/catalog/entries` | Lista wpisów (filtr: `season`, `active`, `date`) |
| `GET` | `/api/catalog/entries/{id}` | Szczegóły wpisu z dostępnością |
| `PUT` | `/api/catalog/entries/{id}/deactivate` | Dezaktywuje wpis (nie usuwa) |
| `GET` | `/api/catalog/entries/{id}/availability?date=...` | Okna dostępności w danym dniu |

### 6.4 Moduł `attraction-plan`

| Metoda | Endpoint | Opis |
|---|---|---|
| `POST` | `/api/plans` | Tworzy nowy plan (pusty, stan: OPEN) |
| `GET` | `/api/plans/{id}` | Pobiera plan z wszystkimi pozycjami |
| `POST` | `/api/plans/{id}/entries` | Dodaje pozycję do planu (MANDATORY/OPTIONAL) |
| `PUT` | `/api/plans/{id}/lock` | Zamyka plan (stan: LOCKED) |
| `POST` | `/api/plans/{id}/entries/{entryId}/reserve` | Rezerwuje bilety (walidacja overbooking + relacji) |
| `DELETE` | `/api/plans/{id}/entries/{entryId}/reserve` | Anuluje rezerwację (zwraca bilety do puli) |
| `GET` | `/api/plans/{id}/tickets/availability` | Dostępność biletów dla wszystkich pozycji planu |

---

## 7. Struktura projektu

Każdy moduł ma identyczną strukturę wewnętrzną (architektura warstwowa):

```
com.attractions/
├── AttractionServiceApplication.java   ← klasa startowa
│
├── core/                               ← MODUŁ 1
│   ├── domain/
│   │   ├── AttractionArchetype.java
│   │   ├── AttractionStatus.java        (enum)
│   │   ├── AttractionGroup.java
│   │   ├── AttractionCategory.java      (enum)
│   │   └── RelationshipType.java        (enum)
│   ├── application/
│   │   ├── AttractionService.java       (logika biznesowa)
│   │   └── AttractionGroupService.java
│   └── api/
│       ├── AttractionController.java
│       └── dto/
│
├── spec/                               ← MODUŁ 2
│   ├── domain/
│   │   ├── AttractionSpec.java          (abstract)
│   │   ├── MuseumSpec.java
│   │   ├── EventSpec.java
│   │   ├── TourSpec.java
│   │   ├── ParkSpec.java
│   │   └── BoatTourSpec.java
│   ├── application/
│   │   ├── SpecificationService.java
│   │   └── SpecificationRegistry.java   (rejestr typów)
│   └── api/
│       └── SpecificationController.java
│
├── catalog/                            ← MODUŁ 3
│   ├── domain/
│   │   ├── CatalogEntry.java
│   │   ├── SeasonPolicy.java            (enum)
│   │   └── AvailabilityWindow.java
│   ├── application/
│   │   └── CatalogService.java
│   └── api/
│       └── CatalogController.java
│
└── plan/                               ← MODUŁ 4
    ├── domain/
    │   ├── AttractionPlan.java
    │   ├── PlanEntry.java
    │   ├── PlanStatus.java              (enum: OPEN / LOCKED)
    │   ├── PresenceType.java            (enum: MANDATORY / OPTIONAL)
    │   └── TicketPool.java
    ├── application/
    │   └── PlanService.java             (walidacja relacji + overbooking)
    └── api/
        └── PlanController.java
```

---

## 8. Dane startowe (hardkodowane)

Dane inicjalizowane przez `DataInitializer` przy starcie aplikacji (`@PostConstruct`). Ilustrują pełny przekrój funkcjonalności: różne typy, grupy, oba sezony.

### Przykładowe archetypy

| Nazwa | Kategoria | Status | Relacja |
|---|---|---|---|
| Muzeum Narodowe | MUSEUM | PUBLISHED | STANDALONE |
| Rejs po Wiśle | BOAT_TOUR | PUBLISHED | MANDATORY_TOGETHER (z wyspą) |
| Wyspa Chopina | ISLAND_TOUR | PUBLISHED | MANDATORY_TOGETHER (z rejsem) |
| Park Łazienkowski | PARK | PUBLISHED | STANDALONE |
| Festiwal Jazzowy | EVENT | PUBLISHED | OPTIONAL_TOGETHER (z parkiem) |
| Zamek Królewski | MUSEUM | PUBLISHED | STANDALONE |
| Wycieczka rowerowa | TOUR | DRAFT | STANDALONE |

### Przykładowe wpisy katalogowe

| Atrakcja | Sezon | Godziny | Pojemność |
|---|---|---|---|
| Muzeum Narodowe (lato) | SUMMER | Wt–N, 10:00–18:00 | 200 os. |
| Muzeum Narodowe (zima) | WINTER | Wt–N, 10:00–16:00 | 150 os. |
| Rejs po Wiśle (lato) | SUMMER | Codziennie, 9:00–20:00 | 80 os. |
| Park Łazienkowski | ALL_YEAR | Codziennie, 6:00–22:00 | 500 os. |
| Festiwal Jazzowy | SUMMER | Pt–N, 18:00–23:00 | 1000 os. |

---

## 10. Podsumowanie – wzorce i decyzje

| Problem | Zastosowane rozwiązanie | Moduł |
|---|---|---|
| Nowy typ atrakcji bez redeploymentu | Wzorzec Specyfikacji + `SpecificationRegistry` | `spec` |
| Niezmienność tożsamości atrakcji | Wzorzec Archetypu Produktu + blokada po DEFINED | `core` |
| Katalog letni vs zimowy | `SeasonPolicy` per `CatalogEntry` (SUMMER/WINTER/ALL_YEAR) | `catalog` |
| Instancja a definicja | Archetyp = definicja, `CatalogEntry` = instancja konfiguracyjna | `catalog` |
| Godziny otwarcia – konfigurowalne | `AvailabilityWindow` per `CatalogEntry`, nie hardkodowane | `catalog` |
| Plan tylko rośnie | Brak metody `removeEntry()`, stan LOCKED blokuje dodawanie | `plan` |
| No-overbooking | `TicketPool.reserve()` z `synchronized`, sprawdzenie przed zapisem | `plan` |
| Rejs + wyspa = razem | `AttractionGroup` + `RelationshipType.MANDATORY_TOGETHER` | `core/plan` |
| Muzeum + kawiarnia = opcja | `RelationshipType.OPTIONAL_TOGETHER` + sugestia w odpowiedzi | `core/plan` |
| Separacja definicji od rezerwacji | Moduły core/spec/catalog = definicja; plan = rezerwacja | all |

> **Kluczowa zasada:** definicje atrakcji (`core`, `spec`, `catalog`) nigdy nie wiedzą o rezerwacjach (`plan`).
> Plan zna katalog i sprawdza archetypy, ale nie modyfikuje ich. Komunikacja jednokierunkowa.

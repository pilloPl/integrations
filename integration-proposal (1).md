# Propozycja Integracji: Procesy Windykacyjne ↔ Core Finance Engine

> ⚠️ **UWAGA:** Przedstawione w dokumencie JSON-y i API to **propozycja, nie deklaracja**.
> Zespół Core może nazwać endpoint'y inaczej, ładunek informacyjny może być podobny, ale forma inna.
> Celem jest pokazanie **kierunku myślenia** i **zakresu funkcjonalności** - szczegóły kontraktu ustalamy wspólnie.

---

## Spis treści

### 1. Filozofia Integracji
- [Kierunek zależności: Procesy → Core](#kierunek-zależności-procesy--core)
- [Relacja Customer-Supplier](#relacja-customer-supplier)
- [Anty-wzorzec: Core jako suma przypadków](#anty-wzorzec-core-jako-suma-przypadków)
- [Zasada uogólnienia](#zasada-uogólnienia)

### 2. Czym jest DebtPart?
- [Koncepcja DebtPart](#koncepcja-debtpart)
- [Jak Dług mapuje się na DebtParty?](#jak-dług-mapuje-się-na-debtparty)
- [Dwa identyfikatory: Debt ID i DebtPart ID](#dwa-identyfikatory-debt-id-i-debtpart-id)
  - [Przypadek 1: Dług = 1 DebtPart](#przypadek-1-dług--1-debtpart-całość-w-jednym-procesie)
  - [Przypadek 2: Podział 50/50](#przypadek-2-dług--2-debtparty-podział-5050-między-procesy)
  - [Przypadek 3: Nakładające się (160% ≠ 100%)](#przypadek-3-dług--2-debtparty-nakładające-się---nie-sumują-się-do-100)
  - [Przypadek 4: Różne procesy per komponent](#przypadek-4-jeden-dług-różne-procesy-dla-różnych-komponentów)
- [Koszty niewpływające na saldo](#koszty-niewpływające-na-saldo)

### 3. Idea produktu
- [Zmiana myślenia](#zmiana-myślenia)
- [ProductElement - atomowy element](#productelement---atomowy-element)
- [Product - zbiór elementów](#product---zbiór-elementów)
- [Przejścia warunkowe](#przejścia-warunkowe)
- [ProductType i FeatureInstances](#producttype-i-featureinstances)
- [Constrainty per kraj](#constrainty-per-kraj)

### 4. RepaymentPlan
- [Czym jest RepaymentPlan?](#czym-jest-repaymentplan)
- [Wiele planów na jeden DebtPart](#wiele-planów-na-jeden-debtpart)

### 5. Payments
- [Spłata na konkretny plan](#spłata-na-konkretny-plan)
- [Spłata bez wskazania planu](#spłata-bez-wskazania-planu)
- [Payment Resolution](#payment-resolution)

### 6. API
- [Dostępne operacje](#dostępne-operacje)
  - **DebtPart:** Tworzenie, Solidarność, Niesolidarność, Limity, Komponenty, Split, Merge, Stan, Historia
  - **Costs:** Dodawanie, Usuwanie, Lista kosztów
  - **ProductCatalog:** ProductElement, Product, Condition, Wyszukiwanie
  - **RepaymentPlan:** Tworzenie, Aktywacja, Lista, Stan
  - **Payments:** Wpłata, Strategie rozliczania
  - **Billing:** Obsługa płatności, Naliczenia

### 7. Q&A
- [Obciążenia i produkty](#obciążenia-i-produkty)
- [DebtPart i komponenty](#debtpart-i-komponenty)
- [Płatności](#płatności)
- [Dodawanie produktów do katalogu](#dodawanie-produktów-do-katalogu)
- [Konfiguracja obsługi finansowej](#konfiguracja-obsługi-finansowej)
- [Warunki i zmiany stóp procentowych](#warunki-i-zmiany-stóp-procentowych)
- [Pobieranie danych finansowych (DWH, Controlling)](#pobieranie-danych-finansowych-dwh-controlling)
- [Przykłady przepływów (pseudo-kod)](#przykłady-przepływów-pseudo-kod)
- [Korekta salda po wyroku/nakazie](#korekta-salda-po-wyrokunakazie)
- [Archiwizacja długu](#archiwizacja-długu)
- [Pełna historia długu od importu](#pełna-historia-długu-od-importu)
- [Śledzenie przejścia między procesami](#śledzenie-przejścia-między-procesami)
- [Historia i audyt](#historia-i-audyt)


---

## 1. Filozofia Integracji

### Kierunek zależności: Procesy → Core

Core Finance Engine to **stabilne jądro** systemu finansowego. Procesy windykacyjne (sądowe, polubowne, z różnych krajów) **dostosowują się do Core**, nie odwrotnie.

```
        ┌───────────────────┐       ┌───────────────────┐       ┌───────────────────┐
        │   Proces Sądowy   │       │ Proces Polubowny  │       │   Proces XYZ      │
        │   (Polska)        │       │ (Rumunia)         │       │   (Kraj N)        │
        │                   │       │                   │       │                   │
        │  Zna koncepty     │       │  Zna koncepty     │       │  Zna koncepty     │
        │  z Core:          │       │  z Core:          │       │  z Core:          │
        │  - DebtPart       │       │  - DebtPart       │       │  - DebtPart       │
        │  - Component      │       │  - Component      │       │  - Component      │
        │  - Share/Owner    │       │  - Share/Owner    │       │  - Share/Owner    │
        │  - Product        │       │  - Product        │       │  - Product        │
        └───────────────────┘       └───────────────────┘       └───────────────────┘
                    │                            │                            │
                    │                            │                            │
                    └────────────────────────────┼────────────────────────────┘
                                                 │
                                                 ▼
                              ┌─────────────────────────────────────┐
                              │         STABILNE API                │
                              │                                     │
                              │      CORE FINANCE ENGINE            │
                              │                                     │
                              │   ┌───────────┐  ┌───────────────┐  │
                              │   │ DebtPart  │  │  Accounting   │  │
                              │   └───────────┘  └───────────────┘  │
                              │                                     │
                              │   ┌───────────┐  ┌───────────────┐  │
                              │   │ Payments  │  │ProductCatalog │  │
                              │   └───────────┘  └───────────────┘  │
                              │                                     │
                              │   ┌───────────┐                     │
                              │   │  Billing  │                     │
                              │   └───────────┘                     │
                              │                                     │
                              └─────────────────────────────────────┘
```

### Relacja Customer-Supplier

Core i procesy są w relacji **Customer-Supplier** (w sensie Domain-Driven Design):

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│   SUPPLIER (Upstream)                    CUSTOMER (Downstream)                  │
│   ═══════════════════                    ════════════════════                   │
│                                                                                 │
│   ┌─────────────────────┐                ┌─────────────────────┐                │
│   │   CORE FINANCE      │   dostarcza    │     PROCESY         │                │
│   │   ENGINE            │ ─────────────► │     WINDYKACYJNE    │                │
│   │                     │      API       │                     │                │
│   │   - DebtPart API    │                │   Konsumują API     │                │
│   │   - Billing API     │                │   Adaptują się      │                │
│   │   - Product API     │                │   do abstrakcji     │                │
│   └─────────────────────┘                └─────────────────────┘                │
│                                                                                 │
│   ZASADY:                                                                       │
│   ────────                                                                      │
│   • Core definiuje kontrakt (API)                                               │
│   • Core w idealnym świecie NIE zna szczegółów procesów                         │
│   • Procesy MUSZĄ mapować swoje koncepty na koncepty Core                       │
│   • Procesy MOGĄ dyktować zmiany w Core JEŚLI da się je uogólnić                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Anty-wzorzec: Core jako suma przypadków

**ZAGROŻENIE:** Core NIE MOŻE stać się sumą wszystkich edge-case'ów z różnych krajów i procesów.

```
                    ╔═══════════════════════════════════════════════════════════╗
                    ║                        ŹLE                                ║
                    ╠═══════════════════════════════════════════════════════════╣
                    ║                                                           ║
                    ║   Core Finance Engine                                     ║
                    ║   ┌─────────────────────────────────────────────────────┐ ║
                    ║   │  if (kraj == "PL" && proces == "sądowy") {          │ ║
                    ║   │      // specyficzna logika dla Polski               │ ║
                    ║   │  }                                                  │ ║
                    ║   │  if (kraj == "RO" && proces == "polubowny") {       │ ║
                    ║   │      // specyficzna logika dla Rumunii              │ ║
                    ║   │  }                                                  │ ║
                    ║   │  if (kraj == "ES" && ...) {                         │ ║
                    ║   │      // kolejny przypadek...                        │ ║
                    ║   │  }                                                  │ ║
                    ║   │  // ... 47 innych if-ów                             │ ║
                    ║   └─────────────────────────────────────────────────────┘ ║
                    ║                                                           ║
                    ║   Rezultat: Nieztrzymywalny system                        ║
                    ║                                                           ║
                    ╚═══════════════════════════════════════════════════════════╝


                    ╔═══════════════════════════════════════════════════════════╗
                    ║                       DOBRZE                              ║
                    ╠═══════════════════════════════════════════════════════════╣
                    ║                                                           ║
                    ║   Core Finance Engine                                     ║
                    ║   ┌─────────────────────────────────────────────────────┐ ║
                    ║   │                                                     │ ║
                    ║   │   UOGÓLNIONE ABSTRAKCJE:                            │ ║
                    ║   │                                                     │ ║
                    ║   │   • DebtPart    - część długu z udziałami           │ ║
                    ║   │   • Component   - składnik (Principal, Interest...) │ ║
                    ║   │   • Share/Owner - podział własności                 │ ║
                    ║   │   • Product     - konfigurowalny produkt            │ ║
                    ║   │   • Condition   - warunek przejścia                 │ ║
                    ║   │                                                     │ ║
                    ║   │   Procesy KONFIGURUJĄ te abstrakcje,                │ ║
                    ║   │   a nie dodają własne if-y do Core                  │ ║
                    ║   │                                                     │ ║
                    ║   └─────────────────────────────────────────────────────┘ ║
                    ║                                                           ║
                    ║   Rezultat: Elastyczny, rozszerzalny system               ║
                    ║                                                           ║
                    ╚═══════════════════════════════════════════════════════════╝
```

### Zasada uogólnienia

Gdy proces potrzebuje nowej funkcjonalności, pytamy:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PROCES: "Potrzebuję obsługi X w Core"                                     │
│                                                                             │
│                            │                                                │
│                            ▼                                                │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  PYTANIE: Czy X to przypadek szczególny,                            │   │
│   │           czy uogólniona potrzeba wielu procesów?                   │   │
│   │           A może szczególny, ale dający się uogólnić?               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                                │
│         ┌──────────────────┼──────────────────┐                             │
│         │                  │                  │                             │
│         ▼                  ▼                  ▼                             │
│   ┌───────────────┐ ┌───────────────┐ ┌────────────────────┐                │
│   │  SZCZEGÓLNY   │ │  UOGÓLNIONY   │ │ SZCZEGÓLNY, ALE    │                │
│   │               │ │               │ │ DAJĄCY SIĘ         │                │
│   │ → Proces      │ │ → Core        │ │ UOGÓLNIĆ           │                │
│   │   obsługuje   │ │   rozszerza   │ │                    │                │
│   │   lokalnie    │ │   abstrakcje  │ │ → Na razie jeden   │                │
│   │               │ │               │ │   proces to chce   │                │
│   │ → Mapuje na   │ │ → Nowa funkcja│ │                    │                │
│   │   istniejące  │ │   dostępna dla│ │ → ALE potrafimy    │                │
│   │   API         │ │   WSZYSTKICH  │ │   przedstawić to   │                │
│   │               │ │               │ │   w sposób ogólny  │                │
│   │               │ │               │ │                    │                │
│   │               │ │               │ │ → Core wprowadza   │                │
│   │               │ │               │ │   abstrakcję       │                │
│   │               │ │               │ │   GOTOWĄ na inne   │                │
│   │               │ │               │ │   kraje/procesy    │                │
│   └───────────────┘ └───────────────┘ └────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Przykłady dobrej generalizacji:**

#### 1. Naliczanie odsetek w różnych jednostkach czasu

```
ŹLE:  if (country == "PL") dailyInterest()
      if (country == "RO") monthlyInterest()
```

```
DOBRZE: Proces polski konfiguruje swój Product z:
        AccrualCalculation {
            Unit = Days,
            Calculator = "SimpleInterest",
            From = [Principal],
            To = Interest
        }

        Proces rumuński konfiguruje:
        AccrualCalculation { Unit = Months, ... }

        Core NIE WIE, że to Polska czy Rumunia.
        Core tylko nalicza odsetki zgodnie z konfiguracją.
```

#### 2. Kolejność alokacji wpłat

```
ŹLE:  if (country == "ES") interestFirst()
      if (country == "DE") principalFirst()
```

```
DOBRZE: Proces hiszpański rejestruje RepaymentPlan z produktem:
        AllocationOrder = [Interest, Fee, Principal]

        Proces niemiecki rejestruje produkt:
        AllocationOrder = [Principal, Interest, Fee]

        Core NIE WIE, że "Hiszpania najpierw odsetki".
        Core alokuje wpłaty według przekazanej kolejności.
```

#### 3. Limity na komponentach długu

```
ŹLE:  if (processType == "court") capPrincipal(50000)
      if (processType == "amicable" && country == "PL") capInterest(10000)
```

```
DOBRZE: Proces sądowy tworzy DebtPart z:
        Limits = {
            Principal: { ShareId_1: { Owner_A: 50000 PLN } }
        }

        Proces polubowny tworzy DebtPart z:
        Limits = {
            Interest: { ShareId_1: { Owner_A: 10000 PLN } }
        }

        Core NIE WIE, że to "sądówka" czy "polubówka".
        Core respektuje limity przy partycjonowaniu.
```

#### 4. Warunkowe zmiany produktu (promocje)

```
ŹLE:  if (paidOnTime >= 3 && country == "PL") lowerRate()
```

```
DOBRZE: Proces polski definiuje Product z możliwymi przejściami:

        ProductElement "StandardRate" (annualRate: 12%)
            ──[Condition: RepaymentsOnTime, counter=3]──►
        ProductElement "PromoRate" (annualRate: 8%)

        Gdy warunek jest spełniony warunek, Core automatycznie
        przełącza na nowy ProductElement.

        Core NIE WIE, że to "polska promocja za terminowość".
        Core sprawdza Condition lub otrzymuje komunikat o spełnionym Condition z zewnątrz i wykonuje zmianę ProductElementu.
```

#### 5. Harmonogramy z kalendarzem świąt

```
ŹLE:  if (country == "PL") skipPolishHolidays()
      if (country == "RO") skipRomanianHolidays()
```

```
DOBRZE: Proces polski tworzy ScheduleTemplate z:
        HolidaysCalendar = PolishCalendar,
        AdjustmentStrategy = Forward,
        BusinessDaysOnly = true

        Proces rumuński tworzy ScheduleTemplate z:
        HolidaysCalendar = RomanianCalendar,
        AdjustmentStrategy = Backward

        Core NIE WIE, że "w Polsce 3 maja jest wolne".
        Core przesuwa daty według dostarczonego kalendarza.
```

---

**Podsumowanie:** Core operuje na abstrakcjach (np. `AccrualCalculation`, `AllocationOrder`, `Limits`,
`Condition`, `HolidaysCalendar`). Słowa "Polska", "Rumunia", "sądówka", "polubówka"
**nie istnieją w kodzie Core** - to procesy mapują swoje koncepty na konfigurację.

---

## 2. Czym jest DebtPart?

### Koncepcja DebtPart

**DebtPart** (cząstka długu) to część długu procesowana razem przez jeden proces.
Core nie wie czym jest "dług" w sensie biznesowym - zna tylko DebtParty.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   DŁUG (pojęcie biznesowe)          DebtPart (pojęcie Core)                 │
│   ════════════════════════          ═══════════════════════                 │
│                                                                             │
│   "Kowalski jest winien             "Cząstka długu z komponentami           │
│    100 000 PLN za kredyt"            i właścicielami, którą                 │
│                                      procesujemy razem"                     │
│                                                                             │
│   Core NIE WIE, że to kredyt.       Core WIE, że jest DebtPart              │
│   Core NIE WIE o umowach.           z Principal, Interest, Fee              │
│                                     i właścicielami.                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Jak Dług mapuje się na DebtParty?

Jeden dług biznesowy może być reprezentowany przez **jeden lub więcej DebtPartów**.
To zależy od tego, jak procesy chcą obsługiwać dług.

> **Uwaga:** Poniżej przedstawiamy kilka typowych przypadków - to nie jest wyczerpująca lista.
> W praktyce kombinacje mogą być dowolne, w zależności od potrzeb biznesowych.

### Dwa identyfikatory: Debt ID i DebtPart ID

Procesy operują **dwoma identyfikatorami**:

| ID | Znaczenie |
|----|-----------|
| **Debt ID** | ID długu biznesowego (np. umowy kredytowej) |
| **DebtPart ID** | ID konkretnej cząstki w Core |

**Jeden Debt ID → wiele DebtPart ID.** To kluczowa relacja - jeden dług biznesowy może mieć wiele cząstek.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Debt ID: "KREDYT-2024-00123"                                            │
│   (dług biznesowy - ten sam dla wszystkich cząstek)                         │
│                                                                             │
│   ┌─────────────────────────┐    ┌─────────────────────────┐                │
│   │  DebtPart ID: "aaa..."  │    │  DebtPart ID: "bbb..."  │                │
│   │                         │    │                         │                │
│   │  Proces: SĄDOWY         │    │  Proces: POLUBOWNY      │                │
│   │  50% długu              │    │  50% długu              │                │
│   └─────────────────────────┘    └─────────────────────────┘                │
│                                                                             │
│   Oba DebtParty mają ten sam Debt ID = "KREDYT-2024-00123"                │
│                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Jak to działa:**

1. **Proces tworzy DebtPart** i przekazuje swój Debt ID (ID długu biznesowego)
2. **Core zwraca DebtPart ID** - unikalny identyfikator cząstki
3. **Proces zapamiętuje oba ID** - wie, że DebtPart "aaa..." należy do długu "KREDYT-2024-00123"
4. **Gdy dług jest dzielony** - każda nowa cząstka dostaje nowy DebtPart ID, ale Debt ID pozostaje ten sam

**Korzyści:**

- **Elastyczne API** - operacje mogą działać na poziomie konkretnego DebtPart ID lub na wszystkich cząstkach danego Debt ID
- **Widok pojedynczy** - stan konkretnej cząstki (jeden proces, jedna część długu)
- **Widok zbiorczy** - stan wszystkich cząstek pod jednym długiem biznesowym (wszystkie procesy, cały dług)
- **Separacja odpowiedzialności** - Core nie musi rozumieć pojęcia "dług" - to tylko atrybut przekazany przez proces

---

---

#### Przypadek 1: Dług = 1 DebtPart (całość w jednym procesie)

Cały dług procesowany polubownie - jeden DebtPart pokrywa 100% długu.

```
┌─────────────────────────────────────────────────────────────────┐
│   Debt ID: "KREDYT-2024-00123"                                │
│   Kwota: 100 000 PLN kapitału + 10 000 PLN odsetek              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  DebtPart ID: "dp-aaa-111"                              │   │
│   │  Debt ID:   "KREDYT-2024-00123"                       │   │
│   │                                                         │   │
│   │  PRINCIPAL: 100 000 PLN                                 │   │
│   │  INTEREST:   10 000 PLN                                 │   │
│   │                                                         │   │
│   │  Proces: POLUBOWNY                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   Debt ID = DebtPart ID (w sensie: 1 dług = 1 cząstka)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Przypadek 2: Dług = 2 DebtParty (podział 50/50 między procesy)

Połowa długu idzie sądownie, połowa polubownie. Dwa osobne DebtParty, każdy 50%.

```
┌─────────────────────────────────────────────────────────────────┐
│   Debt ID: "KREDYT-2024-00123"                                │
│   Kwota: 100 000 PLN kapitału + 10 000 PLN odsetek              │
│                                                                 │
│   ┌───────────────────────────┐ ┌───────────────────────────┐   │
│   │  DebtPart ID: "dp-aaa"    │ │  DebtPart ID: "dp-bbb"    │   │
│   │  Debt ID: "KREDYT-..."  │ │  Debt ID: "KREDYT-..."  │   │
│   │                           │ │                           │   │
│   │  PRINCIPAL: 50 000 PLN    │ │  PRINCIPAL: 50 000 PLN    │   │
│   │  INTEREST:   5 000 PLN    │ │  INTEREST:   5 000 PLN    │   │
│   │                           │ │                           │   │
│   │  Proces: SĄDOWY           │ │  Proces: POLUBOWNY        │   │
│   └───────────────────────────┘ └───────────────────────────┘   │
│                                                                 │
│   Ten sam Debt ID → procesy wiedzą, że to jeden dług          │
│   SUMA: 100% długu = 50% + 50%                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Przypadek 3: Dług = 2 DebtParty (nakładające się - NIE sumują się do 100%!)

Proces sądowy dochodzi 100% długu, proces polubowny równolegle próbuje odzyskać 60%.
**DebtParty mogą się nakładać** - to nie jest błąd, to świadoma decyzja biznesowa.

```
┌─────────────────────────────────────────────────────────────────┐
│   Debt ID: "KREDYT-2024-00123"                                │
│   Kwota: 100 000 PLN kapitału + 10 000 PLN odsetek              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  DebtPart ID: "dp-aaa"                                  │   │
│   │  Debt ID:   "KREDYT-2024-00123"                       │   │
│   │                                                         │   │
│   │  PRINCIPAL: 100 000 PLN                                 │   │
│   │  INTEREST:   10 000 PLN                                 │   │
│   │                                                         │   │
│   │  Proces: SĄDOWY (pełna kwota do wyroku)                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  DebtPart ID: "dp-bbb"                                  │   │
│   │  Debt ID:   "KREDYT-2024-00123"  ← ten sam!           │   │
│   │                                                         │   │
│   │  PRINCIPAL: 60 000 PLN                                  │   │
│   │  INTEREST:   6 000 PLN                                  │   │
│   │                                                         │   │
│   │  Proces: POLUBOWNY (ugoda na 60%)                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   SUMA: 160% ≠ 100%  ← TO JEST OK!                              │
│                                                                 │
│   Ten sam Debt ID → procesy wiedzą, że to ten sam dług        │
│   i mogą synchronizować wpłaty między sobą.                     │
│                                                                 │
│   Scenariusz: Sądownie dochodzimy pełnej kwoty (na wypadek      │
│   gdyby ugoda nie doszła do skutku), a równolegle próbujemy     │
│   ugody na 60%. Jeśli ugoda zostanie spłacona, zamykamy         │
│   proces sądowy. Jeśli nie - kontynuujemy sądówkę.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Przypadek 4: Jeden dług, różne procesy dla różnych komponentów

Kapitał procesowany sądownie, odsetki polubownie (np. po wyroku sądowym dotyczącym tylko kapitału).

```
┌─────────────────────────────────────────────────────────────────┐
│   Debt ID: "KREDYT-2024-00123"                                │
│   Kwota: 100 000 PLN kapitału + 10 000 PLN odsetek              │
│                                                                 │
│   ┌───────────────────────────┐ ┌───────────────────────────┐   │
│   │  DebtPart ID: "dp-aaa"    │ │  DebtPart ID: "dp-bbb"    │   │
│   │  Debt ID: "KREDYT-..."  │ │  Debt ID: "KREDYT-..."  │   │
│   │                           │ │                           │   │
│   │  PRINCIPAL: 100 000 PLN   │ │  INTEREST: 10 000 PLN     │   │
│   │                           │ │                           │   │
│   │  Proces: SĄDOWY           │ │  Proces: POLUBOWNY        │   │
│   │  (tylko kapitał)          │ │  (tylko odsetki)          │   │
│   └───────────────────────────┘ └───────────────────────────┘   │
│                                                                 │
│   Ten sam Debt ID → różne komponenty tego samego długu        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Kluczowa zasada

**Core nie pilnuje, czy DebtParty "sumują się" do jakiegoś długu.**

To odpowiedzialność procesów, żeby wiedzieć:
- Które DebtParty należą do tego samego długu biznesowego (przez Debt ID)
- Jak synchronizować wpłaty między procesami
- Kiedy zamknąć jeden proces po sukcesie drugiego

Core dostarcza narzędzia do zarządzania cząstkami - logika biznesowa jest po stronie procesów.

---

### Koszty niewpływające na saldo

Nie wszystkie koszty związane z obsługą długu wpływają na saldo księgowe (Accounting). Przykłady:
- Koszty wysyłki monitów
- Opłaty za raporty BIK
- Koszty korespondencji
- Opłaty administracyjne

Takie koszty można **dopisywać** do długu w różny sposób:

| Poziom | Identyfikator | Opis |
|--------|---------------|------|
| Dług biznesowy | `debtId` | Koszt dotyczy całego długu (wszystkich cząstek) |
| Cząstka | `debtPartId` | Koszt dotyczy konkretnej cząstki |

**API do zarządzania kosztami:**

```
# Dodanie kosztu do całego długu (debtId)
POST /api/costs
{
  "debtId": "KREDYT-2024-00123",
  "type": "CORRESPONDENCE",
  "amount": 15.00,
  "currency": "PLN",
  "description": "Wysyłka monitu listem poleconym",
  "occurredAt": "2024-03-15"
}

# Dodanie kosztu do konkretnej cząstki
POST /api/costs
{
  "debtPartId": "dp-aaa-111",
  "type": "BIK_REPORT",
  "amount": 25.00,
  "currency": "PLN",
  "description": "Raport BIK dla procesu sądowego"
}

# Usunięcie/anulowanie kosztu
DELETE /api/costs/{costId}

# Lista kosztów dla długu
GET /api/costs/by-debt/{debtId}

# Lista kosztów dla cząstki
GET /api/costs/by-debt-part/{debtPartId}
```

---

## 3. Idea produktu

> **Uwaga:** Kategorie produktów, same produkty, ich zbiory, constrainty oraz features będą ustalane wspólnie w miarę potrzeb.
> Obecne wartości to **punkt startowy, nie zbiór zamknięty** - rozszerzamy w oparciu o realne wymagania procesów.

### Zmiana myślenia

**Windykacja niczego nie sprzedaje** - więc skąd moduł produktów?

Odpowiedź: windykacja **używa elementów** które wpływają na to:
- **Jak rozliczamy klienta** - harmonogram spłat, naliczanie odsetek, kolejność alokacji
- **Jak go obsługujemy** - wakacje kredytowe, umorzenia, zniżki

Każdy taki element może być **produktem**. To fundamentalna zmiana perspektywy:

```
❌ Tradycyjne myślenie:
   "Produkt = coś co sprzedajemy klientowi za pieniądze"

✅ Myślenie w tym module:
   "Produkt = konfigurowalny element wpływający na obsługę długu/dłużnika"
```

| Element | Dlaczego to "produkt"? |
|---------|------------------------|
| **Harmonogram 5 rat miesięcznych** | Konfiguruje jak klient spłaca dług |
| **Naliczanie odsetek 6% rocznie** | Określa jak rośnie zobowiązanie |
| **Wakacje kredytowe 3 miesiące** | Modyfikuje wymagane czasy spłaty |
| **Kolejność: najpierw odsetki** | Wpływa na alokację wpłat |
| **Zniżka 10%** | Modyfikator kwoty |

**Zasada:** Moduł jest **agnostyczny** - nie wie co oznaczają parametry produktu. To moduł konsumujący (np. Billing) interpretuje produkt i wie jak go wykonać.

---

### ProductElement - atomowy element

Najmniejsza jednostka konfiguracji. Każdy element ma **kategorię** określającą jego rolę:

| Kategoria | Rola |
|-----------|------|
| `Schedule` | Harmonogram spłat |
| `Accrual` | Naliczanie (np. odsetki) |
| `AllocationOrder` | Kolejność alokacji wpłat |
| `AmountModifier` | Modyfikator kwoty (zniżka/zwyżka) |
| `AccrualSchedule` | Harmonogram naliczeń |

**Dlaczego generyczny interfejs stringów?**

Produkty mogą być **diametralnie różne** - harmonogram spłat ma zupełnie inne parametry niż kalkulator odsetek:

```
// Harmonogram - ma zupełnie inne parametry...
ProductElement "5_MONTHS_SCHEDULE" {
    category: "Schedule",
    period: "Months",
    count: 5
}

// ...niż naliczanie odsetek
ProductElement "INTEREST_6_PERCENT" {
    category: "Accrual",
    calculator: "simple-interest-6",
    givenComponents: ["PRINCIPAL", "FEE"],
    targetComponent: "INTEREST"
}
```

**Korzyść:** Jeden model domenowy obsługuje wszystkie typy produktów bez eksplozji klas.

---

### Product - zbiór elementów

Product grupuje elementy w logiczną całość. Jeden `Product` może zawierać wiele `ProductElement` z różnych kategorii:

```
Product "STANDARD_REPAYMENT_PLAN" {
    Schedule:       [5_MONTHS_SCHEDULE]
    Accrual:        [INTEREST_6_PERCENT]
    AllocationOrder: [INTEREST_FIRST]
}
```

Typowy zestaw produktów tworzy **bazę dla RepaymentPlan w module Billing**.

---

### Przejścia warunkowe

Niektóre elementy aktywują się po spełnieniu **warunku** (Condition):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ProductElement "StandardRate" (12% rocznie)                               │
│                                                                             │
│       ──[Condition: RepaymentsOnTime, counter=3]──►                         │
│                                                                             │
│   ProductElement "PromoRate" (8% rocznie)                                   │
│                                                                             │
│   Gdy klient terminowo spłaci 3 raty, automatycznie dostaje niższe odsetki  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Dostępne typy warunków:**

| Typ | Opis |
|-----|------|
| `RepaymentsOnTime` | N terminowych wpłat |
| `RepaymentsDelayed` | N opóźnionych wpłat |
| `ManualConfirmation` | Ręczne potwierdzenie operatora |

---

### ProductType i FeatureInstances

Bardziej zaawansowany model podziału konfiguracji produktu:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ProductType (statyczne wartości)                                          │
│   ════════════════════════════════                                          │
│   - Odsetki ustawowe = 11.25%                                               │
│   - Maksymalna stopa = 2x referencyjna NBP                                  │
│   - Kalendarz świąt = PL                                                    │
│                                                                             │
│   FeatureInstances (dynamiczne wartości per etap)                           │
│   ═══════════════════════════════════════════════                           │
│   Etap 1: harmonogram 5 rat                                                 │
│   Etap 2: po 2 opóźnieniach → 1 płatność w 30 dni                           │
│                                                                             │
│   ProductInstance = ProductType + FeatureInstances                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Korzyści:**

1. **Gdy zmienia się stopa ustawowa** - aktualizujemy tylko ProductType, zmiana automatycznie trafia do wszystkich RepaymentPlanów używających tego typu
2. **Wersjonowanie** - ProductType i konfiguracja etapów mogą być wersjonowane niezależnie
3. **Audyt** - wiadomo jaki produkt/cennik obowiązywał w danym momencie

---

### Constrainty per kraj

Ogólna **konstrukcja modułów krajowych będzie taka sama** (te same koncepty: ProductElement, Product, Condition), ale **treść będzie inna**:

| Aspekt | Polska | Rumunia |
|--------|--------|---------|
| Kalendarz świąt | PL (Boże Ciało, 3 Maja...) | RO (Unirea, Ortodoksyjna Wielkanoc...) |
| Max stopa odsetek | 2x stopa referencyjna NBP | Inne limity |
| Dozwolone produkty | Katalog PL | Katalog RO |

**ProductConstraints** (planowane) - reguły które kombinacje są dozwolone:
- W rodzinie może być **tylko jeden aktywny Schedule** naraz
- Niektóre elementy wykluczają się wzajemnie
- Reguły są **różne per kraj**

```
// Polska - kalendarz świąt PL
ProductElement "PL_SCHEDULE" {
    category: "Schedule",
    holidaysCalendar: "PL"  // pomija polskie święta
}

// Rumunia - kalendarz świąt RO
ProductElement "RO_SCHEDULE" {
    category: "Schedule",
    holidaysCalendar: "RO"  // pomija rumuńskie święta
}
```

---

## 4. RepaymentPlan

### Czym jest RepaymentPlan?

**RepaymentPlan** to konkretna kombinacja produktów określająca **jak obsługujemy dług**. Jest kolekcją produktów przypiętą do DebtPart:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   RepaymentPlan "STANDARD_5_RAT"                                            │
│   ══════════════════════════════                                            │
│                                                                             │
│   Kolekcja produktów:                                                       │
│   • Harmonogram: 5 rat miesięcznych                                         │
│   • Naliczanie: odsetki dzienne od kapitału                                 │
│   • Alokacja: najpierw odsetki, potem kapitał                               │
│   • Zniżka: 10% przy terminowych wpłatach                                   │
│                                                                             │
│   Przypięty do: DebtPart "dp-aaa-111"                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

RepaymentPlan definiuje:
- **Jak naliczamy** - odsetki dzienne/miesięczne, od jakich komponentów
- **Jak wymagamy spłaty** - ile rat, w jakich terminach
- **Jak alokujemy wpłaty** - kolejność komponentów
- **Jakie modyfikatory** - zniżki, wakacje kredytowe, promocje

---

### Wiele planów na jeden DebtPart

**Jeden DebtPart może mieć wiele RepaymentPlanów** - to kluczowa elastyczność:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   DebtPart "dp-aaa-111" (dług 50 000 PLN)                                   │
│                                                                             │
│   ┌─────────────────────────┐  ┌─────────────────────────┐                  │
│   │  RepaymentPlan #1       │  │  RepaymentPlan #2       │                  │
│   │  Status: OFFERED        │  │  Status: OFFERED        │                  │
│   │                         │  │                         │                  │
│   │  "5 rat miesięcznych"   │  │  "12 rat miesięcznych"  │                  │
│   │  Rata: ~10 500 PLN      │  │  Rata: ~4 400 PLN       │                  │
│   │  Odsetki: 12%           │  │  Odsetki: 10%           │                  │
│   └─────────────────────────┘  └─────────────────────────┘                  │
│                                                                             │
│   Klient wybiera plan #2 → status zmienia się na ACTIVE                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Scenariusze użycia:**

| Scenariusz | Opis |
|------------|------|
| **Oferta wyboru** | Proces oferuje klientowi 2-3 warianty spłaty, klient wybiera |
| **Wiele aktywnych** | Klient może mieć aktywne plany na różne komponenty (kapitał vs odsetki) |
| **Zmiana planu** | Dezaktywacja starego, aktywacja nowego - historia zachowana |
| **Negocjacje** | Tworzenie kolejnych propozycji aż do akceptacji |

---

## 5. Payments

### Spłata na konkretny plan

Operator może wskazać **konkretny RepaymentPlan**, na który alokowana jest wpłata:

```
POST /api/payments
{
  "amount": 5000.00,
  "currency": "PLN",
  "debtorId": "jan-kowalski-id",
  "repaymentPlanId": "rp-12345",      // ← wskazany plan
  "paymentDate": "2024-03-15"
}
```

W tym przypadku wpłata trafia bezpośrednio na wskazany plan - alokacja według kolejności zdefiniowanej w planie.

---

### Spłata bez wskazania planu

Często wpłata przychodzi **bez informacji**, który plan spłacać - np. przelew bankowy z numerem umowy:

```
POST /api/payments
{
  "amount": 5000.00,
  "currency": "PLN",
  "debtorId": "jan-kowalski-id",
  "debtId": "KREDYT-2024-00123",    // ← tylko dług biznesowy
  "paymentDate": "2024-03-15"
}
```

**Problem:** Dłużnik ma wiele aktywnych RepaymentPlanów - który spłacać?

---

### Payment Resolution

**Payment Resolution** to komponent w module Billing odpowiedzialny za mapowanie wpłaty na RepaymentPlany:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Wpłata 5000 PLN od "jan-kowalski-id"                                      │
│   Debt ID: "KREDYT-2024-00123"                                            │
│   Brak wskazanego RepaymentPlan                                             │
│                                                                             │
│                            │                                                │
│                            ▼                                                │
│              ┌─────────────────────────────┐                                │
│              │     PAYMENT RESOLUTION      │                                │
│              │                             │                                │
│              │  Strategia: HIGHEST_DEBT    │                                │
│              │                             │                                │
│              └─────────────────────────────┘                                │
│                            │                                                │
│         ┌──────────────────┼──────────────────┐                             │
│         ▼                  ▼                  ▼                             │
│   ┌───────────┐      ┌───────────┐      ┌───────────┐                       │
│   │ Plan #1   │      │ Plan #2   │      │ Plan #3   │                       │
│   │ 30 000 PLN│ ◄──  │ 15 000 PLN│      │ 5 000 PLN │                       │
│   │ ACTIVE    │      │ ACTIVE    │      │ ACTIVE    │                       │
│   └───────────┘      └───────────┘      └───────────┘                       │
│                                                                             │
│   Wpłata trafia na Plan #1 (największe zadłużenie)                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Konfigurowalne strategie rozliczania:**

| Strategia | Opis |
|-----------|------|
| `HIGHEST_DEBT` | Najpierw spłacaj największe zadłużenie |
| `LOWEST_DEBT` | Najpierw spłacaj najmniejsze (szybkie zamknięcia) |
| `OLDEST_FIRST` | Najpierw najstarsze plany (wg daty utworzenia) |
| `NEWEST_FIRST` | Najpierw najnowsze plany |
| `HIGHEST_INTEREST` | Najpierw plany z najwyższym oprocentowaniem |
| `PROPORTIONAL` | Proporcjonalnie do salda każdego planu |
| `MANUAL_PRIORITY` | Wg ręcznie ustawionego priorytetu |

**API do zarządzania strategiami:**

```
# Ustawienie domyślnej strategii dla dłużnika
POST /api/payment-resolution/strategy
{
  "debtorId": "jan-kowalski-id",
  "strategy": "HIGHEST_DEBT"
}

# Ustawienie strategii dla konkretnego długu
POST /api/payment-resolution/strategy
{
  "debtId": "KREDYT-2024-00123",
  "strategy": "OLDEST_FIRST"
}

# Pobranie aktualnej strategii
GET /api/payment-resolution/strategy/{debtorId}

# Lista dostępnych strategii
GET /api/payment-resolution/strategies
```

**Priorytet strategii:**
1. Strategia per `debtId` (jeśli ustawiona)
2. Strategia per `debtorId` (jeśli ustawiona)
3. Domyślna strategia systemu (`HIGHEST_DEBT`)

**Rozszerzenie: Resolution na poziomie DebtPartów**

De facto Payment Resolution mógłby obsługiwać **dwa poziomy mapowania**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Wpłata od dłużnika                                                        │
│                                                                             │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────┐                                       │
│   │  POZIOM 1: DebtId → Plan        │  Strategia: który plan najpierw?      │
│   │  (pośrednio przez cząstki)      │  np. HIGHEST_DEBT, OLDEST_FIRST       │
│   └─────────────────────────────────┘                                       │
│         │                                                                   │
│         ▼                                                                   │
│   Alokacja na konkretny RepaymentPlan                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Gdy dłużnik ma wiele DebtPartów pod jednym `debtId`, Payment Resolution wybiera **który plan** spłacać - pośrednio przechodząc przez cząstki, ale z perspektywy API to jedno mapowanie: `debtId` → `RepaymentPlan`.

---

## 6. API

> **Uwaga:** Poniższe JSON-y to **wstępna propozycja** - nie jest to obecne API ani finalny kontrakt.
> Szczegóły będziemy doprecyzowywać wspólnie w stylu **Consumer-Driven Contracts**.

### Dostępne operacje

**DebtPart:**

| Operacja | Endpoint | Opis |
|----------|----------|------|
| [Tworzenie DebtPart](#tworzenie-debtpart) | `POST /api/debt-part` | Utworzenie nowej cząstki długu |
| [Solidarność](#solidarność---wielu-dłużników-odpowiada-za-tę-samą-część) | `POST /api/debt-part` | Wielu dłużników odpowiada za całość |
| [Niesolidarność](#niesolidarność---każdy-odpowiada-za-swoją-część) | `POST /api/debt-part` (×N) | Osobne DebtParty dla każdego |
| [Mieszana solidarność](#mieszana-solidarność---różna-per-komponent) | `POST /api/debt-part` (×N) | Różne modele per komponent |
| [Limity](#limity---ograniczenie-odpowiedzialności) | `POST /api/debt-part` | Ograniczenie odpowiedzialności właściciela |
| [Dodanie komponentu](#dodanie-nowego-komponentu) | `POST .../{id}/components` lub `POST .../by-debt/{debtId}/components` | Doliczenie kosztów (do cząstki lub wszystkich) |
| [Podział (Split)](#podział-debtpart-split) | `POST /api/debt-part/splitting/{id}` | Dzielenie długu (rozwód, cesja) |
| [Łączenie (Merge)](#łączenie-debtpartów-merge) | `POST /api/debt-part/merging` | Konsolidacja długów |
| [Stan cząstki](#sprawdzenie-stanu---kto-ile-jest-winien) | `GET /api/debt-part/find/{id}` | Stan konkretnej cząstki |
| [Stan całego długu](#sprawdzenie-stanu---kto-ile-jest-winien) | `GET .../find-by-debt/{debtId}/summary` | Zagregowany stan wszystkich cząstek |
| [Stan historyczny](#sprawdzenie-stanu-na-konkretną-datę) | `GET /api/debt-part/find/{id}?onDate=...` | Stan na konkretną datę |
| [Historia cząstki](#historia-transformacji-splitmerge) | `GET /api/debt-part/history/{id}` | Historia transformacji cząstki |
| [Historia długu](#historia-transformacji-splitmerge) | `GET .../history-by-debt/{debtId}` | Połączona historia wszystkich cząstek |
| [Wyszukiwanie po Debt ID](#wyszukiwanie-po-debt-id) | `GET /api/debt-part/find-by-debt/{debtId}` | Lista cząstek jednego długu |

> **Opcja `?includeCosts=true`:** Endpointy sprawdzania stanu (`find/{id}`, `find-by-debt/{debtId}`) mogą opcjonalnie zwracać koszty niewpływające na saldo. Dodaje pola `nonBalanceCosts` i `totalWithCosts` do odpowiedzi.

**Costs (niewpływające na saldo):**

| Operacja | Endpoint | Opis |
|----------|----------|------|
| Dodanie kosztu | `POST /api/costs` | Koszt do debtId lub debtPartId |
| Usunięcie kosztu | `DELETE /api/costs/{costId}` | Anulowanie kosztu |
| Lista kosztów (dług) | `GET /api/costs/by-debt/{debtId}` | Koszty całego długu |
| Lista kosztów (cząstka) | `GET /api/costs/by-debt-part/{debtPartId}` | Koszty konkretnej cząstki |

**ProductCatalog:**

| Operacja | Endpoint | Opis |
|----------|----------|------|
| Definiowanie ProductElement | `POST /api/product-catalog/elements` | Utworzenie konfiguracji (harmonogram, naliczanie, alokacja) |
| Definiowanie Product | `POST /api/product-catalog/products` | Zestaw elementów + reguły przejść |
| Definiowanie Condition | `POST /api/product-catalog/conditions` | Warunek przejścia między elementami |
| Produkt po nazwie | `GET /api/product-catalog/products/by-name/{name}` | Szczegóły produktu |
| Lista produktów | `GET /api/product-catalog/products` | Wszystkie dostępne produkty |
| Kategorie i typy | `GET /api/product-catalog/categories` | Dostępne kategorie elementów |

**RepaymentPlan:**

| Operacja | Endpoint | Opis |
|----------|----------|------|
| Tworzenie planu | `POST /api/repayment-plan` | Utworzenie nowego planu spłaty dla DebtPart |
| Aktywacja planu | `POST /api/repayment-plan/{id}/activate` | Zmiana statusu na ACTIVE |
| Dezaktywacja | `POST /api/repayment-plan/{id}/deactivate` | Zmiana statusu na CANCELLED |
| Ofertowanie | `POST /api/repayment-plan/{id}/offer` | Zmiana statusu na OFFERED |
| Lista planów | `GET /api/repayment-plan/by-debt-part/{debtPartId}` | Wszystkie plany dla danego DebtPart |
| Aktywne plany | `GET /api/repayment-plan/by-debt-part/{debtPartId}/active` | Tylko aktywne plany |
| Szczegóły planu | `GET /api/repayment-plan/{id}` | Stan i konfiguracja planu |

**Payments:**

| Operacja | Endpoint | Opis |
|----------|----------|------|
| Rejestracja wpłaty | `POST /api/payments` | Wpłata na plan lub dług (z resolution) |
| Ustawienie strategii | `POST /api/payment-resolution/strategy` | Strategia dla dłużnika lub długu |
| Pobranie strategii | `GET /api/payment-resolution/strategy/{debtorId}` | Aktualna strategia |
| Lista strategii | `GET /api/payment-resolution/strategies` | Dostępne strategie |
| Historia wpłat | `GET /api/payments/by-debt-part/{debtPartId}` | Wpłaty na cząstkę |
| Historia wpłat (dług) | `GET /api/payments/by-debt/{debtId}` | Wpłaty na cały dług |

**Billing:**

| Operacja | Endpoint / Event | Opis |
|----------|------------------|------|
| Obsługa płatności | `Event: PaymentMatched` | Billing nasłuchuje eventów z Payments |
| Naliczenie (on-demand) | `POST /api/billing/accrue` | Naliczenie obciążenia na RepaymentPlan |
| Naliczenie bezpośrednie | `POST /api/billing/accrue-direct` | Naliczenie na DebtPart (bez planu) |

> **Uwaga:** Komunikacja Payments → Billing odbywa się przez **eventy**, nie HTTP. Payments publikuje `PaymentMatched`, Billing subskrybuje i księguje.

---

#### Tworzenie DebtPart

Proces rozpoczyna obsługę długu tworząc DebtPart z komponentami (kapitał, odsetki, opłaty) i definicjami udziałów.

```
POST /api/debt-part
```

**Request:**

Proces przekazuje `debtId` - swój identyfikator długu biznesowego.

```json
{
  "debtId": "KREDYT-2024-00123",
  "name": "kredyt-hipoteczny-123",
  "components": {
    "PRINCIPAL": 50000.00,
    "INTEREST": 5000.00,
    "FEE": 500.00
  },
  "sharesDefinitions": [
    {
      "component": "PRINCIPAL",
      "percentage": 100,
      "owners": ["jan-kowalski-id"]
    },
    {
      "component": "INTEREST",
      "percentage": 100,
      "owners": ["jan-kowalski-id"]
    },
    {
      "component": "FEE",
      "percentage": 100,
      "owners": ["jan-kowalski-id"]
    }
  ]
}
```

**Response (201 Created):**

Core zwraca `debtPartId` - unikalny identyfikator cząstki. Proces zapamiętuje mapowanie: `debtPartId` ↔ `debtId`.

```json
{
  "debtPartId": "dp-aaaa-1111-xxxx",
  "debtId": "KREDYT-2024-00123"
}
```

---

#### Solidarność - wielu dłużników odpowiada za tę samą część

Gdy kilku dłużników odpowiada solidarnie (każdy za całość), podajemy ich razem w `owners`:

```json
{
  "debtId": "KREDYT-2024-00456",
  "name": "kredyt-solidarny-malzenstwo",
  "components": {
    "PRINCIPAL": 100000.00,
    "INTEREST": 10000.00
  },
  "sharesDefinitions": [
    {
      "component": "PRINCIPAL",
      "percentage": 100,
      "owners": ["jan-kowalski-id", "anna-kowalska-id"]
    },
    {
      "component": "INTEREST",
      "percentage": 100,
      "owners": ["jan-kowalski-id", "anna-kowalska-id"]
    }
  ]
}
```

Jan i Anna są solidarnie odpowiedzialni - każdy z nich widzi pełne 100 000 PLN kapitału i 10 000 PLN odsetek.

---

#### Niesolidarność - każdy odpowiada za swoją część

Gdy dłużnicy odpowiadają **niesolidarnie** (każdy tylko za swoją część), tworzymy **osobne DebtParty**.
Oba mają ten sam `debtId` - procesy wiedzą, że to ten sam dług biznesowy.

**DebtPart dla ABC Sp. z o.o. - 60% długu:**
```json
{
  "debtId": "LEASING-2024-00789",
  "name": "abc-spolka-udzial",
  "components": {
    "PRINCIPAL": 60000.00
  },
  "sharesDefinitions": [
    {
      "component": "PRINCIPAL",
      "percentage": 100,
      "owners": ["abc-spolka-id"]
    }
  ]
}
```

**DebtPart dla XYZ S.A. - 40% długu:**
```json
{
  "debtId": "LEASING-2024-00789",
  "name": "xyz-spolka-udzial",
  "components": {
    "PRINCIPAL": 40000.00
  },
  "sharesDefinitions": [
    {
      "component": "PRINCIPAL",
      "percentage": 100,
      "owners": ["xyz-spolka-id"]
    }
  ]
}
```

ABC Sp. z o.o. widzi tylko swoje 60 000 PLN, XYZ S.A. widzi tylko swoje 40 000 PLN.
Każdy DebtPart jest obsługiwany niezależnie - mogą mieć różne harmonogramy spłat, różne procesy.
Ten sam `debtId` pozwala odpytać Core o wszystkie cząstki tego długu.

> **Decyzja architektoniczna:** Niesolidarność modelujemy przez osobne DebtParty, nie przez
> wiele shares w jednym DebtPart. Dzięki temu każda część długu może być niezależnie
> splitowana, mergowana i obsługiwana przez różne procesy.

---

#### Mieszana solidarność - różna per komponent

Przykład: małżeństwo gdzie kapitał dzielony jest nierówno (np. z intercyzy), ale za odsetki odpowiadają solidarnie.

W tym przypadku tworzymy **dwa osobne DebtParty dla kapitału** (niesolidarność) i **jeden wspólny dla odsetek** (solidarność).
Wszystkie trzy mają ten sam `debtId`:

**Kapitał Jana (60%):**
```json
{
  "debtId": "KREDYT-2024-00456",
  "name": "jan-kapital",
  "components": { "PRINCIPAL": 60000.00 },
  "sharesDefinitions": [
    { "component": "PRINCIPAL", "percentage": 100, "owners": ["jan-kowalski-id"] }
  ]
}
```

**Kapitał Anny (40%):**
```json
{
  "debtId": "KREDYT-2024-00456",
  "name": "anna-kapital",
  "components": { "PRINCIPAL": 40000.00 },
  "sharesDefinitions": [
    { "component": "PRINCIPAL", "percentage": 100, "owners": ["anna-kowalska-id"] }
  ]
}
```

**Odsetki solidarnie (oboje):**
```json
{
  "debtId": "KREDYT-2024-00456",
  "name": "odsetki-solidarne",
  "components": { "INTEREST": 10000.00 },
  "sharesDefinitions": [
    { "component": "INTEREST", "percentage": 100, "owners": ["jan-kowalski-id", "anna-kowalska-id"] }
  ]
}
```

- Jan widzi: 60 000 PLN kapitału + 10 000 PLN odsetek = **70 000 PLN**
- Anna widzi: 40 000 PLN kapitału + 10 000 PLN odsetek = **50 000 PLN**

---

#### Limity - ograniczenie odpowiedzialności

Solidarność z limitem - poręczyciel (Marek Nowak) odpowiada solidarnie z głównym dłużnikiem, ale maksymalnie do 30 000 PLN:

```json
{
  "debtId": "KREDYT-2024-00999",
  "name": "kredyt-z-poreczeniem",
  "components": {
    "PRINCIPAL": 100000.00
  },
  "sharesDefinitions": [
    {
      "component": "PRINCIPAL",
      "percentage": 100,
      "owners": ["jan-kowalski-id", "marek-nowak-poreczyciel-id"],
      "limits": [
        {
          "ownerId": "marek-nowak-poreczyciel-id",
          "amount": 30000.00
        }
      ]
    }
  ]
}
```

- Jan Kowalski (główny dłużnik, bez limitu) widzi: 100 000 PLN
- Marek Nowak (poręczyciel, z limitem) widzi: 30 000 PLN
- Jeśli Jan nie zapłaci, Marek odpowiada tylko do 30 000 PLN
- Nadwyżka ponad limit solidarnych trafia na LossAccount

---

#### Dodanie nowego komponentu

Gdy trzeba doliczyć nowy składnik (np. koszty sądowe, karę).

**Wariant 1: Do konkretnego DebtPart**
```
POST /api/debt-part/{debtPartId}/components
```

**Wariant 2: Do wszystkich cząstek danego długu**
```
POST /api/debt-part/by-debt/{debtId}/components
```

Użycie `debtId` doda komponent do **wszystkich** DebtPartów pod tym długiem biznesowym.

**Request:**
```json
{
  "components": {
    "COURT_COSTS": 2500.00
  },
  "sharesDefinitions": [
    {
      "component": "COURT_COSTS",
      "percentage": 100,
      "owners": ["jan-kowalski-id"]
    }
  ]
}
```

---

#### Podział DebtPart (Split)

Dzielenie długu na części - np. przy rozwodzie, cesji, ugodzie częściowej.

**Przykład:** Po rozwodzie dług solidarny małżeństwa dzielimy na dwa osobne zobowiązania.

```
POST /api/debt-part/splitting/{debtPartId}
```

**Request:**
```json
{
  "debtPartsRequests": [
    {
      "name": "jan-kowalski-po-rozwodzie",
      "components": {
        "PRINCIPAL": 60000.00,
        "INTEREST": 6000.00
      },
      "sharesDefinitions": [
        { "component": "PRINCIPAL", "percentage": 100, "owners": ["jan-kowalski-id"] },
        { "component": "INTEREST", "percentage": 100, "owners": ["jan-kowalski-id"] }
      ]
    },
    {
      "name": "anna-kowalska-po-rozwodzie",
      "components": {
        "PRINCIPAL": 40000.00,
        "INTEREST": 4000.00
      },
      "sharesDefinitions": [
        { "component": "PRINCIPAL", "percentage": 100, "owners": ["anna-kowalska-id"] },
        { "component": "INTEREST", "percentage": 100, "owners": ["anna-kowalska-id"] }
      ]
    }
  ]
}
```

**Response (201 Created):**

Domyślnie nowe DebtParty **dziedziczą `debtId`** z oryginału. Można też przekazać nowy `debtId` w request, jeśli split tworzy nowy dług biznesowy (np. cesja na inny podmiot).

```json
{
  "debtParts": [
    {
      "debtPartId": "dp-jan-rozwod-001",
      "debtId": "KREDYT-2024-00456",
      "name": "jan-kowalski-po-rozwodzie",
      "mainBalances": [
        { "component": "PRINCIPAL", "amount": -60000.00 },
        { "component": "INTEREST", "amount": -6000.00 }
      ]
    },
    {
      "debtPartId": "dp-anna-rozwod-002",
      "debtId": "KREDYT-2024-00456",
      "name": "anna-kowalska-po-rozwodzie",
      "mainBalances": [
        { "component": "PRINCIPAL", "amount": -40000.00 },
        { "component": "INTEREST", "amount": -4000.00 }
      ]
    }
  ]
}
```

> **Uwaga:** Suma wartości musi się zgadzać z oryginałem. Dla przypadków z umorzeniem użyj `/splitting/{id}/unchecked`.

---

#### Łączenie DebtPartów (Merge)

Konsolidacja wielu cząstek długów w jeden - np. wspólna ugoda, zakup portfela.

**Przykład:** Klient (Tomasz Wiśniewski) ma kilka osobnych długów, które łączymy w jeden dla łatwiejszej obsługi ugody.

```
POST /api/debt-part/merging
```

**Request:**

Proces **może** podać nowy `debtId` dla skonsolidowanego długu (np. przy konsolidacji różnych długów w ugodę). Jeśli łączymy cząstki tego samego długu, można zachować oryginalny `debtId`.

```json
{
  "debtId": "UGODA-2024-00001",
  "mergeUnderName": "tomasz-wisniewski-konsolidacja",
  "idsToMerge": [
    "dp-kredyt-gotowkowy-111",
    "dp-karta-kredytowa-222"
  ],
  "mergedShares": [
    { "component": "PRINCIPAL", "percentage": 100, "owners": ["tomasz-wisniewski-id"] },
    { "component": "INTEREST", "percentage": 100, "owners": ["tomasz-wisniewski-id"] },
    { "component": "FEE", "percentage": 100, "owners": ["tomasz-wisniewski-id"] }
  ]
}
```

**Response (201 Created):**
```json
{
  "debtPart": {
    "debtPartId": "dp-konsolidacja-001",
    "debtId": "UGODA-2024-00001",
    "name": "tomasz-wisniewski-konsolidacja",
    "mainBalances": [
      { "component": "PRINCIPAL", "amount": -45000.00 },
      { "component": "INTEREST", "amount": -5500.00 },
      { "component": "FEE", "amount": -800.00 }
    ]
  }
}
```

---

#### Sprawdzenie stanu - kto ile jest winien

**Wariant 1: Stan konkretnej cząstki**
```
GET /api/debt-part/find/{debtPartId}
```

**Wariant 2: Zagregowany stan całego długu biznesowego**
```
GET /api/debt-part/find-by-debt/{debtId}/summary
```

Zwraca sumę wszystkich cząstek pod danym `debtId` - widok całego długu biznesowego.

**Response (dla pojedynczej cząstki):**
```json
{
  "debtPartId": "dp-aaaa-1111-xxxx",
  "debtId": "KREDYT-2024-00123",
  "name": "kredyt-hipoteczny-123",
  "mainBalances": [
    { "component": "PRINCIPAL", "amount": -50000.00 },
    { "component": "INTEREST", "amount": -5000.00 },
    { "component": "FEE", "amount": -500.00 }
  ],
  "ownerBalances": [
    { "ownerId": "jan-kowalski-id", "component": "PRINCIPAL", "amount": -50000.00 },
    { "ownerId": "jan-kowalski-id", "component": "INTEREST", "amount": -5000.00 },
    { "ownerId": "jan-kowalski-id", "component": "FEE", "amount": -500.00 }
  ],
  "total": { "amount": -55500.00, "currency": "PLN" }
}
```

---

#### Sprawdzenie stanu na konkretną datę

Stan historyczny - ile było winien w danym momencie:

```
GET /api/debt-part/find/{debtPartId}?onDate=2024-06-15T00:00:00Z
```

Zwraca stan DebtPart na dzień 15 czerwca 2024.

---

#### Historia transformacji (Split/Merge)

Pełna genealogia DebtPart - skąd powstał, na co się podzielił.

**Wariant 1: Historia konkretnej cząstki**
```
GET /api/debt-part/history/{debtPartId}
```

**Wariant 2: Połączona historia całego długu biznesowego**
```
GET /api/debt-part/history-by-debt/{debtId}
```

Zwraca sumę historii wszystkich cząstek pod danym `debtId` - pełna historia transformacji długu.

**Response:**
```json
{
  "transitions": [
    [
      {
        "from": "dp-solidarny-malzenstwo-001",
        "to": "dp-jan-rozwod-001",
        "description": "split",
        "occurredAt": "2024-03-15T10:30:00Z"
      },
      {
        "from": "dp-solidarny-malzenstwo-001",
        "to": "dp-anna-rozwod-002",
        "description": "split",
        "occurredAt": "2024-03-15T10:30:00Z"
      }
    ]
  ]
}
```

---

#### Wyszukiwanie po Debt ID

Pobranie wszystkich DebtPartów należących do tego samego długu biznesowego:

```
GET /api/debt-part/find-by-debt/{debtId}
```

**Przykład:**
```
GET /api/debt-part/find-by-debt/KREDYT-2024-00456
```

**Response:**
```json
{
  "debtId": "KREDYT-2024-00456",
  "debtParts": [
    {
      "debtPartId": "dp-jan-rozwod-001",
      "name": "jan-kowalski-po-rozwodzie",
      "total": { "amount": -66000.00, "currency": "PLN" }
    },
    {
      "debtPartId": "dp-anna-rozwod-002",
      "name": "anna-kowalska-po-rozwodzie",
      "total": { "amount": -44000.00, "currency": "PLN" }
    }
  ]
}
```

Kluczowe dla procesów - pozwala odpytać, które cząstki należą do tego samego długu.

---

## 7. Q&A - Często zadawane pytania

> **Uwaga:** Proponowane w tej sekcji JSON-y i API to **wstęp do rozmowy**, nie obietnica kontraktu.
> Celem jest pokazanie kierunku myślenia - szczegóły będziemy dopracowywać wspólnie.

### Obciążenia i produkty

**Q: Jak przypisać obciążenie (wpływające na saldo) do DebtPart?**

A: Obciążenie to coś, co musimy:
1. **Wiedzieć jak policzyć** - stała kwota, procent, złożone wyliczenie
2. **Wiedzieć na jaki komponent dać** - FEE, COURT_COSTS, itp.
3. **Zaudytować** - kto, kiedy, dlaczego naliczył
4. **Obsłużyć w Billing** - Billing musi wiedzieć jak to traktować przy spłacie

To wszystko = **Product**. Billing będzie miał wiele typowych produktów, które automatycznie obsłuży jako "nalicz X na komponent Y".

**Co proces może skonfigurować:**
- **Jak liczyć** - stała kwota, procent od salda, złożona formuła
- **Na jaki komponent** - FEE, ADMIN_COST, REMINDER_FEE
- **Kiedy naliczyć** - jednorazowo (on-demand), cyklicznie (co miesiąc)


**Kroki do naliczenia obciążenia:**

```
# KROK 1: Zdefiniuj produkt (raz, reużywalny i jeśli go nie ma)
POST /api/product-catalog/elements
{
  "name": "REMINDER_FEE_15",
  "parameters": {
    "category": "Accrual",
    "targetComponent": "FEE",
    "schedule": "one-time (date)",
    "calculator": "fixed-15-pln"
  }
}

# KROK 2: Dodaj produkt do istniejącego RepaymentPlan
POST /api/repayment-plan/{planId}/add-element
{
  "productElementName": "REMINDER_FEE_15"
}

# KROK 3: Wywołaj naliczenie (dla produktów ON_DEMAND)
POST /api/billing/accrue
{
  "repaymentPlanId": "rp-123",
  "productElementName": "REMINDER_FEE_15",
  "reason": "Wysłano monit SMS"
}
```

**A co jeśli nie ma RepaymentPlan?**

Może być tak, że DebtPart nie ma jeszcze RepaymentPlan (np. dług świeżo utworzony, czeka na ofertę), a mimo to chcemy naliczać koszty. Billing zostanie rozszerzony o taką możliwość:

```
# Naliczenie bezpośrednio na DebtPart (bez RepaymentPlan)
POST /api/billing/accrue-direct
{
  "debtPartId": "dp-aaa-111",
  "productElementName": "ADMIN_FEE_50",
  "reason": "Opłata za założenie sprawy"
}
```

Billing utworzy wpis księgowy na wskazanym komponencie, nawet bez aktywnego planu spłaty.

---

### DebtPart i komponenty

**Q: Jak przekazać składowe sądowe salda (kapitał zasądzony, odsetki zasądzone, koszty sądowe)?**

A: Wywołaj utworzenie DebtPart z konkretnymi komponentami:

```
POST /api/debt-part
{
  "debtId": "SPRAWA-SADOWA-2024-001",
  "name": "wyrok-kowalski",
  "components": {
    "PRINCIPAL_ADJUDICATED": 50000.00,
    "INTEREST_ADJUDICATED": 8500.00,
    "COURT_COSTS": 2400.00,
    "LEGAL_REPRESENTATION": 3600.00
  },
  "sharesDefinitions": [
    { "component": "PRINCIPAL", "percentage": 100, "owners": ["jan-kowalski-id"] },
    { "component": "INTEREST", "percentage": 100, "owners": ["jan-kowalski-id"] },
    { "component": "COURT_COSTS", "percentage": 100, "owners": ["jan-kowalski-id"] },
    { "component": "LEGAL", "percentage": 100, "owners": ["jan-kowalski-id"] }
  ]
}
```

Nazwy komponentów są elastyczne - proces definiuje jakie komponenty potrzebuje. Core traktuje je jako osobne "szufladki" na kwoty.

---

### Płatności

**Q: Jak powiązać płatność z DebtPartem?**

**1. Ręczne wskazanie DebtPart/RepaymentPlan:**

Proces rejestrując wpłatę w Payments może jawnie wskazać cel:

```

{
  "amount": 5000.00,
  "debtPartId": "dp-aaa-111",           // opcjonalnie
  "repaymentPlanId": "rp-456",          // opcjonalnie
  "paidBy": "jan-kowalski-id",
  "paidAt": "2024-03-15"
}
```

Payments opublikuje `PaymentMatched` z tymi danymi → Billing zaksięguje na wskazany plan.

**2. Bez wskazania - Payment Resolution:**

Jeśli nie podano `debtPartId` ani `repaymentPlanId`:

```

{
  "amount": 5000.00,
  "debtId": "KREDYT-2024-00123",      // tylko dług biznesowy
  "paidBy": "jan-kowalski-id",
  "paidAt": "2024-03-15"
}
```

Billing odbierze event i użyje **Payment Resolution** z konfigurowalnymi strategiami (HIGHEST_DEBT, OLDEST_FIRST, itp.) do wyboru planu. Szczegóły: [Payment Resolution](#payment-resolution) w sekcji Payments.

---

### Dodawanie produktów do katalogu

**Q: Jak dodawać produkty do katalogu?**

A: Patrz:
- **API** → sekcja [Dostępne operacje](#dostępne-operacje) → ProductCatalog
- **Przykłady** → [Konfiguracja obsługi finansowej](#konfiguracja-obsługi-finansowej) poniżej
- **Tunowanie produktów** → [Konfiguracja obsługi finansowej](#konfiguracja-obsługi-finansowej) → "Produkty będą bogate i konfigurowalne"

---

### Konfiguracja obsługi finansowej

**Q: Jak skonfigurować obsługę finansową sprawy (RepaymentPlan)?**

A: W dwóch krokach:

**1. Skonfiguruj produkty** - zdefiniuj elementy które będą używane:

```
POST /api/product-catalog/elements
{
  "name": "HARMONOGRAM_5_RAT",
  "parameters": {
    "category": "Schedule",
    "period": "Months",
    "count": 5
  }
}
```

**2. Utwórz RepaymentPlan** - przypisz produkty do DebtPart:

```
POST /api/repayment-plan
{
  "debtPartId": "dp-aaa-111",
  "name": "plan-5-rat-standardowy",
  "productElements": ["HARMONOGRAM_5_RAT", "ODSETKI_6_PROCENT", "ALOKACJA_ODSETKI_NAJPIERW"]
}
```

Szczegóły API: sekcja [Dostępne operacje](#dostępne-operacje) → ProductCatalog i RepaymentPlan.

**Produkty będą bogate i konfigurowalne:**

Nie trzeba tworzyć nowego produktu na każdy wariant. Produkty można "tunować" przez parametry:

```
POST /api/product-catalog/elements
{
  "name": "HARMONOGRAM_ELASTYCZNY",
  "parameters": {
    "category": "Schedule",
    "period": "Months",           // można zmienić na "Weeks"
    "count": 5,                   // można zmienić liczbę rat
    "startDelay": 14,             // dni opóźnienia pierwszej raty
    "holidaysCalendar": "PL"      // można zmienić na "RO"
  }
}
```

Jeden produkt, wiele konfiguracji - procesy dostosowują parametry do swoich potrzeb.

---

### Warunki i zmiany stóp procentowych

**Q: Jak obsługiwać warunki w produktach? Co gdy zmieniają się stopy odsetkowe?**

A: Warunki (Condition) pozwalają na automatyczne przejścia między produktami:

```
# Przykład: po 3 terminowych wpłatach → niższe odsetki
POST /api/product-catalog/conditions
{
  "name": "3_TERMINOWE_WPLATY",
  "type": "RepaymentsOnTime",
  "parameters": { "counter": 3 }
}
```

**Lista warunków jest doprecyzowywana wspólnie** - nie może być dowolna, bo Billing musi je zrozumieć i zaprogramować obsługę. Obecne typy:

| Typ | Opis |
|-----|------|
| `RepaymentsOnTime` | N terminowych wpłat |
| `RepaymentsDelayed` | N opóźnionych wpłat |
| `ManualConfirmation` | Ręczne potwierdzenie operatora |

Nowe typy warunków dodajemy po uzgodnieniu z zespołem Core.

---

**Przypadek: zmieniły się stopy odsetkowe**

Gdy zmienia się coś wpływającego na wiele produktów (np. stopa referencyjna NBP), **wystarczy jeden strzał do jednego miejsca**:

```
# Aktualizacja cennika (albo produktu, do ustalenia)
PUT /api/pricing/calculators/simple-interest-statutory
{
  "parameters": {
    "annualRate": 11.25    // nowa stopa ustawowa
  }
}
```

**Dlaczego to działa?**

Produkty odwołują się do cennika przez nazwę (`calculator: "simple-interest-statutory"`), nie przez skopiowaną wartość. Zmiana w cenniku automatycznie wpływa na wszystkie produkty które go używają.

Alternatywnie, produkty mogą korzystać z **wartości domyślnych w ProductType**:

```
ProductType "POLSKI_STANDARD" {
  statutoryInterestRate: 11.25%    // ← zmiana tu
  maxRate: 2x referencyjna NBP
  holidaysCalendar: PL
}

ProductElement "ODSETKI_USTAWOWE" {
  category: "Accrual",
  calculator: "simple-interest",
  rate: "@statutoryInterestRate"   // ← pobiera z ProductType
}
```

Jedna zmiana w ProductType → automatyczna propagacja do wszystkich produktów w tej rodzinie.
Czy to będzie zmiana cennika czy producttype - to wewnętrzna decyzja zespołu core.

---

### Pobieranie danych finansowych (DWH, Controlling)

**Q: Jak pobrać dane finansowe z Core? Jak pobrać wartości finansowe dla portfela?**

A: Obecnie dostępne widoki:
- **Historia długu/cząstki** → [Historia i audyt](#historia-i-audyt)
- **Stan na dowolny moment** → `GET /api/debt-part/find/{id}?onDate=...`
- **Zagregowany stan** → `GET /api/debt-part/find-by-debt/{debtId}/summary`

**Jeśli potrzebne dane nie są jeszcze widoczne** w istniejących widokach - sprecyzujcie jakie dane potrzebujecie, a Core je wystawi.

> **Uwaga dla Controllingu:** Jeśli dane mają być agregowane per portfel lub w innych przekrojach analitycznych, prawdopodobnie powinny iść przez hurtownię (DWH), która pobierze surowe dane z Core i przygotuje odpowiednie widoki raportowe.

---

### Przykłady przepływów (pseudo-kod)

**Q: Jak wygląda kod rozpoczęcia i zakończenia postępowań?**

---

**a) Rozpoczęcie postępowania polubownego po imporcie**

To po prostu utworzenie DebtPart:

```
// Proces polubowny importuje dług i tworzy cząstkę
POST /api/debt-part
{
  "debtId": "KREDYT-2024-00123",
  "name": "polubowne-kowalski",
  "components": {
    "PRINCIPAL": 50000.00,
    "INTEREST": 5000.00
  },
  "sharesDefinitions": [...]
}
```

---

**b) Rozpoczęcie postępowania sądowego (2 stare obciążenia + 1 stary koszt + 1 nowe obciążenie + skapitalizowane odsetki)**

Również utworzenie DebtPart - proces po swojej stronie wie jakie kwoty zebrać:

```
// Proces sądowy zbiera dane i tworzy cząstkę
POST /api/debt-part
{
  "debtId": "KREDYT-2024-00123",
  "name": "sadowe-kowalski",
  "components": {
    "PRINCIPAL": 50000.00,           // stare obciążenie 1
    "INTEREST_OLD": 3000.00,         // stare obciążenie 2
    "INTEREST_CAPITALIZED": 2500.00, // skapitalizowane odsetki
    "COURT_FEE": 1500.00             // nowe obciążenie
  },
  "sharesDefinitions": [...]
}

// Opcjonalnie: dodanie kosztu niewpływającego na saldo
POST /api/costs
{
  "debtPartId": "dp-sadowe-001",
  "type": "CORRESPONDENCE",
  "amount": 50.00,
  "description": "Stary koszt korespondencji"
}
```

Proces wie skąd wziąć kwoty (poprzedni system, excel, itp.) - Core tylko przyjmuje gotowe wartości.

---

**c) Zakończenie postępowania sądowego z pełnoprawnym wyrokiem**

Co oznacza "zakończyć"?

- Czy DebtPart **nie może już być obsługiwany**? → patrz [Archiwizacja długu](#archiwizacja-długu)
- Czy chodzi o **korektę salda** według wyroku? → patrz [Korekta salda po wyroku/nakazie](#korekta-salda-po-wyrokunakazie)
- Czy chodzi o **utworzenie nowego DebtPart** z kwotami z wyroku?

Jeśli wyrok określa nowe kwoty do dochodzenia:

```
// Opcja 1: Korekta istniejącego DebtPart
POST /api/debt-part/{id}/components
{
  "components": {
    "PRINCIPAL_ADJUDICATED": 50000.00,
    "INTEREST_ADJUDICATED": 8500.00,
    "COURT_COSTS": 2400.00,
    "LEGAL_REPRESENTATION": 3600.00
  }
}

// Opcja 2: Nowy DebtPart (np. egzekucyjny) ze starego
POST /api/debt-part/splitting/{id}
{
  "debtPartsRequests": [
    {
      "name": "egzekucyjne-kowalski-wyrok",
      "components": {
        "PRINCIPAL_ADJUDICATED": 50000.00,
        ...
      }
    }
  ]
}
```

Sprecyzujcie co dokładnie oznacza "zakończenie" w waszym procesie.

---

**d) Równoległe postępowania: polubowne 1000 PLN + komornicze 800 PLN → później +200 PLN**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Debt ID: "KREDYT-2024-00123"                                            │
│                                                                             │
│   ┌─────────────────────────────┐    ┌─────────────────────────────┐        │
│   │  DebtPart: dp-polubowne-001 │    │  DebtPart: dp-komornicze-001│        │
│   │                             │    │                             │        │
│   │  PRINCIPAL: 1000 PLN        │    │  PRINCIPAL: 800 PLN         │        │
│   │                             │    │  (później +200 PLN)         │        │
│   │  Status: AKTYWNE            │    │  Status: AKTYWNE            │        │
│   │  Proces: POLUBOWNY          │    │  Proces: KOMORNICZY         │        │
│   └─────────────────────────────┘    └─────────────────────────────┘        │
│                                                                             │
│   Oba postępowania równolegle windykują tego samego dłużnika.               │
│   Ten sam debtId → procesy wiedzą, że to ten sam dług.                    │
│                                                                             │
│   Timeline:                                                                 │
│   ─────────────────────────────────────────────────────────────────────     │
│   T1: Polubowne 1000 PLN ─────────────────────────────────────────────►     │
│   T2:              Komornicze 800 PLN ────────────────────────────────►     │
│   T3:                            +200 PLN (potwierdzenie kosztów) ────►     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

To zarządzanie DebtPartami z tym samym `debtId`:

```
// Krok 1: Polubowne na pełną kwotę
POST /api/debt-part
{
  "debtId": "KREDYT-2024-00123",
  "name": "polubowne-kowalski",
  "components": { "PRINCIPAL": 1000.00 }
}
// → dp-polubowne-001

// Krok 2: Komornicze na 800 PLN (równolegle)
POST /api/debt-part
{
  "debtId": "KREDYT-2024-00123",   // ten sam debtId!
  "name": "komornicze-kowalski",
  "components": { "PRINCIPAL": 800.00 }
}
// → dp-komornicze-001

// Krok 3: Po otrzymaniu potwierdzenia kosztów - zwiększenie o 200 PLN
POST /api/debt-part/dp-komornicze-001/components
{
  "components": { "ENFORCEMENT_COSTS": 200.00 }
}
```

**Synchronizacja jest po stronie procesów** - to procesy wiedzą kiedy i co stworzyć/zaktualizować.

---

**Uwaga o operacjach wielokrokowych:**

Dla operacji wymagających wielu strzałów, np.:
- Zdefiniuj 3 produkty
- Zamknij stary DebtPart
- Otwórz nowy z innymi komponentami
- Przypisz RepaymentPlan

...gdyby takie sekwencje się powtarzały, Core **mógłby rozważyć** wystawienie dodatkowego komponentu (nad istniejącymi modułami), do którego procesy uderzają z jednym wywołaniem.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Proces                                                                    │
│      │                                                                      │
│      ▼                                                                      │
│   ┌─────────────────────────────┐                                           │
│   │  Fasada "Złożone operacje"  │  ← jeden strzał                           │
│   │  (hipotetyczna)             │                                           │
│   └─────────────────────────────┘                                           │
│      │                                                                      │
│      ├──► ProductCatalog (3 produkty)                                       │
│      ├──► DebtPart (zamknięcie + otwarcie)                                  │
│      └──► RepaymentPlan (przypisanie)                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Sugerujemy jednak tego NIE robić.** Dlaczego?

Taka logika będzie **specyficzna dla procesów krajowych** - co dla Polski znaczy "rozpocznij sądówkę", dla Rumunii może znaczyć coś innego. Wprowadzenie takiej fasady:
- Zmieni filozofię modułu (Core zaczyna rozumieć procesy krajowe)
- Spowoduje eksplozję endpoint'ów per kraj/proces
- Wrócimy do anty-wzorca opisanego w sekcji [Anty-wzorzec: Core jako suma przypadków](#anty-wzorzec-core-jako-suma-przypadków)

**Rekomendacja:** Procesy orkiestrują wywołania po swojej stronie. Core dostarcza atomowe operacje.

---

### Korekta salda po wyroku/nakazie

**Q: Jak skorygować saldo po otrzymaniu wyroku lub nakazu zapłaty?**

A: Zależy od stanu DebtPart:

**Scenariusz 1: Brak aktywnego RepaymentPlan**

Prosta sytuacja - można dodać/zmienić kwoty na komponentach:

```
POST /api/debt-part/{id}/components
{
  "components": {
    "COURT_COSTS": 2400.00,
    "LEGAL_REPRESENTATION": 3600.00
  }
}
```

Core może to obsłużyć jako:
- **Edycja DebtPart** - modyfikacja z pełną historią zmian
- **Nowy DebtPart ze starego** - split z korektą (zachowuje genealogię)

+ ewentualna korekta odsetek

**Scenariusz 2: Aktywny RepaymentPlan**

Bardziej skomplikowane - do której raty dodać nowe kwoty?

**Do ustalenia wspólnie:**

| Aspekt | Pytanie |
|--------|---------|
| **Czy zawsze można?** | Korekta możliwa zawsze, czy tylko bez aktywnego planu? |
| **Strategia alokacji** | Jak rozłożyć kwotę na raty? Proporcjonalnie? Do ostatniej raty? Nowa rata? |
| **Przeliczenie planu** | Czy aktywny plan ma się przeliczyć automatycznie? |
| **Audit** | Jak oznaczyć korektę w historii? (wyrok, nakaz, błąd poprzedniego systemu) |

Jeśli korekta ma być możliwa przy aktywnym planie, Core może wystawić strategie:

```
POST /api/debt-part/{id}/balance-correction
{
  "components": {
    "COURT_COSTS": 2400.00
  },
  "reason": "Nakaz zapłaty z dnia 2024-03-15",
  "allocationStrategy": "TO_LAST_INSTALLMENT"  // lub PROPORTIONAL, NEW_INSTALLMENT
}
```
+ ewentualna korekta odsetek
Sprecyzujcie wymagania, a Core doda odpowiednie API.

---

### Archiwizacja długu

**Q: Jak zarchiwizować dług?**

A: Najpierw musimy ustalić co oznacza "archiwizacja":

- Czy to znaczy, że długu **nie można już obsługiwać** i Core odmówi przyszłych operacji?
- Czy zarchiwizowany dług ma być **widoczny w historii**, ale nieaktywny?
- Czy archiwizacja jest **odwracalna**?

**Jeśli archiwizacja = blokada dalszej obsługi**, Core może wystawić taką opcję:

```
POST /api/debt-part/{id}/archive
{
  "reason": "Dług przedawniony"
}
```

**Do ustalenia wspólnie:**

| Aspekt | Pytanie |
|--------|---------|
| **Warunki** | Kiedy można archiwizować? Zawsze? Tylko gdy brak aktywnego RepaymentPlan? |
| **Blokada** | Jakie operacje blokujemy? Wszystkie? Tylko nowe plany? |
| **Odwracalność** | Czy można "odarchiwizować"? Pod jakimi warunkami? |
| **Historia** | Czy zarchiwizowany dług jest widoczny w widokach historycznych? |

Sprecyzujcie wymagania, a Core doda odpowiednie API.

---

### Pełna historia długu od importu

**Q: Jak prześledzić całą historię długu od momentu importu?**

A: Widok historii cząstek/długu to zapewnia. Patrz [Historia i audyt](#historia-i-audyt) poniżej:
- **Cały dług** → `GET /api/history/by-debt/{debtId}` - historia wszystkich cząstek
- **Graf transformacji** → `GET /api/debt-part/history/{id}` - skąd powstała, na co się podzieliła
- **Stan na dowolny moment** → `GET /api/debt-part/find/{id}?onDate=...`

---

### Śledzenie przejścia między procesami

**Q: Jak śledzić przejście z DebtPart polubownego na sądowy (np. przy złożeniu pozwu)?**

A: Widok historii cząstek/długu to zapewnia. Patrz [Historia i audyt](#historia-i-audyt) poniżej:
- **Graf transformacji** → `GET /api/debt-part/history/{id}` - pokaże split/merge między cząstkami
- **Ten sam debtId** → obie cząstki (polubowna i sądowa) mają wspólny `debtId`, więc można odpytać o całą historię długu: `GET /api/history/by-debt/{debtId}`

Przykład: DebtPart polubowny `dp-polubowny-001` został zamknięty, utworzono `dp-sadowy-002` - historia pokaże tę transformację.

---

### Historia i audyt

**Q: Jak śledzić zmienność i historię DebtPart?**

A: Podstawowe operacje są już opisane w sekcji [Dostępne operacje](#dostępne-operacje):
- **Historia transformacji (graf)** - `GET /api/debt-part/history/{id}` - skąd cząstka powstała, na co się podzieliła
- **Stan na dowolny moment** - `GET /api/debt-part/find/{id}?onDate=...` - ile było winien w danej dacie

Dodatkowo Core wystawia rozszerzony widok stanu na dwóch poziomach:

| Poziom | Endpoint | Co zawiera |
|--------|----------|------------|
| Cały dług | `GET /api/history/by-debt/{debtId}` | Stan wszystkich cząstek danego długu |
| Cząstka | `GET /api/history/by-debt-part/{debtPartId}` | Stan konkretnej cząstki |

**Stan na konkretną datę:**

Domyślnie endpoint zwraca **aktualny stan**. Można też odpytać o stan historyczny:

```
GET /api/history/by-debt-part/{debtPartId}?onDate=2024-06-15
```

Wtedy odpowiedź uwzględnia wszystkie operacje do podanej daty (włącznie).

**Co będzie widoczne w widoku:**

- **Produkty** - przypisane RepaymentPlany, aktywne ProductElementy
- **Stany kont** - salda komponentów (PRINCIPAL, INTEREST, FEE...)
- **Płatności** - przeprocesowany wpłaty w tym płatności sprzed cutoff (przeniesione z poprzedniego systemu)
- **Koszty** - 
- **Transformacje** - z jakiej cząstki powstała (split) lub w jaką została włączona (merge)

**Przykład odpowiedzi API:**

```json
{
  "debtPartId": "dp-aaa-111",
  "debtId": "KREDYT-2024-00123",
  "asOfDate": "2024-03-15",
  "balances": {
    "PRINCIPAL": -45000.00,
    "INTEREST": -5250.00,
    "FEE": -500.00
  },
  "total": -50750.00,
  "repaymentPlans": [
    { "id": "rp-5-rat", "name": "5-rat-miesiecznie", "status": "ACTIVE" }
  ],
  "paymentsTotal": 10000.00,
  "paymentsBeforeCutoff": 5000.00,
  "nonBalanceCosts": 150.00,
  "origin": {
    "type": "SPLIT",
    "sourceDebtPartId": "dp-original-001",
    "occurredAt": "2024-01-10"
  }
}
```

---



# APPlikacja

> Mobilna aplikacja do nauki prawa poprzez krótkie, interaktywne pytania testowe.

[![Android](https://img.shields.io/badge/Android-28%2B-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.0-7F52FF?logo=kotlin&logoColor=white)](https://kotlinlang.org/)
[![Jetpack Compose](https://img.shields.io/badge/Jetpack_Compose-Material_3-4285F4?logo=jetpackcompose&logoColor=white)](https://developer.android.com/compose)
[![Firebase](https://img.shields.io/badge/Firebase-Auth_%7C_Storage_%7C_Firestore-FFCA28?logo=firebase&logoColor=black)](https://firebase.google.com/)
[![Room](https://img.shields.io/badge/Room-offline--first-2F6F61)](https://developer.android.com/training/data-storage/room)

APPlikacja pomaga przygotowywać się do prawniczych egzaminów testowych. Łączy
mechanikę pionowo przewijanych „rolek” z wiarygodnym źródłem prawnym, nauką
offline oraz wersjonowanymi pakietami pytań pobieranymi na żądanie.

Projekt powstał jako natywna aplikacja Android w Kotlinie i Jetpack Compose.
Kod źródłowy pozostaje w prywatnym repozytorium — tutaj prezentuję działający
produkt, jego architekturę oraz najważniejsze decyzje techniczne.

## Demo

<!--
Przed publikacją:
1. Dodaj nagranie jako media/applikacja-demo.mp4.
2. Dodaj pionowy kadr z nagrania jako media/demo-cover.png.
3. Najlepszą zgodność przeglądarek zapewni MP4 zakodowany jako H.264.
-->

[![Obejrzyj demo APPlikacji](media/demo-cover.png)](media/applikacja-demo.mp4)

**[▶ Odtwórz pełne demo](media/applikacja-demo.mp4)**

W demonstracji pokazuję logowanie, pobranie materiałów, naukę w formie rolek,
sprawdzenie odpowiedzi oraz gest przełączający pytanie na treść właściwego
przepisu.

## Najważniejsze możliwości

- logowanie przez e-mail i Google z wykorzystaniem Firebase Authentication,
- katalog materiałów pobierany dynamicznie z Firebase Storage,
- pobieranie wybranych, wersjonowanych pakietów pytań `.json.gz`,
- kontrola integralności paczki przez rozmiar pliku i sumę SHA-256,
- rozpakowanie, walidacja i transakcyjny import poza głównym wątkiem,
- lokalna baza Room umożliwiająca naukę bez połączenia z internetem,
- pionowy feed pytań oparty na `VerticalPager`,
- gest poziomy przełączający kartę pytania i kartę źródła prawnego,
- zapis odpowiedzi i indywidualnego postępu nauki,
- aktywowanie i wyłączanie materiałów bez ponownego pobierania,
- automatyczna aktualizacja aktywnych pakietów,
- jasny i ciemny motyw Material 3,
- podstawowe wsparcie dostępności, w tym akcje TalkBack.

## Jak działa aplikacja

```mermaid
flowchart LR
    subgraph PUB["Przygotowanie treści"]
        P["Parser aktów prawnych"] --> E["Lokalny edytor pytań"]
        E --> G["Walidacja i generator manifestu"]
        G --> Z["Pakiety .json.gz<br/>+ manifest.json"]
    end

    subgraph FB["Firebase"]
        A["Authentication"]
        F["Firestore<br/>konto i accountTier"]
        S["Storage<br/>manifest i pakiety"]
    end

    subgraph APP["Aplikacja Android"]
        M["Panel Materiały"]
        V["SHA-256, gzip<br/>i walidacja schematu"]
        R["Room"]
        N["Nauka<br/>rolki pytań"]
        PR["Postęp użytkownika"]
    end

    Z --> S
    A --> APP
    F --> M
    S --> M
    M --> V
    V --> R
    R --> N
    N --> PR
    PR --> R
```

1. Po zalogowaniu aplikacja pobiera mały `manifest.json` opisujący dostępne
   dziedziny, wersje i lokalizacje pakietów.
2. Użytkownik wybiera materiały, które chce zachować na urządzeniu.
3. Aplikacja pobiera skompresowany pakiet do pliku tymczasowego i sprawdza jego
   sumę SHA-256.
4. Rozpakowanie, walidacja i import do Room odbywają się na `Dispatchers.IO`,
   dzięki czemu interfejs pozostaje responsywny.
5. Ekran Nauka losuje pytania wyłącznie z aktywnych pakietów zapisanych lokalnie.

## Interfejs

<!--
Zastąp poniższe pliki własnymi zrzutami ekranu. Najlepiej użyć obrazów o tej
samej wysokości i przyciąć je do samego ekranu telefonu.
-->

| Nauka | Materiały | Źródło pytania |
| :---: | :---: | :---: |
| ![Ekran nauki](media/learning.png) | ![Panel materiałów](media/materials.png) | ![Treść przepisu](media/source.png) |

### Nauka w formie rolek

Każde pytanie zajmuje jedną pionową stronę. Po udzieleniu odpowiedzi aplikacja
natychmiast wskazuje wynik. Przesunięcie karty w prawo odsłania pełną treść
przepisu, z którego pochodzi pytanie, a przesunięcie w lewo wraca do testu.
Pozwala to sprawdzić podstawę prawną bez opuszczania bieżącej rolki.

### Materiały dostępne offline

Panel Materiały pokazuje pakiety opublikowane w zdalnym manifeście oraz ich
aktualny stan: niepobrany, pobierany, instalowany, aktywny, nieaktywny albo
wymagający aktualizacji. Wyłączenie materiału zwalnia miejsce w limicie konta,
ale zachowuje dane na urządzeniu.

## Architektura

Aplikacja wykorzystuje podejście offline-first. Firebase nie służy do
pobierania każdego pytania osobno — backend dystrybuuje wersjonowane paczki,
a właściwa nauka działa na lokalnych danych.

| Obszar | Rozwiązanie |
| --- | --- |
| Interfejs | Jetpack Compose, Material 3, Navigation Compose |
| Stan UI | ViewModel, StateFlow |
| Operacje asynchroniczne | Kotlin Coroutines, `Dispatchers.IO` |
| Lokalna baza | Room |
| Uwierzytelnianie | Firebase Authentication, Google Sign-In |
| Dane konta | Cloud Firestore |
| Dystrybucja treści | Firebase Storage, JSON, gzip, SHA-256 |
| Budowanie | Gradle Kotlin DSL |
| Testy | JUnit, testy parserów i polityk, Compose UI tests |

### Przepływ stanu materiału

```mermaid
stateDiagram-v2
    [*] --> Niepobrany
    Niepobrany --> Pobieranie: wybór pakietu
    Pobieranie --> Weryfikacja: pobrano plik
    Weryfikacja --> Import: SHA-256 i schemat poprawne
    Weryfikacja --> Błąd: plik niepoprawny
    Import --> Aktywny: transakcja zakończona
    Import --> Błąd: import nieudany
    Błąd --> Pobieranie: ponów
    Aktywny --> Nieaktywny: wyłącz
    Nieaktywny --> Aktywny: aktualna wersja
    Nieaktywny --> Pobieranie: dostępna aktualizacja
```

## Lokalny model danych

Room przechowuje zarówno wersjonowaną treść prawną, jak i lokalny postęp
użytkownika. Pakiet jest importowany jako jeden dokument, dlatego aktualizacja
nie usuwa pozostałych dziedzin ani historii odpowiedzi.

```mermaid
erDiagram
    DOCUMENTS ||--o{ DOCUMENT_NODES : contains
    DOCUMENT_NODES ||--o{ DOCUMENT_NODES : parent_of
    DOCUMENT_NODES ||--o{ QUESTIONS : provides_source_for
    QUESTIONS ||--|{ ANSWERS : has
    QUESTIONS ||--o| QUESTION_PROGRESS : tracks
    QUESTIONS ||--o{ ANSWER_ATTEMPTS : records
    DOCUMENT_NODES ||--o{ ANSWER_ATTEMPTS : contextualizes
    QUESTIONS ||--o{ REPORTS : concerns
    DOCUMENT_NODES ||--o{ REPORTS : points_to

    DOCUMENTS {
        string id PK
        string title
        string lawCategoryId
        string lawCategoryName
        string contentVersion
        string packageId
        datetime importedAt
        boolean isActive
    }

    DOCUMENT_NODES {
        string id PK
        string documentId FK
        string parentId FK
        string nodeType
        int position
        string reference
        string title
        string text
        string hierarchyPathJson
        int probabilityScore
    }

    QUESTIONS {
        string id PK
        string documentNodeId FK
        string content
        int probabilityScore
        datetime createdAt
        datetime updatedAt
    }

    ANSWERS {
        string id PK
        string questionId FK
        int position
        string content
        boolean isCorrect
    }

    QUESTION_PROGRESS {
        string questionId PK
        int seenCount
        int correctAnswersCount
        int wrongAnswersCount
        int currentStreak
        int lapseCount
        datetime lastSeenAt
        datetime nextReviewAt
        int intervalDays
        double difficulty
        int masteryScore
        boolean isMastered
        boolean dirty
    }

    ANSWER_ATTEMPTS {
        string attemptId PK
        string questionId FK
        string documentNodeId FK
        datetime answeredAt
        int selectedAnswerIndex
        boolean wasCorrect
        long responseTimeMs
        string sessionType
    }

    REPORTS {
        string id PK
        string questionId FK
        string documentNodeId FK
        datetime createdAt
        string message
        boolean dirty
    }
```

## Najciekawsze decyzje techniczne

### Paczki zamiast wielu odczytów

Treść pytań nie jest pobierana z Firestore rekord po rekordzie. Mały manifest
opisuje dostępne materiały, a Firebase Storage dostarcza skompresowane pakiety.
Ogranicza to liczbę operacji sieciowych, ułatwia kontrolę kosztów i pozwala
korzystać z aplikacji offline.

### Bezpieczna aktualizacja treści

Nowa wersja trafia najpierw do pliku tymczasowego. Dopiero po sprawdzeniu
rozmiaru, SHA-256, gzip i schematu jest zapisywana transakcyjnie. Poprzednia
wersja pozostaje użyteczna, jeśli pobranie lub walidacja się nie powiedzie.

### Stabilne identyfikatory

Dokumenty, przepisy, pytania i odpowiedzi używają stabilnych identyfikatorów
tekstowych. Aktualizacja pakietu może dzięki temu zachować postęp przypisany do
pytań, które nie zmieniły swojej tożsamości.

### Płynny interfejs

Transfer, dekompresja, parsowanie i import wykonywane są poza głównym wątkiem.
Stan operacji jest obserwowany przez Compose, dlatego użytkownik może zmienić
zakładkę, gdy materiały nadal są przygotowywane.

## Jakość

Projekt zawiera testy obejmujące m.in.:

- parser i walidację manifestu,
- parser pakietów treści,
- limit aktywnych materiałów dla typów kont,
- harmonogram powtórek,
- nawigację głównych zakładek,
- gest przełączania pytania i źródła,
- stany oraz odświeżanie panelu Materiały,
- generator manifestu publikacyjnego.

Przed zatwierdzaniem zmian uruchamiane są testy jednostkowe, kompilacja aplikacji
i testów instrumentacyjnych oraz Android Lint.

## Status projektu

APPlikacja jest rozwijanym MVP. Aktualny przepływ obejmuje:

- rejestrację i logowanie,
- pobieranie oraz zarządzanie materiałami,
- lokalny import i naukę offline,
- rolki pytań wraz ze źródłami,
- zapisywanie postępu na urządzeniu.

W kolejnych etapach planowane są synchronizacja postępu między urządzeniami,
pełny tryb egzaminacyjny, statystyki oraz bezpieczny proces obsługi kont Premium.

## O projekcie

APPlikację zaprojektowałem i rozwijam samodzielnie — od modelu produktu i
interfejsu, przez aplikację Android oraz lokalną bazę, po pipeline przygotowania
i dystrybucji treści.

Repozytorium jest prezentacją projektu i nie zawiera jego kodu źródłowego.
Chętnie opowiem o implementacji, kompromisach architektonicznych i dalszym
kierunku rozwoju podczas rozmowy technicznej.

<!--
Opcjonalnie dodaj:

## Kontakt

- LinkedIn: https://www.linkedin.com/in/TWOJ-PROFIL
- GitHub: https://github.com/TWOJ-LOGIN
- E-mail: twoj@email.pl
-->

---

> APPlikacja jest narzędziem edukacyjnym i nie stanowi źródła porad prawnych.

# Afisz — strona z listą wydarzeń

Statyczna strona hostowana na GitHub Pages, z panelem administratora, który
publikuje zmiany bezpośrednio do repozytorium przez GitHub API.

## Pliki
- `index.html` — strona powitalna z listą wydarzeń (nadchodzące / minione)
- `admin.html` — panel administratora: edycja i publikacja wydarzeń na GitHub
- `events.json` — dane wydarzeń, czytane przez `index.html`
- `style.css` — wspólny, minimalistyczny styl

## 1. Wystawienie na GitHub Pages
1. Utwórz repozytorium na GitHub (np. `afisz`) — może być publiczne lub prywatne (GitHub Pages dla prywatnych repo wymaga płatnego planu; dla publicznych jest darmowy).
2. Wrzuć do niego wszystkie pliki z tego folderu.
3. **Settings → Pages**, branch `main`, folder `/root`, zapisz.
4. Strona będzie dostępna pod `https://<twoja-nazwa-uzytkownika>.github.io/afisz/`.

## 2. Token do panelu administratora
GitHub REST API jest darmowe (limit 5000 żądań/godzinę dla uwierzytelnionych
zapytań — w tym przypadku nieosiągalny). Żeby panel administratora mógł
publikować zmiany, potrzebny jest token dostępu:

1. Wejdź na `github.com/settings/personal-access-tokens/new`.
2. Wybierz **fine-grained personal access token**.
3. Ogranicz go do **jednego repozytorium** (tego z Twoją stroną).
4. Nadaj uprawnienie **Repository permissions → Contents: Read and write**.
5. Ustaw krótki termin ważności (np. 90 dni) i wygeneruj token.
6. Skopiuj go — będzie potrzebny tylko raz, przy logowaniu w `admin.html`.

Token nie jest nigdzie zapisywany poza Twoją przeglądarką: trafia wyłącznie
w nagłówku żądań do `api.github.com`. Opcjonalne zaznaczenie „zapamiętaj na
czas karty" trzyma go w `sessionStorage` (znika po zamknięciu karty).

## 3. Jak działa edycja
1. Otwórz `admin.html`, podaj `właściciel/repo`, gałąź i token.
2. Dodawaj / edytuj / usuwaj wydarzenia — zmiany są na razie robocze (tylko w tej karcie).
3. Kliknij **„Publikuj na GitHub"** — panel wykonuje commit pliku `events.json`
   bezpośrednio do repozytorium przez GitHub API.
4. Strona główna (`index.html`) pokaże nową wersję od razu po odświeżeniu
   (ewentualne opóźnienie cache przeglądarki/CDN GitHub Pages to zwykle kilka–kilkadziesiąt sekund).

Przyciski „Pobierz kopię JSON" i „Wczytaj plik" zostały jako zapasowa opcja
ręczna, gdyby publikacja przez API była z jakiegoś powodu niedostępna.

## 4. Rezerwacja miejsc (Firebase)

Rezerwacja miejsc jest współdzielona dla wszystkich odwiedzających (żeby dwie
osoby nie mogły zająć tego samego miejsca), więc wymaga darmowej bazy danych
**Firebase Firestore**:

1. Wejdź na `console.firebase.google.com` i utwórz nowy projekt (bez karty płatniczej — wystarczy plan darmowy „Spark”).
2. W menu po lewej: **Build → Firestore Database → Create database** (tryb produkcyjny, dowolny region).
3. **Build → Authentication → Sign-in method → Anonymous → Enable** (strona loguje każdego odwiedzającego anonimowo, żeby reguły mogły rozróżnić „zalogowanego" gościa od bota).
4. **Project settings (ikona zębatki) → Your apps → Web (`</>`)** — zarejestruj aplikację i skopiuj obiekt `firebaseConfig`.
5. Wklej te wartości do pliku `firebase-config.js` w repozytorium (podmień plik przez „Edit" na GitHub, tak jak inne pliki).
6. W **Firestore Database → Rules** wklej poniższe reguły i kliknij „Publish":

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /events/{eventId}/seats/{seatId} {
         allow read: if true;
         allow create: if request.auth != null
           && !exists(/databases/$(database)/documents/events/$(eventId)/seats/$(seatId));
         allow update, delete: if false;
       }
     }
   }
   ```

   Te reguły pozwalają każdemu czytać zajęte miejsca, ale zapisać (zarezerwować)
   może tylko zalogowany (choćby anonimowo) odwiedzający, i tylko jeśli miejsce
   jest jeszcze wolne — więc dwie osoby nie zajmą tego samego miejsca.

7. W panelu `admin.html`, przy edycji wydarzenia, zaznacz „włącz rezerwację miejsc" i uzupełnij rzędy, liczbę miejsc w rzędzie, rzędy VIP oraz ceny.
8. Na stronie głównej przy wydarzeniu pojawi się przycisk „Zarezerwuj miejsca", prowadzący do `rezerwacja.html?event=<id>`.

Odwołanie rezerwacji (np. gdy ktoś zrezygnuje) nie ma na razie przycisku w
panelu — reguły celowo blokują `update`/`delete`, żeby nikt z zewnątrz nie mógł
zwolnić cudzego miejsca. Zwolnienie miejsca można zrobić ręcznie, usuwając
odpowiedni dokument w **Firestore Database → Data** w konsoli Firebase.

Płatności w tej wersji nie są podłączone — formularz rezerwacji zbiera tylko
imię i e-mail. Realne pobieranie opłat (np. przez Stripe Payment Links) można
dodać w kolejnym kroku bez zmiany reszty architektury — daj znać, jeśli chcesz, żebym to przygotował.

## Uwagi bezpieczeństwa
- Token ogranicz do jednego repozytorium i tylko do uprawnienia Contents — tak, żeby w razie wycieku szkoda była jak najmniejsza.
- Nie udostępniaj linku do `admin.html` publicznie — każdy, kto go otworzy i wklei swój token z dostępem do repo, będzie mógł publikować zmiany.
- Jeśli dwie osoby edytują równocześnie, druga publikacja może się nie powieść (konflikt wersji pliku) — panel pokaże błąd; wtedy należy się rozłączyć i połączyć ponownie, by pobrać świeżą wersję.

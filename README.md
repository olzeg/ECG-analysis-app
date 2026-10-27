# Projekt diagnostyka · analiza EKG

Wspólne repo zespołu: strona → backend → analizy EKG → zapis i wyniki.
Na razie sam szkielet, bez aplikacji. Moduły: **ECG_PREPROCESSING, R_PEAKS,
HRV1, HRV2**.

> **Hasła, klucze API, tokeny i dane dostępowe do Azure nigdy nie trafiają
> do kodu, notebooków ani commitów. Wpisuj je tylko do lokalnej konfiguracji.**

## Co gdzie jest?

| Folder | Co tu robimy |
| --- | --- |
| `backend/` | API i organizacja analiz; kod w `src/`, testy w `tests/` |
| `frontend/` | Strona: HTML, CSS, JavaScript i wykresy |
| `modules/` | Cztery moduły EKG; każdy ma własne `src/` i `tests/` |
| `shared/` | Wspólny odczyt MIT-BIH (`data_io/`) i formaty komunikacji (`contracts/`) |
| `database/` | Migracje i skrypty bazy, nie pliki danych |
| `infra/`, `docker/`, `.github/` | Azure, kontenery i automatyzacja — Cloud/DevOps |
| `tests/` | Testy współpracy komponentów i całej aplikacji |
| `notebooks/` | Eksperymenty; bez sekretów i poufnych wyników |
| `data/`, `outputs/` | Lokalne dane i wyniki — Git je pomija |

Puste foldery mają `.gitkeep`, żeby Git je zapamiętał. Każdy pisze kod i testy
swojej części; zmiany wspólnych formatów uzgadniamy z osobami, które ich używają.

## Start: zainstaluj i przygotuj środowisko

W Windows otwórz **PowerShell** i zainstaluj narzędzia, jeśli ich nie masz:

```powershell
winget install --id Git.Git -e
winget install --id astral-sh.uv -e
```

Otwórz ponownie terminal. W VS Code otwórz folder repo i wybierz
**Terminal → Nowy terminal**. Wszystkie dalsze polecenia wykonuj w głównym folderze repo.

```powershell
git --version
uv --version
uv sync --locked
```

`uv` przygotuje Pythona 3.12 i osobne środowisko `.venv` (może potrzebować internetu).
Nie trzeba aktywować środowiska ręcznie.

## uv: biblioteki i uruchamianie

**Po co uv?** Żeby każdy mógł zainstalować zgodny zestaw bibliotek i uruchomić
kod bez ręcznego ustalania, co kto ma na komputerze. uv zarządza Pythonem,
bibliotekami i osobnym środowiskiem projektu — nie miesza ich z innymi projektami.

| Plik / folder | Za co odpowiada |
| --- | --- |
| `pyproject.toml` | Opis projektu, wymagana wersja Pythona i lista potrzebnych bibliotek |
| `uv.lock` | Dokładne wersje bibliotek i ich zależności, dobrane przez uv |
| `.venv/` | Faktycznie zainstalowane środowisko na Twoim komputerze |

Gdy wpiszesz **`uv add numpy`**, uv zapisze wymaganie w `pyproject.toml`,
zaktualizuje `uv.lock` i zainstaluje bibliotekę w `.venv`. Nie musisz ręcznie
dopisywać jej do listy. **`pyproject.toml` i `uv.lock` commitujemy razem**,
a `.venv` zostaje lokalnie. Na start lista bibliotek jest pusta.

```powershell
uv add numpy                 # dodaj potrzebną bibliotekę
uv add --dev pytest          # dodaj narzędzie do pracy/testów
uv remove numpy              # usuń, jeśli niepotrzebna
uv sync --locked             # odtwórz środowisko, np. po pobraniu zmian
uv run python backend/src/nazwa.py  # uruchom własny, istniejący plik
```

To przykłady: nazwę biblioteki i skryptu dopasuj do zadania.
`uv run` uruchamia kod w środowisku projektu, więc widzi jego biblioteki.

**Ktoś dodał bibliotekę?** Po pobraniu jego zmian wykonaj `uv sync --locked`.
uv odtworzy zależności z locka, bez samodzielnego wybierania nowszych wersji.
Jeśli zgłosi niezgodność locka z projektem, sprawdź, czy pobrano oba pliki.

Używaj `uv add` zamiast `pip install`; nie edytuj ręcznie `uv.lock` i nie
przesyłaj innym `.venv`. Jeden główny projekt uv obsługuje na razie cały kod
Python. [Więcej o uv](https://docs.astral.sh/uv/guides/projects/).

## Konfiguracja: dlaczego dwa pliki?

- **`config.example.toml`** — pusty wzór dla wszystkich, zapisany w Git.
  Pokazuje nazwy ustawień. **Nigdy nie wpisuj tu prawdziwych sekretów.**
- **`config.local.toml`** — Twoja prywatna kopia z ustawieniami i danymi dostępowymi.
  Git ją ignoruje. Edytuj tylko tę kopię.

Jeśli nie masz pliku lokalnego, utwórz go **jeden raz**:

```powershell
Copy-Item config.example.toml config.local.toml
```

Nie kopiuj ponownie na istniejący plik — nadpiszesz własne ustawienia.
W TOML sekcje są w `[nawiasach]`, a wartości tekstowe w `"cudzysłowie"`.
Nowe ustawienie dopisz też do wzoru, ale z pustą lub fikcyjną wartością.

### Jak użyć ustawień w Pythonie?

Sam plik nie wczytuje się automatycznie. Ten fragment w swoim skrypcie odczytuje
go modułem `tomllib` (wbudowany w Python 3.12, bez instalowania bibliotek):

```python
from pathlib import Path
import tomllib

# Uruchamiaj skrypt z głównego folderu repo, jak w poleceniu uv powyżej.
with Path("config.local.toml").open("rb") as file:
    config = tomllib.load(file)

environment = config["app"]["environment"]
tenant_id = config["azure"]["tenant_id"]
client_id = config["azure"]["client_id"]
client_secret = config["azure"]["client_secret"]
# Używaj tych zmiennych jako argumentów klienta Azure, zamiast wpisywać hasło w kod.
```

To odczyt ustawień, nie logowanie do Azure. **Nie drukuj `config` ani sekretów.**
Plik lokalny to zwykły tekst, nie sejf; w Azure użyjemy uzgodnionego magazynu
sekretów/tożsamości usług. `.env` też jest ignorowany, ale projekt go nie wczytuje.

## Git: zapisuj małe zmiany, sprawdzaj przed wysłaniem

Git zapisuje historię. **Commit** to zapis lokalny, **push** wysyła go na serwer,
a **branch** to gałąź pracy nad zadaniem. Repo jest na razie lokalne, bez serwera.

Ustaw raz własne dane autora (to nie login ani hasło):

```powershell
git config user.name "Imię Nazwisko"
git config user.email "twoj@email.pl"
```

Pierwszy commit struktury wykonuje opiekun: `git add .`, potem
`git diff --cached` (sprawdź każdy plik i brak sekretów), a następnie
`git commit -m "Przygotuj strukture repo"`.

Po pierwszym commicie pracuj tak:

```powershell
git switch -c zadanie/krotki-opis   # nowa gałąź; na istniejącą wróć bez -c
# Edytuj pliki, sprawdź działanie i uruchom testy swojej części.
git status                        # co się zmieniło?
git diff                          # przejrzyj zmiany; nowe pliki otwórz w edytorze
git add backend/src               # dodaj tylko pliki/katalog swojej zmiany
git diff --cached                 # TO trafi do commita: sprawdź też sekrety!
git commit -m "Opisz konkretna zmiane"
```

Gdy pojawi się wspólne repo, pobierz je przez `git clone ADRES_REPO`.
Przed nowym zadaniem, z zapisanymi zmianami: `git switch main`, potem
`git pull --ff-only`. Po commicie: `git push -u origin NAZWA_GALEZI`
i Pull Request do sprawdzenia przez drugą osobę. Przy konflikcie poproś o pomoc;
nie używaj w ciemno `reset --hard` ani `push --force`.

> **Przed każdym commitem sprawdź, czy nie dodajesz haseł lub tokenów.**
> `.gitignore` pomija lokalny config, `.env`, klucze, dane i `.venv`, ale nie
> wykrywa sekretów wpisanych w kod i nie usuwa wcześniejszych commitów.
> Jeśli sekret trafił do historii: unieważnij go i zgłoś opiekunowi.
> Samo usunięcie linijki nie wystarczy. Nie wymuszaj dodania plików przez `git add -f`.

## Struktura commitów
Dodanie czegoś/zrobienie cześćpracy:
`git commit -m "feat: add/build/etc. *your_comment*"`
Naprawienie buga:
`git commit -m "bug-fix: change/romove/etc. *your_comment*"`


## Info od waszego DecOpsa
Ustawiłem w yamlu wersje Postgres:15-alpine -> Jeśli wymagana jest zmiana proszę o kontakt
Zakładam że system będzie oparty o Django i przetwarzane będą surowe pliki sygnałowe (.DTA, CSV).

Moje podjeście do Kontenerycji tego projektu to Compose + współdzielone wolumny danych. tnz. Każdy moduł (i nie tylko backend, frontend, database) ma swój kontener którym jest cześcią sieci, wolumen jest współdzielony miedzy modułami a backendem który zarządza procesami.

Model komunikacyji między backendem a modułami to podejście asynchroniczne Celery + Redis ze współdzielonymi wolumenami danych. Proszę przed rozpoczęciem pracy zapoznać się z tym podejściem.
https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/redis.html


Tak samo założyłem, że backend i moduły będą pisane w pythonie. Więcej info o szczegółach konteneryzacji plików python w `Dockerfile.python`.

Używajcie pytest pls.
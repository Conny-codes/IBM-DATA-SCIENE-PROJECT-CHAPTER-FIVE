# Merkzettel: "Extracting and Visualizing Stock Data"

IBM-Abschlussprojekt (Skills Network). Ziel: Aktienkurse und Quartalsumsätze von **Tesla (TSLA)** und **GameStop (GME)** beschaffen, säubern und in einem zweiteiligen Dashboard darstellen.

Der eigentliche Lerninhalt sind **zwei verschiedene Wege, an Daten zu kommen**:

| Weg | Womit | Wofür im Projekt |
|---|---|---|
| Fertige API abfragen | `yfinance` | Kursdaten (täglich, seit Börsengang) |
| Webseite scrapen | `requests` + `BeautifulSoup` / `pandas.read_html` | Quartalsumsätze aus einer HTML-Tabelle |

---

## 1. Werkzeugkasten

```python
import yfinance as yf          # Finanzdaten-API (Wrapper um Yahoo Finance)
import pandas as pd            # DataFrames = Tabellen im Speicher
import requests                # HTTP-Request, holt die rohe Webseite
from bs4 import BeautifulSoup  # HTML-Parser
import matplotlib.pyplot as plt # statische Plots
from io import StringIO        # String → dateiartiges Objekt
```

`warnings.filterwarnings("ignore", category=FutureWarning)` unterdrückt nur Konsolen-Rauschen von veralteten pandas-Aufrufen. Hat keinen Einfluss auf das Ergebnis.

`StringIO` ist nötig, weil `pd.read_html()` seit pandas 2.1 keinen rohen HTML-**String** mehr akzeptiert, sondern ein Datei-Objekt oder eine URL. `StringIO(html_data)` verpackt den String so, dass pandas ihn wie eine Datei lesen kann.

---

## 2. Die vorgegebene Funktion `make_graph`

Du musstest sie nicht schreiben, nur korrekt aufrufen. Was sie tut:

1. Schneidet beide DataFrames zeitlich ab (Kurs bis 2021-06-14, Umsatz bis 2021-04-30) — deshalb enden die Grafiken 2021, egal wie aktuell deine Daten sind.
2. Legt zwei untereinanderliegende Subplots an (`plt.subplots(2, 1, sharex=True)`).
3. Oben Aktienkurs (`Close`), unten Umsatz (`Revenue`).
4. `astype("float")` — sie erwartet, dass die Spalten **numerisch konvertierbar** sind. Genau deshalb muss vorher `$` und `,` aus der Revenue-Spalte raus.

**Erwarteter Input:** Kurs-DataFrame mit Spalten `Date` + `Close`, Umsatz-DataFrame mit `Date` + `Revenue`, plus Name als String.

---

## 3. Aufgabe für Aufgabe

### Q1 — Tesla-Kursdaten über die API

```python
tesla = yf.Ticker("TSLA")
tesla_data = tesla.history(period="max")
tesla_data.reset_index(inplace=True)
tesla_data.head()
```

- `yf.Ticker("TSLA")` erzeugt ein **Ticker-Objekt** — noch keine Daten, nur ein Zugriffs-Handle auf das Wertpapier.
- `.history(period="max")` löst den eigentlichen Abruf aus und gibt einen DataFrame zurück: `Open, High, Low, Close, Volume, Dividends, Stock Splits`, ein Zeile pro Handelstag (Tesla ab 2010-06-29).
- **`reset_index(inplace=True)` ist der wichtige Schritt:** Das Datum steckt zunächst im *Index* der Tabelle, nicht in einer Spalte. `reset_index` schiebt es in eine echte Spalte `Date`. Ohne das schlägt später `stock_data.Date` in `make_graph` fehl.
- `inplace=True` = ändert das Objekt direkt, statt eine Kopie zurückzugeben.

### Q2 — Tesla-Umsatz per Webscraping

```python
url = "https://cf-courses-data.s3...../revenue.htm"
html_data = requests.get(url).text
soup = BeautifulSoup(html_data, "html.parser")

tesla_revenue = pd.read_html(StringIO(html_data))[1]
tesla_revenue.columns = ["Date", "Revenue"]
```

- `requests.get(url).text` lädt die Seite und gibt den **HTML-Quelltext als String** zurück.
- `BeautifulSoup(...)` parst das HTML in einen navigierbaren Baum. → **In deiner Lösung wird `soup` angelegt, aber nie benutzt.** Du hast den Kurzweg über `read_html` genommen; das ist legitim, die Zeile ist nur Ballast (bzw. Pflichtschritt der Aufgabenstellung).
- `pd.read_html(...)` findet *alle* `<table>`-Elemente der Seite und gibt eine **Liste von DataFrames** zurück. `[1]` = die zweite Tabelle = **Quartalsumsätze** (`[0]` wären die Jahresumsätze).
- `.columns = [...]` überschreibt die Spaltennamen mit den geforderten `Date` / `Revenue`.

**Datenreinigung:**

```python
tesla_revenue["Revenue"] = tesla_revenue["Revenue"].str.replace(',|\$', "", regex=True)
tesla_revenue.dropna(inplace=True)
tesla_revenue = tesla_revenue[tesla_revenue["Revenue"] != ""]
tesla_revenue.tail()
```

- `.str.replace(..., regex=True)` mit dem Muster `,|\$`: Das `|` heißt "oder", `\$` ist das maskierte Dollarzeichen (in Regex bedeutet `$` sonst "Zeilenende"). Ergebnis: aus `$21,462` wird `21462`.
- `.dropna()` entfernt Zeilen mit fehlenden Werten (`NaN`).
- Die letzte Zeile filtert **leere Strings** — die sind kein `NaN` und überleben `dropna()`, würden aber bei `astype("float")` crashen.
- **Achtung Reihenfolge:** Der Wert bleibt danach ein *String*, nur ohne Sonderzeichen. Die Umwandlung in Zahlen passiert erst in `make_graph`.

### Q3 — GameStop-Kursdaten

Identisch zu Q1, nur mit `GME`:

```python
gme = yf.Ticker("GME")
gme_data = gme.history(period="max")
gme_data.reset_index(inplace=True)
gme_data.head()
```

Daten ab 2002-02-13.

### Q4 — GameStop-Umsatz per Webscraping

```python
url2 = "https://cf-courses-data.s3...../stock.html"
html_data_2 = requests.get(url2).text
gme_revenue = pd.read_html(StringIO(html_data_2))[1]
gme_revenue.columns = ["Date", "Revenue"]

gme_revenue["Revenue"] = gme_revenue["Revenue"].str.replace(r',|\$', "", regex=True)
gme_revenue.dropna(inplace=True)
gme_revenue = gme_revenue[gme_revenue["Revenue"] != ""]
```

Gleiches Muster wie Q2. Unterschied im Detail: hier steht ein `r` vor dem Regex-String (`r',|\$'`) — ein **Raw String**, bei dem Python den Backslash nicht selbst interpretiert. Das ist die saubere Schreibweise; in Q2 fehlt sie, funktioniert dort aber noch (mit DeprecationWarning).

### Q5 / Q6 — Dashboard zeichnen

```python
make_graph(tesla_data, tesla_revenue, 'Tesla')
make_graph(gme_data, gme_revenue, 'GameStop')
```

Nur noch Funktionsaufruf. Die gesamte Arbeit steckte in Beschaffung + Reinigung — das ist die eigentliche Lektion des Projekts.

---

## 4. Wiederkehrende Muster (das Übertragbare)

| Muster | Code | Wozu |
|---|---|---|
| API-Handle erzeugen | `yf.Ticker("XYZ")` | Objekt, noch kein Abruf |
| Daten ziehen | `.history(period="max")` | DataFrame |
| Index → Spalte | `reset_index(inplace=True)` | Datum benutzbar machen |
| Seite laden | `requests.get(url).text` | HTML als String |
| Tabellen extrahieren | `pd.read_html(StringIO(html))[n]` | Liste von DataFrames |
| Zeichen entfernen | `.str.replace(r'a|b', "", regex=True)` | Sonderzeichen aus Strings |
| Leerwerte weg | `.dropna()` + `df[df.col != ""]` | zwei verschiedene Fälle! |
| Erste/letzte Zeilen | `.head()` / `.tail()` | Sichtprüfung |

---

## 5. Drei Schwachstellen in deinem eingereichten Notebook

Falls du es nochmal anfasst:

1. **Die Ausgabe von `tesla_revenue.tail()` ist veraltet.** Die Zellen wurden in falscher Reihenfolge ausgeführt (Ausführungsnummern: `dropna`=10, `tail()`=11, `str.replace`=19). Die sichtbare Tabelle zeigt deshalb noch `$31`, `$28` — also den *ungereinigten* Zustand. Für eine screenshot-basierte Bewertung ist das der Fehler, der Punkte kostet.
2. **`gme_revenue.tail()` fehlt komplett.** Die Aufgabenstellung fordert die Anzeige der letzten fünf Zeilen; die Zelle ist im Notebook nicht vorhanden.
3. **Copy-Paste-Fehler in Q4:** `soup = BeautifulSoup(html_data, "html.parser")` — müsste `html_data_2` sein. Folgenlos, weil `soup` nie verwendet wird, aber es zeigt, dass die Zeile ungeprüft übernommen wurde.

**Grundregel für die Abgabe:** vor jedem Screenshot `Kernel → Restart & Run All`. Dann stimmen Ausführungsreihenfolge und sichtbare Ausgaben garantiert überein.

---

## 6. Baukasten zum Wiederverwenden

Drei Bausteine, die unabhängig voneinander funktionieren. Jeder ist so kommentiert, dass du ihn in ein fremdes Projekt kopieren und nur die markierten Stellen anpassen musst.

### Baustein A — Kursdaten für ein beliebiges Wertpapier

```python
import yfinance as yf

# ↓ NUR DAS HIER ANPASSEN: Börsenkürzel als Text
ticker = yf.Ticker("AAPL")          # Apple. Weitere: "MSFT", "SAP.DE", "^GDAXI"
                                    # .DE = Xetra, ^ = Index statt Einzelaktie
                                    # Diese Zeile ruft noch nichts ab, sie legt
                                    # nur den Zugriffspunkt an.

df = ticker.history(period="max")   # ↓ hier ggf. Zeitraum ändern:
                                    # "max" | "10y" | "1y" | "6mo" | "5d"
                                    # Optional: interval="1d" | "1wk" | "1mo"
                                    # Erst DIESE Zeile geht ins Netz.

df.reset_index(inplace=True)        # Datum vom Index in eine echte Spalte "Date".
                                    # Ohne das kannst du df.Date nicht ansprechen
                                    # und kein Diagramm daraus bauen.

print(df.columns.tolist())          # Sichtkontrolle: welche Spalten gibt es?
print(df.shape)                     # (Zeilenanzahl, Spaltenanzahl)
```

**Was du zusätzlich aus dem Ticker-Objekt ziehen kannst** (dieselbe Logik, andere Methode):

```python
ticker.dividends      # Dividendenhistorie
ticker.splits         # Aktiensplits
ticker.info           # Stammdaten als Dictionary (Branche, Land, Kennzahlen)
ticker.quarterly_financials   # Bilanzdaten - macht das Scrapen oft überflüssig
```

**Wann Baustein A nicht reicht:** Yahoo liefert keine garantierte Verfügbarkeit und keine Fundamentaldaten in Tiefe. Für ernsthafte Auswertungen sind `alpha_vantage`, `financialmodelingprep` oder direkt die Börsen-APIs die stabilere Wahl.

---

### Baustein B — HTML-Tabelle von einer Webseite holen

Der schnelle Weg. Funktioniert immer dann, wenn die Daten wirklich in einem `<table>`-Element stehen.

```python
import requests, pandas as pd
from io import StringIO

url = "https://beispiel.de/seite.html"      # ← anpassen

html = requests.get(url).text               # lädt die Seite, .text = Quelltext
                                            # als String (das, was der Browser
                                            # unter "Seitenquelltext" zeigt)

tabellen = pd.read_html(StringIO(html))     # findet ALLE <table> auf der Seite
                                            # und gibt eine LISTE von DataFrames
                                            # zurück. StringIO ist nur Verpackung:
                                            # pandas will ein dateiartiges Objekt,
                                            # keinen nackten String.

print(len(tabellen))                        # ← ERST ZÄHLEN, DANN GREIFEN.
for i, t in enumerate(tabellen):            # zeigt Index + erste Zeilen jeder
    print(i, t.shape)                       # Tabelle, damit du weißt, welche du
    print(t.head(2), "\n")                  # brauchst

df = tabellen[1]                            # ← Index anpassen. Python zählt ab 0.
                                            # Im Kursprojekt war [1] = Quartals-,
                                            # [0] = Jahresumsätze.
df.columns = ["Date", "Revenue"]            # ← Spalten umbenennen. Anzahl muss
                                            # zur tatsächlichen Spaltenzahl passen.
```

**Warum der `print(len(...))`-Schritt wichtig ist:** Im Kursprojekt war die `[1]` vorgegeben. Auf einer fremden Seite ist sie geraten. Ändert die Seite ihr Layout, holst du stillschweigend die falsche Tabelle — der Code läuft trotzdem durch, die Zahlen sind aber Müll. Das ist die häufigste unentdeckte Fehlerquelle beim Scraping.

**Wenn `read_html` nichts findet:** Dann sind die Daten kein `<table>`, sondern per JavaScript nachgeladene `<div>`s. Dann brauchst du BeautifulSoup zum manuellen Navigieren oder `selenium`/`playwright`, um die Seite erst rendern zu lassen.

---

### Baustein B2 — dieselbe Aufgabe mit BeautifulSoup (der Weg, den du übersprungen hast)

Brauchst du, sobald die Struktur nicht als saubere Tabelle vorliegt — und in Prüfungen, die explizit danach fragen.

```python
from bs4 import BeautifulSoup
import pandas as pd

soup = BeautifulSoup(html, "html.parser")   # zerlegt den HTML-String in einen
                                            # durchsuchbaren Baum

tabelle = soup.find_all("tbody")[1]         # find_all gibt eine Liste ALLER
                                            # passenden Elemente. [1] = zweiter
                                            # Tabellenkörper.
                                            # find() (ohne _all) gäbe nur den ersten.

zeilen = []
for tr in tabelle.find_all("tr"):           # tr = table row, eine Tabellenzeile
    zellen = tr.find_all("td")              # td = table data, eine einzelne Zelle
    if len(zellen) < 2:                     # Sicherheitsnetz: Zeilen ohne genug
        continue                            # Zellen (z.B. Überschriften) überspringen
    datum = zellen[0].text.strip()          # .text  = Inhalt ohne HTML-Tags
    umsatz = zellen[1].text.strip()         # .strip = Leerzeichen/Umbrüche weg
    zeilen.append({"Date": datum, "Revenue": umsatz})

df = pd.DataFrame(zeilen)                   # Liste von Dictionaries → DataFrame.
                                            # Die Schlüssel werden zu Spaltennamen.
```

Das ist mehr Code, aber du kontrollierst jeden Schritt. Baustein B ist der Sonderfall, in dem pandas dir diese Schleife abnimmt.

---

### Baustein C — Textspalte in rechenbare Zahlen verwandeln

Der Teil, den du am häufigsten wiederverwenden wirst. Jede gescrapte Tabelle kommt als Text.

```python
spalte = "Revenue"                          # ← anpassen

# 1. Störzeichen entfernen
df[spalte] = df[spalte].str.replace(r'[,$€%\s]', "", regex=True)
#            └ Spalte  └ als Text └ ersetzen
#   r'...'          = Raw String. Der Backslash bleibt so stehen, wie er dasteht,
#                     statt von Python vorab interpretiert zu werden. Immer nutzen,
#                     wenn ein Regex einen Backslash enthält.
#   [ ... ]         = Zeichenklasse: "eines dieser Zeichen". Kürzer als ,|\$|€
#   \s              = jedes Leerzeichen, auch Tabs und Umbrüche
#   ""              = ersetze durch nichts = löschen
#   regex=True      = das Muster ist ein Suchmuster, kein wörtlicher Text.
#                     Ohne diesen Schalter sucht pandas buchstäblich nach "[,$€%\s]".

# 2. Zwei verschiedene Arten von Nichts entfernen
df = df.dropna(subset=[spalte])             # NaN = fehlender Wert
df = df[df[spalte] != ""]                   # "" = leerer Text. dropna sieht den NICHT.
                                            # Beide Zeilen sind nötig - genau hier
                                            # scheitern die meisten Skripte.

# 3. Jetzt erst umwandeln
df[spalte] = pd.to_numeric(df[spalte], errors="coerce")
#   pd.to_numeric ist robuster als .astype(float):
#   errors="coerce" macht aus unkonvertierbarem Schrott ein NaN, statt das
#   ganze Programm abstürzen zu lassen. Danach kannst du nochmal dropna()
#   aufrufen und siehst an der Zeilenzahl, wie viel Müll drin war.

df = df.dropna(subset=[spalte])             # die neu entstandenen NaN wegräumen

# 4. Datumsspalte ebenfalls konvertieren
df["Date"] = pd.to_datetime(df["Date"], errors="coerce")
#   Ohne das behandelt matplotlib Daten als Text und die Zeitachse wird unbrauchbar
#   (25.12. läge dann zwischen 24. und 26. Januar, weil alphabetisch sortiert wird).

df = df.sort_values("Date")                 # gescrapte Tabellen sind oft absteigend
                                            # sortiert. Für Liniendiagramme muss es
                                            # aufsteigend sein, sonst zeichnet die
                                            # Linie rückwärts.
```

**Prüfen, ob es geklappt hat:**

```python
print(df.dtypes)        # muss float64 bzw. datetime64 zeigen, nicht "object".
                        # "object" heißt bei pandas immer: das ist noch Text.
print(df.isna().sum())  # wie viele Lücken pro Spalte sind übrig?
```

---

### Alles zusammen als wiederverwendbare Funktion

```python
def scrape_tabelle(url, tabellen_index=0, spalten=None, zahl_spalte=None):
    """Holt eine HTML-Tabelle von einer URL und gibt sie gesäubert zurück.

    url            : Adresse der Webseite
    tabellen_index : welche Tabelle der Seite (ab 0 zählen)
    spalten        : Liste neuer Spaltennamen, z.B. ["Date", "Revenue"]
    zahl_spalte    : Name der Spalte, die in Zahlen umgewandelt werden soll
    """
    html = requests.get(url).text
    df = pd.read_html(StringIO(html))[tabellen_index]

    if spalten:                                  # nur umbenennen, wenn gewünscht
        df.columns = spalten

    if zahl_spalte:
        df[zahl_spalte] = df[zahl_spalte].str.replace(r'[,$€%\s]', "", regex=True)
        df = df[df[zahl_spalte] != ""]
        df[zahl_spalte] = pd.to_numeric(df[zahl_spalte], errors="coerce")
        df = df.dropna(subset=[zahl_spalte])

    return df

# Aufruf - der komplette Frage-2-Block auf eine Zeile eingedampft:
tesla_revenue = scrape_tabelle(url, 1, ["Date", "Revenue"], "Revenue")
```

Der Docstring in den dreifachen Anführungszeichen ist keine Kosmetik: `help(scrape_tabelle)` zeigt ihn dir in drei Monaten an, wenn du vergessen hast, was `tabellen_index` bedeutet.

---

### Zwei Dinge, die dieser Baukasten nicht abdeckt

- **Rechtliches.** Scraping ist nicht pauschal erlaubt. Prüfe `robots.txt` und die Nutzungsbedingungen der Seite, bevor du automatisiert abrufst — besonders bei kommerzieller Verwendung.
- **Zuverlässigkeit.** Jeder Scraper bricht, sobald die Zielseite ihr Layout ändert. Wenn eine offizielle API existiert, nimm die — auch wenn sie umständlicher wirkt.

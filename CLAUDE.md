# Kontobuch — Projekt-Doku

Persönliches Finanz-Tracking-Tool für Hendrik. Eine einzige HTML-Datei, kein Server,
keine Cloud — läuft komplett im Browser. Diese Doku fasst alle Architektur- und
Modellentscheidungen zusammen, damit neue Chats nicht wieder bei null anfangen.

## Zweck & Umfang
Drei Konten einlesen und auswerten:
- **Volksbank** (privat, Girokonto)
- **N26** (Geschäftskonto)
- **willbe/LLB** (Tagesgeld, für Rücklagen)

Kategorisierung mit Unterkategorien, 50/30/20-Zielerreichung, Rücklagentöpfe mit
automatischer Ratenberechnung, 12-Monats-Forecast.

## Architektur
- Eine HTML-Datei, vanilla JS, kein Build-Prozess.
- Speicherung: `localStorage`. Fallback auf `window.storage`, falls die Datei
  ausnahmsweise innerhalb einer Claude-Artifact-Umgebung statt lokal geöffnet wird.
- Tabs: Dashboard, Buchungen, Rücklagen, Zuordnen, Forecast, Kategorien.

## Datenmodell (State `S`)
- `S.tx`: Buchungen — `{id, acc, date, merchant, note, amt, cat, sub, manual, biz, ign, tr, pot}`
- `S.cats`: Hauptkategorien mit `subs[]`; jede Unterkategorie hat `goal: need|want|save` + `kw[]` (Stichwörter fürs Auto-Matching)
- `S.pots`: Rücklagentöpfe — Feldbedeutung siehe unten
- `S.fc`: Forecast — `{start, from, base, events[]}`
- `S.me`: `{name, addr}` — **nur lokal, nie im Quelltext hardcoden** (siehe Privacy-Regel)
- `S.learn`: gelernte Kategorie-Zuordnung je Händler (Schlüssel `mkey()`)
- `S.bal`: letzter bekannter Kontostand je Konto, aus Auszügen übernommen
- `S.hidden`: gelöschte Topf-Keys, damit sie beim nächsten Import nicht neu entstehen

## Parser (vier Formate, automatische Erkennung)
1. **Volksbank PDF** — Bu-Tag/Wert-Datum ohne Jahr, Betrag mit S/H-Suffix statt Vorzeichen, mehrzeilige Buchungsblöcke (Verwendungszweck steht oft erst in der Folgezeile).
2. **N26 PDF** — Kopfzeile `Name TT.MM.JJJJ ±Betrag€`, Details darunter.
3. **willbe/LLB PDF** — Datum ohne Jahr, keine S/H-Spalte — Richtung wird aus der Saldoänderung zwischen den Zeilen bestimmt. Enthält rotierten Randtext, der gefiltert werden muss.
4. **willbe CSV-Export** — RFC4180 mit eingebetteten Zeilenumbrüchen in Feldern, Minuszeichen mit Leerzeichen abgesetzt (`"- 160,00"`).

**Bekannte Stolperfallen** (schon mal Bugs verursacht):
- `/i`-Flag an einer Regex mit `{6,}`-Quantor macht den Filter zu aggressiv und verschluckt normale Wörter.
- Format-Erkennung darf sich nicht auf IBANs stützen — die tauchen als Gegenkonto auch in fremden Auszügen auf.
- Eigener Name/Adresse/IBAN **nie** im Code — führte einmal dazu, dass private Daten im (für GitHub gedachten) Quelltext standen. Jetzt ausschließlich über `S.me`, lokal im Browser gepflegt.

## Rücklagentöpfe (Modell nach mehreren Iterationen)
Vier Gruppen: `fix` (feste Jahres-/Quartalsrechnungen), `flex` (variable Rücklagen wie Urlaub/Geschenke), `not` (Notgroschen), `umz` (Umzug).

- **Feste Töpfe**: Eingabe ist nur Abbuchungsbetrag + Fälligkeitsmonate (Mehrfachauswahl). Rate = Betrag × Anzahl Termine ÷ 12, vollautomatisch. Bestand = Rate × Monate seit der letzten Abbuchung — direkt vor der Fälligkeit ist der Topf zu 100 % gefüllt, direkt danach bei 0. Optionales Feld „tatsächlich" für eine von der Soll-Rate abweichende Ist-Überweisung (z. B. wenn ein Dauerauftrag nach einer Beitragserhöhung noch nicht angepasst ist).
- **Flexible Töpfe**: nur Rate + Stichtagsbestand. Entnahmen werden manuell zugeordnet (Tab „Rücklagen" → „Entnahmen zuordnen").
- **Sparziele** (Notgroschen/Umzug): Zielbetrag + Zieldatum. Rate = (Ziel − aktueller Bestand) ÷ Restmonate, läuft automatisch in den Forecast ein.

## Forecast
12 Monate, Startkontostand (Girokonto) + geplante Posten (monatlich/jährlich/einmalig).
Die Summe aller Topf-Raten fließt automatisch als Ausgabe ein. Jahresrechnungen selbst
tauchen **nicht** im Forecast auf — die werden aus dem jeweiligen Topf bezahlt, das
Girokonto merkt davon nichts.

## Teststrategie
- JS aus dem `<script>`-Tag per Regex extrahieren, mit `node --check` auf Syntaxfehler prüfen, bevor irgendwas in den Browser geht.
- Mit Playwright (Chromium) echte Läufe gegen **echte** Kontoauszüge — Werte per `page.evaluate()` aus dem DOM lesen, nicht per Screenshot (die kann ich als Claude nicht zuverlässig selbst auswerten).
- Nie nur gegen synthetische Beispiele testen — die meisten echten Bugs kamen erst mit echten PDFs zum Vorschein (Formaterkennung, Regex-Filter, Rundungsfälle).

## Privacy-Regel
Name, Adresse, IBAN dürfen nie im Quelltext stehen. Der Maßstab: Die Datei muss
bedenkenlos öffentlich hostbar sein.

## Setup & Workflow
- Lokal: `~/Projects/kontobuch`, eigenes Git-Repo, ein Repo pro Tool (nicht mit anderen Projekten vermischen).
- Entwicklung über Claude Code Desktop, nicht mehr claude.ai-Chat — diese Datei wird automatisch als Kontext geladen.
- Hosting: GitHub Pages (Repo public, `index.html` im Root). Kostenlos, ein Push aktualisiert die gehostete Version automatisch.
- Bei jeder Änderung: `node --check` vor dem Ausliefern, bei echten Auszügen mit Playwright/DOM verifizieren (siehe Teststrategie oben), nie nur mit Fantasiedaten testen.

## Offene Punkte (Stand zuletzt)
- Mannschaftskasse: fest oder flexibel? Noch nicht geklärt.
- ASI Servicegebühr: Fälligkeitsmonat unbekannt (~47–48 €, „irgendwann im Frühjahr").
- TBO: Mail an den Verein raus, Antwort steht aus.
- Müll-Topf `ACC_STMT_MTH_DT_LLB…` (Randtext-Artefakt) muss einmalig manuell gelöscht werden — der Regex-Fix verhindert nur die Neuanlage.
- Mobile Nutzung: kein Sync zwischen Geräten, nur JSON-Sicherung als Workaround.

# Kontobuch — Projekt-Doku

Persönliches Finanz-Tracking-Tool für Hendrik. Eine einzige HTML-Datei, kein Server,
keine Cloud — läuft komplett im Browser. Diese Doku fasst alle Architektur- und
Modellentscheidungen zusammen, damit neue Chats nicht wieder bei null anfangen.

**Live:** https://hendrikjansenhsd.github.io/kontobuch/
**Repo:** https://github.com/hendrikjansenhsd/kontobuch (public, persönlicher Account,
nicht die Hochschul-Org `HSD-W-MBA`, unter der der Account sonst auch hängt)
**Lokal:** `~/Projects/kontobuch`

## Zweck & Umfang
Vier Konten/Quellen einlesen und auswerten:
- **Volksbank** (privat, Girokonto)
- **N26** (Geschäftskonto)
- **willbe/LLB** (Tagesgeld, für Rücklagen)
- **PayPal** (privat, nur Text-Einfügen — siehe Parser unten)

Kategorisierung mit Unterkategorien, 50/30/20-Zielerreichung, Rücklagentöpfe mit
automatischer Ratenberechnung, 12-Monats-Forecast inkl. Rücklagen-Wasserfall,
optionale Google-Drive-Anbindung zum automatischen Einlesen von Auszügen.

## Architektur
- Eine HTML-Datei, vanilla JS, kein Build-Prozess.
- Speicherung: `localStorage`. Fallback auf `window.storage`, falls die Datei
  ausnahmsweise innerhalb einer Claude-Artifact-Umgebung statt lokal geöffnet wird.
- Tabs: Dashboard, Buchungen, Rücklagen, Zuordnen, Forecast, Kategorien.
- Google Drive: optionale, rein clientseitige OAuth-Anbindung (Google Identity
  Services + Picker API), scope bewusst `drive.file` statt `drive.readonly` — die App
  bekommt nur Zugriff auf den einen vom Nutzer gewählten Ordner, nie den ganzen Drive.
  Client-ID/Ordner/importierte Datei-IDs liegen nur in `S.drive`, nie im Code.

## Datenmodell (State `S`)
- `S.tx`: Buchungen — `{id, acc, date, merchant, note, amt, cat, sub, manual, biz, ign, tr, pot}`
- `S.cats`: Hauptkategorien mit `subs[]`; jede Unterkategorie hat `goal: need|want|save` + `kw[]` (Stichwörter fürs Auto-Matching)
- `S.pots`: Rücklagentöpfe — Feldbedeutung siehe unten
- `S.fc`: Forecast — `{start, from, base, events[]}`
- `S.me`: `{name, addr}` — **nur lokal, nie im Quelltext hardcoden** (siehe Privacy-Regel)
- `S.learn`: gelernte Kategorie-Zuordnung je Händler (Schlüssel `mkey()`)
- `S.bal`: letzter bekannter Kontostand je Konto, aus Auszügen übernommen
- `S.hidden`: gelöschte Topf-Keys (auch für die Sparziel-Defaults, siehe unten)
- `S.drive`: `{clientId, folderId, folderName, imported}` — Google-Drive-Verbindung, nur lokal
- `ACCOUNTS`: `vb` (privat), `n26` (geschäft), `wb` (ruecklage), `pp` PayPal (privat)

## Parser (vier Auszugs-Formate + PayPal-Sonderfall)
1. **Volksbank PDF** — Bu-Tag/Wert-Datum ohne Jahr, Betrag mit S/H-Suffix statt Vorzeichen, mehrzeilige Buchungsblöcke (Verwendungszweck steht oft erst in der Folgezeile).
2. **N26 PDF** — Kopfzeile `Name TT.MM.JJJJ ±Betrag€`, Details darunter.
3. **willbe/LLB PDF** — Datum ohne Jahr, keine S/H-Spalte — Richtung wird aus der Saldoänderung zwischen den Zeilen bestimmt. Enthält rotierten Randtext, der gefiltert werden muss.
4. **willbe CSV-Export** — RFC4180 mit eingebetteten Zeilenumbrüchen in Feldern, Minuszeichen mit Leerzeichen abgesetzt (`"- 160,00"`).
5. **PayPal** — kein Export verfügbar, nur „Text einfügen" (Aktivitäten-Seite mit Strg/Cmd+A markieren & kopieren). Parser erkennt Buchungen an Betragszeile + folgender Datumszeile, ignoriert Navigation/Fußzeile robust statt über Skip-Liste. Vier Buchungsarten unterschiedlich behandelt:
   - **„Zahlung im Einzugsverfahren"** → reichert eine schon importierte, vage Volksbank/N26-Lastschrift („PayPal (Empfänger unbenannt)") mit dem echten Empfänger an, statt sie doppelt anzulegen. **Wichtig für den Workflow: erst Bank-Auszüge importieren, dann PayPal einfügen**, sonst findet die Anreicherung kein Ziel.
   - **„Zahlung"** (aus PayPal-Guthaben) und **„Geld erhalten/gesendet"** (Personen) → neue Buchungen auf Konto `pp`. P2P-Buchungen bewusst ohne Kategorie-Rateraten (Personennamen matchen nicht sinnvoll auf Kategorien) — landen im Zuordnen-Tab.
   - **„Sofortzahlung auf Bankkonto"** → verknüpft sich automatisch über die bestehende Umbuchungs-Erkennung (`linkTransfers`) mit der Bankgutschrift, kein Sondercode nötig.

**Bekannte Stolperfallen** (schon mal Bugs verursacht):
- `/i`-Flag an einer Regex mit `{6,}`-Quantor macht den Filter zu aggressiv und verschluckt normale Wörter.
- Format-Erkennung darf sich nicht auf IBANs stützen — die tauchen als Gegenkonto auch in fremden Auszügen auf.
- Eigener Name/Adresse/IBAN **nie** im Code — führte einmal dazu, dass private Daten im (für GitHub gedachten) Quelltext standen. Jetzt ausschließlich über `S.me`, lokal im Browser gepflegt.
- **Kurze Keywords kollidieren mit Standard-Buchungstext, nicht nur mit Referenznummern.** Bug gefunden: `'ASI '` (Kurs-Kategorie) matchte als Substring in „B**ASI**slastschrift" — betraf potenziell *jede* sonst unkategorisierte Lastschrift, nicht nur den gemeldeten Einzelfall. Bei neuen Keywords ≤4 Zeichen immer gegen gängige Buchungstext-Bausteine prüfen (Basislastschrift, Kartenzahlung girocard, Überweisungsgutschrift, IBAN/EREF/MREF/…), nicht nur gegen den einen gemeldeten Fall.
- `suggest()` filtert Referenznummern (IBAN/EREF/MREF/CRED/BIC/Auftragsnummer) über `stripRefNoise()` vorm Matching raus — ohne das matchten kurze Keywords zufällig innerhalb von Mandatsreferenzen.
- Duplikate Keyword über zwei Kategorien (z. B. „DEBEKA" gleichzeitig bei Kranken und Unfall) sind nicht zuverlässig auflösbar, wenn dieselbe Handelsregister-/Vereinsbezeichnung für unterschiedliche Vertragsarten steht — im Zweifel den Nutzer fragen, ob es wirklich nur eine Bedeutung hat, bevor das Keyword verschoben statt dupliziert wird.

## Rücklagentöpfe (Modell nach mehreren Iterationen)
Vier Gruppen: `fix` (feste Jahres-/Quartalsrechnungen), `flex` (variable Rücklagen wie Urlaub/Geschenke/Mannschaftskasse), `not` (Notgroschen), `umz` (Umzug).

- **Feste Töpfe**: Eingabe ist Abbuchungsbetrag + Fälligkeitsmonate (Mehrfachauswahl). Rate = Betrag × Anzahl Termine ÷ 12, vollautomatisch. Bestand = Rate × Monate seit der letzten Abbuchung. Feld „tatsächlich" überschreibt die berechnete Soll-Rate für den Forecast, falls der reale Dauerauftrag noch nicht angepasst wurde — **`potRate()` bevorzugt „tatsächlich" gegenüber der Soll-Formel**, das beim Korrigieren eines Topfs leicht zu übersehen.
- **Flexible Töpfe**: Rate + Stichtagsbestand (`baseAmt`/`baseYM`). Entnahmen werden manuell zugeordnet (Tab „Rücklagen" → „Entnahmen zuordnen").
- **Sparziele** (Notgroschen/Umzug): Zielbetrag, optional Zieldatum (`until`) für eine automatisch errechnete Rate. Bei Hendrik bewusst **ohne** `until` — die Rate kommt stattdessen dynamisch aus dem Forecast-Wasserfall (siehe unten), nicht aus der Datums-Formel.
- **`POT_DEFAULTS`** (feste Töpfe) und **`POT_FLEX_DEFAULTS`** (flexible Töpfe): bekannte, von Hendrik genannte Töpfe (Bahncard, ADAC, TBO, Bayer04, Google One, Scalable Capital, Urlaub, Physio, Mannschaftskasse) werden bei der **erstmaligen** Anlage automatisch mit Betrag/Monat bzw. Rate/Startguthaben vorbelegt — Datenschutz-unbedenklich, da nur Fälligkeitsmonate/Beträge, keine Namen/IBANs. **Wirkt nicht rückwirkend** auf schon existierende Töpfe im Browser des Nutzers (das ist personenbezogene Laufzeit-Daten, kein Code) — dafür braucht es einmalig manuelle Korrektur durch Hendrik selbst.
- **`ensureGoalPots()`**: legt Notgroschen (10.000 €) und Umzug (5.000 €) automatisch an, falls noch kein Topf in der jeweiligen Gruppe existiert — reine Sparziele ohne Buchungshistorie, respektiert `S.hidden` bei Löschung.
- **Wichtiges Flag-Muster**: `p.manual` verhindert, dass `derivePots()` Name/Rate aus der Zahlungshistorie erneut überschreibt, sobald der Nutzer (oder ein `*_DEFAULTS`-Eintrag) den Topf angefasst hat — **muss beim Setzen von Default-Werten explizit mitgesetzt werden**, sonst überschreibt der nächste `render()`-Lauf die Rate wieder mit dem alten historischen Wert (Bug beim Bauen der Flex-Defaults gefunden und gefixt). `p.amountManual` ist das Pendant fürs Betragsfeld bei festen Töpfen.
- **`potForTx(t)`**: Buchungen, die schon einem Topf zugeordnet sind, zeigen in der Buchungen-Tabelle „→ Topf: XY" statt eines Kategorie-Dropdowns (das dort wirkungslos wäre) und zählen nicht im „Zuordnen"-Badge.
- **`isRuecklagenTransfer`**: Kategorie „Vorsorge → Rücklagen-Überweisung" für die Volksbank-Abbuchung Richtung willbe — bewusst komplett aus `inFlow()` (Einnahmen/Ausgaben-Berechnung) ausgeschlossen, unabhängig von der Umbuchungs-Erkennung, sonst würde dieselbe Bewegung doppelt ins Sparen (50/30/20) einfließen (einmal über die willbe-Gutschrift via `isSaving`, einmal über die Volksbank-Kategorie).

## Forecast
12 Monate, Startkontostand (Girokonto) + geplante Posten (monatlich/jährlich/einmalig).
Die Summe aller Topf-Raten (`potRate()`) fließt automatisch als Ausgabe ein. Jahresrechnungen
selbst tauchen **nicht** im Forecast auf — die werden aus dem jeweiligen Topf bezahlt, das
Girokonto merkt davon nichts.

**Rücklagen-Wasserfall** (neuer Abschnitt unterhalb der Monatstabelle): verteilt den
Forecast-Überschuss (Ein − Aus, funktioniert nur weil Notgroschen/Umzug keine eigene
Rate haben und den Überschuss so nicht selbst schon wegschmälern) zuerst auf Notgroschen
bis zum Ziel, dann Umzug, der Rest als Empfehlung Richtung Scalable Capital. Zeigt auch,
ab welchem Monat beide Ziel-Töpfe voraussichtlich voll sind.

## Dashboard
- Verlauf-Chart oben, breit; darunter Ausgaben/Einnahmen nach Kategorie nebeneinander;
  darunter Zielerreichung/Veränderungen.
- Zeitraum: Presets (3/6/12/Alles) + „Eigener Zeitraum" (Von/Bis-Monatsfelder, Default
  bei erster Auswahl: letzte 12 Monate der tatsächlichen Abdeckung, nicht die volle
  historische Spanne). Ø-Werte teilen nur durch Monate mit echten Einnahmen/Ausgaben
  (`inFlow`-gefiltert), nicht durch alle Kalendermonate im Zeitraum — sonst verwässern
  Lücken (z. B. ein neu hinzugekommenes Konto) den Durchschnitt künstlich.
- Abdeckungs-Anzeige über dem Verlauf: pro Konto, welcher Monatsbereich Buchungen hat.
  Näherung, keine echte Auszugs-Tracking — ein Monat mit Auszug aber zufällig ohne
  Aktivität sieht wie eine Lücke aus.
- **Verauslagt** (GB-Button): Buchung bleibt im Privat-Scope sichtbar (ausgegraut wie
  ignorierte Buchungen), zählt aber nicht in Privat-Summen. Wechselt weiterhin
  automatisch in die Geschäftlich-Auswertung.

## Teststrategie
- **Kein Node.js in dieser Entwicklungsumgebung** — `node --check` funktioniert hier nicht.
  Stattdessen: lokalen Preview-Server über `.claude/launch.json` (`python3 -m http.server`)
  + Claude-Browser-Tools (`preview_start`/`javascript_tool`/`read_console_messages`)
  nutzen, um Syntaxfehler und Laufzeitverhalten direkt im Browser zu prüfen.
- **Wichtig beim Testen über die Browser-Tools**: `localStorage.clear()` allein reicht
  nicht für einen sauberen Neustart — der In-Memory-Zustand der Seite bleibt bestehen.
  Tab schließen + neu öffnen (oder harte Navigation), sonst testet man gegen Daten
  einer vorherigen Testrunde.
- Wo möglich mit **echten** Daten testen, nicht nur Fantasiebeispielen — z. B. wurde
  der PayPal-Parser gegen einen echten Aktivitäten-Export getestet (30 Buchungen, alle
  vier Typen exakt richtig klassifiziert). Danach immer `localStorage.clear()` und
  keine echten Namen/Beträge im Code oder in Commit-Messages.
- Nach jedem Test: temporäres `.claude/launch.json` wieder löschen (liegt außerhalb
  des Kontobuch-Repos, im übergeordneten Arbeitsverzeichnis der jeweiligen Session).

## Privacy-Regel
Name, Adresse, IBAN dürfen nie im Quelltext stehen. Der Maßstab: Die Datei muss
bedenkenlos öffentlich hostbar sein. Gilt auch für Drittdaten (z. B. Namen aus
PayPal-P2P-Zahlungen) — die App verarbeitet sie nur zur Laufzeit im Browser des
Nutzers, nichts davon landet je im Code oder in Commits.

## Setup & Workflow
- Lokal: `~/Projects/kontobuch`, eigenes Git-Repo, ein Repo pro Tool (nicht mit anderen Projekten vermischen).
- Entwicklung über Claude Code Desktop, nicht mehr claude.ai-Chat — diese Datei wird automatisch als Kontext geladen.
- Hosting: GitHub Pages (Repo public, `index.html` im Root). `gh` CLI ist auf diesem
  Mac installiert und bei `hendrikjansenhsd` (persönlicher Account) authentifiziert.
  Ein Push aktualisiert die gehostete Version — Pages-Build danach ggf. per
  `gh api -X POST repos/hendrikjansenhsd/kontobuch/pages/builds` manuell antriggern
  und mit `gh api repos/hendrikjansenhsd/kontobuch/pages/builds/latest --jq '.status'`
  auf `built` pollen, bevor man den Nutzer auf die Live-URL verweist.
- Bei jeder Änderung: siehe Teststrategie oben, nie nur mit Fantasiedaten testen.
- **Personendaten-Grenze**: Topf-Beträge, Kontostände, gelernte Kategorien etc. sind
  `S`-Laufzeitdaten im Browser des Nutzers — Code-Änderungen (auch `*_DEFAULTS`-Tabellen)
  wirken **nie rückwirkend** auf schon bestehende Objekte in seiner Live-Instanz, nur
  auf neu angelegte. Das dem Nutzer proaktiv erklären, statt stillschweigend zu
  versuchen, etwas "automatisch" zu reparieren, das es nicht sein kann.

## Offene Punkte (Stand zuletzt)
- **Manuelle Korrekturen, die nur Hendrik selbst in seiner Live-Instanz machen kann**
  (Rücklagen-Tab): Töpfe „ASI Servicegebühr", „Semesterbeitrag" und „Hendrik Jansen"
  (Duplikat von Google One) löschen; TBO-Topf „Abbuchung" auf 114 € und Feld
  „tatsächlich" auf 19 € (oder leeren) setzen.
- Müll-Topf `ACC_STMT_MTH_DT_LLB…` (Randtext-Artefakt, alter Bug) — Status der
  manuellen Löschung unklar, ggf. nochmal prüfen.
- Google Drive: Client-ID wurde erzeugt, Verbindungsaufbau (OAuth + Ordnerauswahl)
  war zuletzt noch nicht gemeinsam live getestet (nur Code-seitig verifiziert).
- Scalable-Capital-Steuerpauschale: bewusst als „fester Topf mit manuell jährlich
  aktualisiertem Betrag" modelliert, kein automatisches Auslesen des Depotwerts.
- Mobile Nutzung: kein Sync zwischen Geräten außer JSON-Sicherung oder Google Drive.

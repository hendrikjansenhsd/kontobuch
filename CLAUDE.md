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
- `S.ui`: `{potsFold}` — reine Ansichtszustände, die über Sitzungen erhalten bleiben sollen
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
- **Im UI aufgerufene Funktion war nie definiert.** `setPot(id,val)` wurde vom Dropdown „Entnahmen zuordnen" per `onchange` aufgerufen, existierte aber nicht — jede Auswahl lief in einen `ReferenceError`, es ließ sich also **keine einzige Entnahme zuordnen**. Das fiel lange nicht auf, weil der Bestand damals ohnehin geschätzt wurde und die fehlende Zuordnung nichts sichtbar veränderte. Lehre: Bei `onclick`/`onchange`-Handlern in Template-Strings prüft nichts, ob die Funktion existiert — nach dem Bauen einer neuen UI-Aktion den Handler einmal wirklich auslösen, nicht nur die Kachel ansehen. `grep -c 'function <name>'` ist der billige Check.
- Duplikate Keyword über zwei Kategorien (z. B. „DEBEKA" gleichzeitig bei Kranken und Unfall) sind nicht zuverlässig auflösbar, wenn dieselbe Handelsregister-/Vereinsbezeichnung für unterschiedliche Vertragsarten steht — im Zweifel den Nutzer fragen, ob es wirklich nur eine Bedeutung hat, bevor das Keyword verschoben statt dupliziert wird.

## Rücklagentöpfe (Modell nach mehreren Iterationen)
Vier Gruppen: `fix` (feste Jahres-/Quartalsrechnungen), `flex` (variable Rücklagen wie Urlaub/Geschenke/Mannschaftskasse), `not` (Notgroschen), `umz` (Umzug).

**Bestand = echte Zuflüsse − zugeordnete Entnahmen** (`potBalance()`). Der willbe-Auszug
trägt den Verwendungszweck jeder einzelnen Einzahlung, deshalb ist der Bestand exakt
bekannt und wird nicht geschätzt. Nur Töpfe **ohne** importierte Buchungen (`p.n===0`,
also manuell angelegte) fallen auf die rechnerische Fortschreibung `potBalanceEst()`
zurück (Rate × Monate bzw. Stichtagswert). Vorher galt die Fortschreibung immer — das lag
zwangsläufig neben dem Kontostand, sobald sich eine Rate geändert hatte, und zeigte Töpfe,
die aus einer **einzelnen großen Einzahlung** bestehen (Notgroschen, Einmal-Rücklagen),
als 0 € an: `f.n<3` setzt die Rate auf 0, und ohne `amount`/`months`/`baseAmt` fiel
`potBalance` auf `return 0`. Das betraf den größten Teil des Kontos. `p.inSum` wurde
damals schon berechnet, aber nie verwendet. Deshalb sind bei Töpfen mit Historie die
Felder „Stand am Ende" (`baseAmt`/`baseYM`) ausgeblendet — sie wären wirkungslos.
`potSrc()` zeigt in jeder Topf-Kachel die Herkunft („N Einzahlungen … − M Entnahmen …").

**Konsistenz-Probe** (gegen echte Auszüge verifiziert, Zahlen bleiben bewusst aus dieser
Doku): Summe aller Zuflüsse − Summe aller Entnahmen muss den Kontostand ergeben. Sind
alle Entnahmen einem Topf zugeordnet, bleibt als „nicht zugeordnet" **genau die Summe der
Zinsabschlüsse** übrig — die gehören zu keinem Zweck. Weicht der Rest darüber hinaus ab,
sind Entnahmen offen; der Hinweistext unter „nicht zugeordnet" benennt das mit Anzahl und
Betrag, statt pauschal „prüfe die Bestände" zu sagen.

### Konto-Abgleich (Stichtag)
Weil `setPot` jahrelang defekt war, wurden Entnahmen nie zugeordnet — die Topf-Summe lag
weit über dem Kontostand, und viele Alt-Entnahmen sind nachträglich nicht mehr zuordenbar.
Der Abgleich (`alignPlan`/`alignApply`, Panel „Konto abgleichen") löst das über einen
Stichtag statt über Nacharbeit:
- **Die Vorschau ist editierbar** — pro Topf ein Eingabefeld für den Einstandswert. Das ist
  der Kern, nicht ein Extra: bei flexiblen Töpfen ist der historische Stand *nur deshalb*
  so hoch, weil Entnahmen nie abgezogen wurden — real liegt in so einem Topf oft nur noch
  eine Monatsrate. Den echten Wert kennt nur der Nutzer. Ein früherer Entwurf hat flexible
  Töpfe auf ihrem angezeigten Wert *festgeschrieben*; das war der zentrale Denkfehler. **Diese Werte gehören nie in den Code** (persönliche Daten,
  öffentliches Repo), sondern werden eingetippt.
- Feste Töpfe (`amount` + `months`) bekommen als Vorschlag den **Soll-Stand**:
  `potNeedRate × Monate seit lastDue`, gedeckelt auf `amount` — genau so viel, dass die
  nächste Abbuchung gedeckt ist. Auch überschreibbar.
- Die **Sparziele sind die Auffang-Töpfe** und nicht editierbar: der Rest zum echten
  Kontostand füllt nach der Wasserfall-Reihenfolge des Forecasts erst den Notgroschen bis
  zu seinem Ziel, dann den Umzug. Danach ist „nicht zugeordnet" exakt 0 €.
  `goalTarget(p)` fällt dabei auf den `GOAL_DEFAULTS`-Zielbetrag zurück, wenn am Topf
  keiner eingetragen ist — sonst hätte der Notgroschen keinen Deckel und der Umzug bekäme
  nie etwas. Genau dieser Fall tritt auf, wenn der Notgroschen aus einer Einzahlung
  abgeleitet wurde statt über `ensureGoalPots` zu entstehen.
- **Der Stichtagsmonat ist wählbar** (Vorgabe: laufender Monat). Wichtig, weil „Stand zum
  1.8." zweideutig ist: mit Stichtag Ende August sind die August-Einzahlungen im
  eingetragenen Wert enthalten, mit Ende Juli kommen sie oben drauf.
- **Jede Zeile hat ein Löschhäkchen.** Töpfe ohne Einzahlung seit über 6 Monaten sind
  vorausgewählt (`alignStale`), Sparziele nie. Aktive Töpfe lassen sich zusätzlich
  abhaken — nötig etwa für einen Topf, dessen Dauerauftrag noch läuft, der aber weg soll.
- Bewusst als **Vorschau mit Bestätigung**: der Vorgang überschreibt Bestände und löscht
  Töpfe. `alignUndo()` hebt die Stichtage wieder auf; gelöschte Töpfe kommen nicht zurück
  (sie stehen in `S.hidden`).

**Was der Nutzer erwartet — und was gebaut ist:** Der eingetippte Wert ist ein
*Einstandswert*, kein wiederkehrender Sollwert. Ab dem Stichtag entwickelt sich der Topf
allein durch die echten Ein- und Auszahlungen weiter: eingetragener Wert plus die nächste
Monatsrate, minus zugeordnete Entnahmen. Es wird deshalb bewusst **kein** Sollwert pro Topf
gespeichert, der bei einem späteren Abgleich erneut gesetzt würde.

**`p.baseLock` ist der Schalter, nicht `p.baseYM`.** Das ist entscheidend: `POT_FLEX_DEFAULTS`
setzt bei der Erstanlage einen `baseAmt`/`baseYM`, der nie als Stichtag gedacht war. Hing
die Stichtagslogik an `baseYM` allein, verlor z. B. der Physio-Topf beim Update ungefragt
seine gesamte Einzahlungshistorie und fiel auf den Default-Startwert zurück — im Test
aufgefallen, bevor es live ging. Nur der Abgleich setzt `baseLock`.

**Nach einem Abgleich müssen Bewegungen vor dem Stichtag überall ausgeblendet werden**
(`alignYM()` / `afterAlign(t)`): in der Entnahmen-Liste, im „nicht zugeordnet"-Hinweis und
in der Zins-Zeile. Sonst behauptet die App weiter, es seien Entnahmen offen und die
Bestände zu hoch, obwohl der Rest exakt 0 € ist — und die Liste füllt sich dauerhaft mit
Altposten, die niemand mehr zuordnen kann.

- **Feste Töpfe**: Eingabe ist Abbuchungsbetrag + Fälligkeitsmonate (Mehrfachauswahl). Rate = Betrag × Anzahl Termine ÷ 12, vollautomatisch. Bestand = Rate × Monate seit der letzten Abbuchung. Feld „tatsächlich" überschreibt die berechnete Soll-Rate für den Forecast, falls der reale Dauerauftrag noch nicht angepasst wurde — **`potRate()` bevorzugt „tatsächlich" gegenüber der Soll-Formel**, das beim Korrigieren eines Topfs leicht zu übersehen.
- **Flexible Töpfe**: Rate + Stichtagsbestand (`baseAmt`/`baseYM`). Entnahmen werden manuell zugeordnet (Tab „Rücklagen" → „Entnahmen zuordnen").
- **Sparziele** (Notgroschen/Umzug): Zielbetrag, optional Zieldatum (`until`) für eine automatisch errechnete Rate. Bei Hendrik bewusst **ohne** `until` — die Rate kommt stattdessen dynamisch aus dem Forecast-Wasserfall (siehe unten), nicht aus der Datums-Formel.
- **`POT_DEFAULTS`** (feste Töpfe) und **`POT_FLEX_DEFAULTS`** (flexible Töpfe): bekannte, von Hendrik genannte Töpfe (Bahncard, ADAC, TBO, Bayer04, Google One, Scalable Capital, Urlaub, Physio, Mannschaftskasse) werden bei der **erstmaligen** Anlage automatisch mit Betrag/Monat bzw. Rate/Startguthaben vorbelegt — Datenschutz-unbedenklich, da nur Fälligkeitsmonate/Beträge, keine Namen/IBANs. **Wirkt nicht rückwirkend** auf schon existierende Töpfe im Browser des Nutzers (das ist personenbezogene Laufzeit-Daten, kein Code) — dafür braucht es einmalig manuelle Korrektur durch Hendrik selbst.
- **`ensureGoalPots()`**: legt Notgroschen (10.000 €) und Umzug (5.000 €) automatisch an, falls noch kein Topf in der jeweiligen Gruppe existiert — reine Sparziele ohne Buchungshistorie, respektiert `S.hidden` bei Löschung.
- **Wichtiges Flag-Muster**: `p.manual` verhindert, dass `derivePots()` Name/Rate aus der Zahlungshistorie erneut überschreibt, sobald der Nutzer (oder ein `*_DEFAULTS`-Eintrag) den Topf angefasst hat — **muss beim Setzen von Default-Werten explizit mitgesetzt werden**, sonst überschreibt der nächste `render()`-Lauf die Rate wieder mit dem alten historischen Wert (Bug beim Bauen der Flex-Defaults gefunden und gefixt). `p.amountManual` ist das Pendant fürs Betragsfeld bei festen Töpfen.
- **`potForTx(t)`**: Buchungen, die schon einem Topf zugeordnet sind, zeigen in der Buchungen-Tabelle „→ Topf: XY" statt eines Kategorie-Dropdowns (das dort wirkungslos wäre) und zählen nicht im „Zuordnen"-Badge.
- **Einzahlungen eines gelöschten Topfes verschwinden nicht.** Steht ein Topf-Key in
  `S.hidden`, werden seine Zuflüsse wie zwecklose behandelt und über den Betrag zugeordnet
  (`derivePots`, `potForTx`). Dadurch wandern die Einzahlungen des Geister-Topfes mit dem
  eigenen Namen nach dessen Löschung zu Google One — **auch ohne hinterlegten `S.me.name`**,
  was der Weg ist, auf dem Hendrik das Problem tatsächlich loswird. Vorher fiel ihr Geld
  komplett aus der Aufteilung heraus.
- **Zuflüsse ohne Zweckzeile** (`potlessInflow()` + `potByAmount()`): Ein Teil der willbe-Gutschriften kommt nur als `Gutschrift / eigener Name / Auftragsnummer` — ohne Zweck. Bei Hendrik betrifft das 13 der 22 Google-One-Einzahlungen. Je nachdem, ob der eigene Name unter `S.me` hinterlegt war, entstand daraus früher ein **Geister-Topf mit dem eigenen Namen** (das dokumentierte Duplikat „Hendrik Jansen") oder die Buchung fiel per `POT_SKIP` ganz heraus. Jetzt bilden solche Zuflüsse keinen Topf mehr, sondern werden über den **Betrag** dem Topf zugeschlagen, dessen Rate exakt passt — aber nur bei **genau einem** Treffer, sonst bleiben sie unzugeordnet statt geraten. Wichtig: Das setzt voraus, dass `S.me.name` gesetzt ist, sonst greift `isMine()` nicht.
- **Entnahmen** (`setPot()`, `potForOut()`, `potOutAuto()`): Entnahmen tragen im Auszug keinen Topf-Bezug — bei willbe heißen alle nur „Zahlung willbe", weil die Überweisung kein Kommentarfeld hat. Für **feste** Töpfe ist der Topf erschließbar (Betrag ≈ `p.amount`, Monat an einem Fälligkeitsmonat ± 1, weil das Zurückholen vom Tagesgeld nicht am Abbuchungstag passiert); der Treffer wird nur **vorgeschlagen**, gesetzt wird auf Klick. Realistische Quote bei Hendriks Daten: **2 von 20** — die großen Beträge gehören zu Urlaub/Geschenke/Physio/Semesterbeitrag, für die es keine ableitbare Regel gibt. Nicht versuchen, die Quote durch weichere Toleranzen zu heben; das produziert falsche Zuordnungen, die niemand mehr nachprüft.
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

## Rücklagen-Tab: Layout
- Die Töpfe-Liste (linke Spalte) ist **einklappbar** (`potsFold()`, Zustand in
  `S.ui.potsFold`, wird mitgespeichert). Eingeklappt bleibt nur ein „Bearbeiten"-Knopf
  stehen; die rechte Spalte schaltet dann auf zwei Spalten um (`.g2.fold>.col` wird
  `flex-direction:row`), sodass die Aufteilung die volle Höhe und den meisten Platz bekommt.
  Wichtig: nur das **direkte** Kind umstellen — `#potTools` ist ebenfalls `.col` und muss
  weiter untereinander stapeln.
- **Die Werkzeug-Panels rechts (`#potTools`) dürfen nicht schrumpfen** (`flex:0 0 auto`),
  sonst laufen ihre Texte übereinander: `.panel.fix>.b` steht auf `overflow:visible`, ein
  schrumpfender Kasten schneidet seinen Inhalt also nicht ab, sondern lässt ihn über den
  nächsten laufen. Stattdessen scrollt die Spalte als Ganzes (`overflow-y:auto`).
  Das war live sichtbar und ist leicht wieder einzubauen.
- Die Aufteilung ist ein **horizontales Balkendiagramm** (`renderBars`), nach Gruppen
  unterteilt und darin nach Betrag sortiert. Vorher ein Ring — der wurde mit 17 Töpfen
  unlesbar, weil kleine Posten zu Splittern schrumpfen. Die Balkenlänge bezieht sich auf
  den **größten Einzelposten**, nicht auf die Gesamtsumme, sonst wären alle kleinen Töpfe
  unsichtbar; der Anteil am Konto steht zusätzlich als Prozentwert daneben.

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
- **Echte Daten testen, ohne sie ins Repo zu legen**: `launch.json` mit
  `["-m","http.server","<port>","--directory","<scratchpad>"]` aus dem Scratchpad
  servieren, `index.html` per Symlink aus dem Repo dorthin verlinken und die echten
  Auszüge daneben legen. Dann im Browser `fetch('<datei>')` + `parseCSV(...)` direkt
  aufrufen, `S.tx` setzen, `derivePots()` — so lässt sich die volle Topf-Rechnung gegen
  echte Daten prüfen, ohne dass eine Datei mit Finanzdaten je im Git-Verzeichnis liegt.
  Hendriks Auszüge liegen in seinem Google Drive (Ordner „Willbe" → `export.csv`, und die
  Volksbank-Kontoauszug-PDFs); über den Drive-Connector lesbar. Danach Scratchpad-Kopien
  **und** das heruntergeladene Tool-Ergebnis löschen.
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
  (Rücklagen-Tab): Töpfe „ASI Servicegebühr" und „Semesterbeitrag" löschen (Semesterbeitrag
  ist ausgelaufen, letzter Zufluss Ende 2025; ASI läuft in den Daten dagegen aktiv weiter
  — vor dem Löschen kurz rückfragen). Den Topf „Hendrik Jansen" löschen: er bildet sich
  nicht mehr neu, **solange `S.me.name` in den Einstellungen gesetzt ist** — ist er noch
  da, blockiert er als zweiter Topf mit derselben Rate die eindeutige Betrags-Zuordnung
  der zwecklosen Google-One-Zuflüsse.
- **Alt-Entnahmen zuordnen ist hinfällig, sobald Hendrik das Konto abgleicht** (Stand
  2026-08-11: er hat sich dafür entschieden, statt die Vergangenheit nachzuarbeiten).
  Wer künftig neue Entnahmen zuordnet, tut das nur noch für Bewegungen **nach** dem
  Stichtag; die Automatik deckt dabei nur feste Jahresrechnungen ab.
- **`S.me.name` sollte gesetzt sein**, damit `isMine()` greift und beim Import gar kein Topf
  mit dem eigenen Namen entsteht. Ist er schon da, reicht inzwischen aber das Löschen: die
  Einzahlungen werden dann über den Betrag zugeordnet (siehe `S.hidden`-Regel oben).
- **TBO geklärt** (Stand 2026-08-11): Der Beitrag wurde laut Vereins-Mail erhöht. Alt
  57 € × 2/Jahr (= die 9,50 €/Mon. aus der Historie), neu 114 € × 2/Jahr. `POT_DEFAULTS`
  ist damit korrekt; Hendrik muss seinen **Dauerauftrag von 9,50 € auf 19 €** erhöhen.
  Der Topf-Hinweis warnt von selbst, solange die Ist-Rate darunter liegt.
- Müll-Topf `ACC_STMT_MTH_DT_LLB…` (Randtext-Artefakt, alter Bug) — Status der
  manuellen Löschung unklar, ggf. nochmal prüfen.
- **Kurze Keywords kollidieren weiterhin** (bewusst zurückgestellt, Hendrik: „erstmal
  nicht machen"): `squash()` entfernt auch Leerzeichen, damit ist der Wortgrenzen-Schutz
  durch ein angehängtes Leerzeichen (`'TK '`, `'STAR '`, `'O2 '`, `'RE 1'`) **wirkungslos**
  — `TK` matcht z. B. in „MARK**TK**AUF". Ohne Schutz gebaut und genauso kurz: `'GAS'`
  (GASTHAUS/GASTRONOMIE), `'OBI'` (M**OBI**LFUNK/MOBILCOM/IMMOBILIEN), `'GYM'`
  (GYMNASIUM), `'TBO'` (SPOR**TBO**DEN), `'NETTO'` (NETTOBETRAG), `'MUELLER'` (häufiger
  Nachname bei Überweisungen). Die Längen-Regel in `suggest()` rettet nur Fälle, in denen
  zufällig ein längeres Keyword im selben Text steht. Ein Fix an der **Matching-Logik**
  (Wortgrenzen) würde sofort auch Hendriks bestehende Keywords heilen; ein Fix an
  `DEFAULT_CATS` wirkt nur auf Neuanlagen.
- Google Drive: Client-ID wurde erzeugt, Verbindungsaufbau (OAuth + Ordnerauswahl)
  war zuletzt noch nicht gemeinsam live getestet (nur Code-seitig verifiziert).
- Scalable-Capital-Steuerpauschale: bewusst als „fester Topf mit manuell jährlich
  aktualisiertem Betrag" modelliert, kein automatisches Auslesen des Depotwerts.
- Mobile Nutzung: kein Sync zwischen Geräten außer JSON-Sicherung oder Google Drive.

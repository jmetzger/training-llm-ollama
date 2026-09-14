# Prompt-Engineering-Grundlagen

Voraussetzung: Ollama laeuft, mindestens ein Modell ist installiert (siehe
[ollama/installation.md](../ollama/installation.md)).

## Hintergrund

Ein Prompt besteht typischerweise aus mehreren Rollen:

| Rolle | Zweck |
|---|---|
| `system` | Rahmenbedingungen, die fuer die ganze Unterhaltung gelten (Ton, Sprache, Kontext) |
| `user` | Die eigentliche Anfrage |
| `assistant` | Bisherige Antworten des Modells (fuer Mehrschritt-Dialoge) |

Alle Beispiele hier laufen ueber die Ollama-API (`/api/chat`), die genauso von LangChain (Tag 2)
angesprochen wird.

## Schritt 1: Vage vs. praezise Anweisung

Vage Anfrage:

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Fasse zusammen: Ollama ist ein Tool zum lokalen Betrieb von LLMs."}],
  "stream": false
}'
```

Ergebnis (Beispiel): eine mehrere Absaetze lange Antwort mit Aufzaehlung von Vorteilen und
Anwendungsfaellen - obwohl nur eine Zusammenfassung verlangt war.

Praezise Anfrage mit klarer Formatvorgabe:

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Beschreibe Ollama in genau einem Satz (max. 20 Woerter). Keine Aufzaehlung, keine Erklaerung der Vorteile."}],
  "stream": false
}'
```

**Beobachtetes Ergebnis (Ueberraschung):** `llama3.2` antwortete hier mit:

```
Ollama ist eine traditionelle japanische Kochmethode zur Zubereitung von Reis und anderen Gerichten.
```

Das ist frei erfunden (Halluzination) - eine praezise Formatvorgabe erzwingt zwar Kuerze, aber
nicht Korrektheit. Das Modell "weiss" schlicht nichts Verlaessliches ueber Ollama und fuellt die
Luecke mit einer plausibel klingenden Antwort.

## Schritt 2: Grounding per System-Prompt

Wenn man den relevanten Kontext explizit mitgibt, verschwindet die Halluzination:

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {"role": "system", "content": "Kontext: Ollama ist eine Open-Source-Software zum lokalen Ausfuehren von Large Language Models auf dem eigenen Rechner."},
    {"role": "user", "content": "Beschreibe Ollama in genau einem Satz (max. 20 Woerter). Keine Aufzaehlung, keine Erklaerung der Vorteile."}
  ],
  "stream": false
}'
```

Ergebnis: korrekte, kurze Antwort auf Basis des mitgegebenen Kontexts.

**Das ist die zentrale Erkenntnis fuer Tag 2 (RAG):** Ein Modell antwortet nur so gut wie der
Kontext, den es bekommt. Retrieval Augmented Generation automatisiert genau das - passenden
Kontext aus eigenen Dokumenten in den Prompt einfuegen, bevor das Modell antwortet.

## Schritt 3: Few-Shot-Prompting

Beispiele im Prompt lenken Format und Stil der Antwort, auch ohne System-Prompt:

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Klassifiziere die Stimmung als POSITIV, NEUTRAL oder NEGATIV. Antworte NUR mit dem einen Wort, keine Erklaerung.\n\nText: Der Support hat drei Wochen nicht geantwortet.\nStimmung: NEGATIV\n\nText: Das Update lief ohne Probleme durch.\nStimmung: POSITIV\n\nText: Die Rechnung kam heute per Post.\nStimmung:"}],
  "stream": false
}'
```

Erwartete Ausgabe: `NEUTRAL` - ein einzelnes Wort, im exakt gleichen Format wie die Beispiele.

**Ausprobieren:** Den Satz "Antworte NUR mit dem einen Wort, keine Erklaerung." aus dem Prompt
entfernen und erneut ausfuehren. Das Modell antwortet dann oft mit einer ausformulierten
Begruendung statt nur mit dem Label - Few-Shot-Beispiele allein erzwingen kein Format, die
explizite Anweisung dazu schon.

## Schritt 4: Strukturierte Ausgabe erzwingen (JSON)

Ollama kann gueltiges JSON garantieren (Parameter `format: "json"`):

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Gib die Stadt und das Land als JSON zurueck. Text: Die Konferenz findet in Hamburg, Deutschland statt."}],
  "format": "json",
  "stream": false
}'
```

Erwartete Ausgabe:

```
{ "Stadt": "Hamburg", "Land": "Deutschland" }
```

Das ist die Grundlage fuer LangChain-Agents (Tag 2): Tool-Calling nutzt intern denselben
Mechanismus, um Funktionsaufrufe als strukturiertes JSON statt als Freitext zu erzeugen.

## Zusammenfassung

| Technik | Wirkt gegen |
|---|---|
| Klare Formatvorgabe ("ein Satz", "nur ein Wort") | Zu lange/unfokussierte Antworten |
| Kontext/Grounding im System-Prompt | Halluzinationen |
| Few-Shot-Beispiele | Falscher Stil/falsches Antwortschema |
| `format: "json"` | Unstrukturierte Freitextantworten, wenn Weiterverarbeitung noetig ist |

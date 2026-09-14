# Performanceoptimierung: Modellgroesse und Quantisierung

Voraussetzung: mindestens zwei unterschiedlich grosse Modelle sind installiert (siehe
[ollama/modelle.md](../ollama/modelle.md)), z.B. `granite4.1:3b` und `granite4.1:8b`.

## Hintergrund

"Quantisierung" reduziert die Praezision der Modellgewichte (z.B. von 16-Bit auf 4-Bit), um
Speicherbedarf und Rechenaufwand zu senken - auf Kosten von etwas Antwortqualitaet. Ollama-Tags
wie `:8b` (volle Groesse, meist schon Q4-quantisiert als Default-Tag) oder explizite Suffixe wie
`-q4_0`/`-q8_0` zeigen die Quantisierungsstufe. Kleinere Parameterzahl (3b vs. 8b) wirkt in die
gleiche Richtung wie staerkere Quantisierung: weniger VRAM, mehr Tokens/Sekunde, aber
potenziell schwaechere Antworten.

## Schritt 1: Tokens/Sekunde messen

```
ollama run granite4.1:3b --verbose "Erklaere in zwei Saetzen, was Retrieval Augmented Generation ist."
```

```
ollama run granite4.1:8b --verbose "Erklaere in zwei Saetzen, was Retrieval Augmented Generation ist."
```

`--verbose` zeigt am Ende Metriken wie `eval rate` (Tokens/Sekunde) und `total duration`.

## Schritt 2: VRAM parallel beobachten

Im zweiten Terminal waehrend beider Laeufe:

```
nvidia-smi --query-gpu=memory.used,memory.free --format=csv -l 1
```

## Leitfragen

- Wie stark unterscheidet sich die `eval rate` (Tokens/Sekunde) zwischen 3B und 8B?
- Wie stark unterscheidet sich die inhaltliche Qualitaet der Antwort? Reicht das kleinere Modell
  fuer diese Aufgabe?
- Auf einer 16-GB-Trainings-VM laeuft `granite4.1:8b` komplett auf der GPU (siehe
  [ollama/erste-inbetriebnahme.md](../ollama/erste-inbetriebnahme.md)) - lohnt sich das kleinere
  Modell dort ueberhaupt noch, oder nur auf ressourcenschwaecheren Maschinen?

## Getestete Ergebnisse (14.09.2026, 6-GB-Notebook, identischer Prompt, drei Wiederholungen je Modell)

| Modell | Processor (`ollama ps`) | eval rate | load duration | total duration |
|---|---|---|---|---|
| granite4.1:3b | 100% GPU (2,5 GB) | ~91 tokens/s (90,7-93,6) | 3,6-3,7 s | 4,7-5,0 s |
| granite4.1:8b | 31 %/69 % CPU/GPU (6,2 GB) | ~18 tokens/s (17,3-18,8) | 5,0-5,1 s | 10,7-22,8 s |

Reproduzierbares Ergebnis, klar wie erwartet: `granite4.1:3b` generiert **~5x schneller**
(eval rate) als `granite4.1:8b`. Der Grund steht in der `ollama ps`-Spalte: 3b laeuft
komplett auf der GPU (100 %), 8b muss einen Teil der Layer auf die CPU auslagern (31 %/69 %
CPU/GPU-Split), weil es nicht komplett in die 6 GB VRAM passt - und CPU-Rechenschritte sind
um ein Vielfaches langsamer als GPU-Schritte. Inhaltlich beantworten beide Modelle die Frage
sachlich korrekt, granite4.1:8b tendenziell etwas ausfuehrlicher.

**Hinweis fuer die Durchfuehrung:** Einzelmessungen koennen stark schwanken (z.B. durch ein
kurz zuvor noch geladenes anderes Modell, das erst verdraengt werden muss - siehe
[ollama/erste-inbetriebnahme.md](../ollama/erste-inbetriebnahme.md)). Mehrere Wiederholungen
pro Modell zu messen und die `ollama ps`-Zeile mit zu dokumentieren, ist wichtiger als eine
einzelne `--verbose`-Zahl unkommentiert zu uebernehmen.

## Einordnung

Die Entscheidung "welches Modell in Produktion" ist ein Tradeoff aus drei Achsen: Antwortqualitaet,
Latenz/Durchsatz, VRAM-Bedarf (= Hardwarekosten). Auf dieser 6-GB-Hardware ist der Fall klar:
Das kleinere Modell ist massiv schneller, weil es vollstaendig auf der GPU laeuft. Auf einer
16-GB-Trainings-VM, wo auch `granite4.1:8b` komplett auf der GPU laeuft (siehe
[ollama/erste-inbetriebnahme.md](../ollama/erste-inbetriebnahme.md)), faellt der
Geschwindigkeitsunterschied vermutlich deutlich kleiner aus - das waere ein guter Vergleichstest
auf einer echten Trainings-VM. Die Uebung in [ressourcenmanagement.md](ressourcenmanagement.md)
zeigt danach, wie man mehrere Modelle kontrolliert nebeneinander betreibt, statt nur eines
auszuwaehlen.

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

## Getestete Ergebnisse (14.09.2026, 6-GB-Notebook, identischer Prompt)

| Modell | Processor (`ollama ps`) | eval rate | load duration | total duration |
|---|---|---|---|---|
| granite4.1:3b | 100% GPU (2,5 GB) | 20,99 tokens/s | 9,7 s | 23,4 s |
| granite4.1:8b | 31 %/69 % CPU/GPU (6,2 GB) | 19,28 tokens/s | 5,0 s | 9,8 s |

Ueberraschung: Die `eval rate` (reine Token-Generierungsgeschwindigkeit) unterscheidet sich auf
diesem Notebook kaum zwischen 3B und 8B - beide werden durch die GPU/CPU-Aufteilung gebremst,
das 3B-Modell zusaetzlich durch die laengere Ladezeit (9,7 s, vermutlich weil vorher ein
groesseres Modell aus dem VRAM verdraengt werden musste). Der eigentliche Vorteil des kleineren
Modells zeigt sich also nicht in der Tokengeschwindigkeit, sondern darin, dass es komplett auf
der GPU laeuft (100 % vs. 31 %/69 %) - das wird erst auf laenglicheren Antworten oder unter
gleichzeitiger Last relevant. Inhaltlich beantworten beide Modelle die Frage sachlich korrekt;
granite4.1:8b antwortet knapper (83 Tokens) als granite4.1:3b (133 Tokens) bei gleicher
Kernaussage.

## Einordnung

Die Entscheidung "welches Modell in Produktion" ist ein Tradeoff aus drei Achsen: Antwortqualitaet,
Latenz/Durchsatz, VRAM-Bedarf (= Hardwarekosten). Es gibt kein pauschal "bestes" Modell - und wie
das Testergebnis oben zeigt, ist "kleiner = schneller" auf ressourcenknapper Hardware keine
Selbstverstaendlichkeit, wenn beide Modelle ohnehin CPU-Offloading brauchen. Auf einer
16-GB-Trainings-VM, wo granite4.1:8b komplett auf der GPU laeuft, faellt der Unterschied
vermutlich deutlicher aus. Die Uebung in [ressourcenmanagement.md](ressourcenmanagement.md)
zeigt danach, wie man mehrere Modelle kontrolliert nebeneinander betreibt, statt nur eines
auszuwaehlen.

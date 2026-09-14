# Modelle herunterladen und parametrisieren

Voraussetzung: [installation.md](installation.md) ist abgeschlossen.

## Schritt 1: Modell herunterladen

```
ollama pull llama3.2
```

Ollama laedt das Modell in mehreren Layern (Manifest, Gewichte, Parameter). Ist ein Layer schon
lokal vorhanden (z.B. weil die VM aus einem vorbereiteten Referenz-Image geklont wurde), prueft
Ollama nur die Pruefsumme (SHA256) statt neu zu laden - der Befehl laeuft dann in Sekunden statt
Minuten durch, zeigt aber trotzdem den vollstaendigen echten Ablauf:

```
pulling manifest
pulling dde5aa3fc5ff: 100% |██████████████████| 2.0 GB
...
verifying sha256 digest
writing manifest
success
```

## Schritt 2: Installierte Modelle und Details ansehen

```
ollama list
ollama show llama3.2
```

`ollama show` zeigt Architektur, Parametergroesse, Kontextlaenge und die **Capabilities** des
Modells - u.a. ob es `tools` (Tool-Calling) unterstuetzt. Das ist wichtig fuer Tag 2
(LangChain-Agents).

## Schritt 3: Modell mit eigenen Parametern anpassen

Modelle lassen sich per Modelfile parametrisieren, ohne die Basis-Gewichte neu herunterzuladen.
Verzeichnis anlegen und Modelfile erstellen:

```
mkdir -p ~/modelle/llama3.2-de
cd ~/modelle/llama3.2-de
```

```
# vi Modelfile
FROM llama3.2
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
SYSTEM "Du antwortest immer auf Deutsch, kurz und praezise."
```

| Parameter | Bedeutung |
|---|---|
| `temperature` | Zufaelligkeit der Antwort (0 = deterministisch, 1 = kreativer) |
| `num_ctx` | Kontextfenster in Token (Standard oft 4096, hier auf 8192 erhoeht) |
| `SYSTEM` | Systemprompt, der bei jeder Anfrage automatisch mitgeschickt wird |

Daraus ein neues, eigenstaendiges Modell erzeugen:

```
ollama create llama3.2-de -f Modelfile
```

Ollama uebernimmt dabei die vorhandenen Gewichts-Layer des Basismodells 1:1
(`using existing layer ...`) und erzeugt nur fuer die neuen Parameter/den Systemprompt einen
zusaetzlichen kleinen Layer - kein erneuter GB-Download.

## Schritt 4: Test

```
ollama run llama3.2-de "Tell me a fun fact about octopuses."
```

Erwartete Ausgabe (Beispiel, trotz englischer Frage auf Deutsch wegen SYSTEM-Prompt):

```
Octopussen koennen ihre Arme abwerfen und wieder anwachsen, um sich vor Raubtieren zu schuetzen.
```

**Hinweis Ladezeit:** Der erste Aufruf eines Modells nach dem Erstellen/Neustart dauert je nach
GPU-VRAM deutlich laenger als Folgeaufrufe (auf einer knappen 6-GB-Karte z.B. 20-30 Sekunden statt
Millisekunden) - das Modell wird dabei komplett neu in den Speicher geladen. Ollama haelt geladene
Modelle danach per Default 5 Minuten warm (`OLLAMA_KEEP_ALIVE`).

## Aufraeumen

```
ollama rm llama3.2-de
```

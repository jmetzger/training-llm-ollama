# Ressourcenmanagement

Voraussetzung: [quantisierung-und-performance.md](quantisierung-und-performance.md) - ihr habt
schon gesehen, dass Ollama Modelle automatisch aus dem VRAM wirft, wenn kein Platz mehr da ist
(siehe auch [ollama/erste-inbetriebnahme.md](../ollama/erste-inbetriebnahme.md)). Dieses
Verhalten laesst sich ueber Umgebungsvariablen steuern statt es dem Zufall zu ueberlassen.

## Wichtige Stellschrauben

| Variable | Wirkung | Default |
|---|---|---|
| `OLLAMA_MAX_LOADED_MODELS` | wie viele Modelle gleichzeitig im Speicher bleiben duerfen | 3x Anzahl GPUs |
| `OLLAMA_NUM_PARALLEL` | wie viele Anfragen ein geladenes Modell parallel bearbeitet | 4 (abhaengig vom Speicher) |
| `OLLAMA_KEEP_ALIVE` | wie lange ein Modell nach der letzten Anfrage im Speicher bleibt | 5m |

## Schritt 1: Aktuelles Verhalten pruefen

```
ollama ps
```

Zeitspalte `UNTIL` zeigt den aktuellen `OLLAMA_KEEP_ALIVE`-Wert in Aktion.

## Schritt 2: Override setzen

```
sudo systemctl edit ollama
```

Im Editor einfuegen:

```
[Service]
Environment="OLLAMA_KEEP_ALIVE=30s"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
```

Speichern, dann:

```
sudo systemctl restart ollama
```

**Achtung:** Der Neustart trennt gerade laufende Anfragen und entlaedt alle Modelle - auf einer
geteilten Maschine (z.B. wenn mehrere an derselben VM arbeiten) vorher abstimmen.

## Schritt 3: Effekt beobachten

Zwei Modelle nacheinander anfragen (kurzer Prompt reicht):

```
ollama run granite4.1:3b "Hallo"
ollama run llama3.2 "Hallo"
ollama ps
```

## Leitfragen

- Mit `OLLAMA_MAX_LOADED_MODELS=1`: bleibt das erste Modell nach der zweiten Anfrage noch in
  `ollama ps` stehen?
- Wartet `ollama ps` nach `OLLAMA_KEEP_ALIVE=30s` tatsaechlich nur noch 30 Sekunden, bis das
  Modell aus dem Speicher verschwindet (statt der Default-5-Minuten)?
- Warum ist `OLLAMA_NUM_PARALLEL` besonders in einer Trainings-Umgebung mit 12 Teilnehmern pro
  VM relevant, wenn jede VM aber nur einem Teilnehmer gehoert? (Stichwort: mehrere Anfragen
  eines einzelnen Nutzers, z.B. aus einer Streamlit- oder LangChain-App mit mehreren parallelen
  Chat-Sessions.)

Getestet (14.09.2026): Nach `granite4.1:3b` gleich `llama3.2` angefragt - `ollama ps` zeigt
danach nur noch `llama3.2` (29 Sekunden verbleibend), `granite4.1:3b` ist sofort verschwunden.
Nach Ablauf der 30 Sekunden ist `ollama ps` komplett leer (Default waeren hier 5 Minuten
gewesen).

## Schritt 4: Aufraeumen

Override wieder entfernen, damit die Defaults gelten:

```
sudo systemctl revert ollama
sudo systemctl restart ollama
```

## Einordnung

`OLLAMA_MAX_LOADED_MODELS` und `OLLAMA_KEEP_ALIVE` sind die direkten Stellschrauben gegen das
Verhalten aus Tag 1 ("Modell verschwindet ploetzlich aus `ollama ps`") - in Produktion will man
dieses Verhalten bewusst konfigurieren statt es als ueberraschenden Seiteneffekt zu erleben.

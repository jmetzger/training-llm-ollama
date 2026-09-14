# Praktische Uebung: Lokale Modellinbetriebnahme

Voraussetzung: [installation.md](installation.md) und [modelle.md](modelle.md) sind
abgeschlossen.

## Aufgabe

Du hast Ollama installiert und weisst, wie man Modelle laedt und parametrisiert. Jetzt wendest
du das selbststaendig an und beobachtest, wie sich Modellgroesse und VRAM auf den Betrieb
auswirken.

1. Waehle zwei unterschiedlich grosse Modelle aus deiner installierten Liste (`ollama list`)
   oder lade sie neu (`ollama pull ...`).
2. Schicke beiden Modellen dieselbe Frage, z.B. ueber die API:

```
curl http://localhost:11434/api/chat -d '{"model":"<modell1>","messages":[{"role":"user","content":"Nenne eine Hauptstadt in Europa."}],"stream":false}'
```

3. Beobachte parallel in einem zweiten Terminal:

```
ollama ps
nvidia-smi
```

## Leitfragen

- Welches Modell laeuft komplett auf der GPU (`100% GPU` in `ollama ps`), welches teilweise
  auf der CPU?
- Frage direkt danach ein zweites, groesseres Modell - bleibt das erste Modell in `ollama ps`
  stehen, oder verschwindet es?
- Warum passiert das? (Stichwort: verfuegbarer VRAM, `OLLAMA_MAX_LOADED_MODELS`)

## Beobachtung (Beispiel, getestet auf 6-GB-Notebook)

Nacheinander `llama3.2` (2,6 GB) und `granite4.1:8b` (6,2 GB) angefragt:

```
NAME               SIZE      PROCESSOR    CONTEXT    UNTIL
llama3.2:latest    2.6 GB    100% GPU     4096       4 minutes from now
```

`granite4.1:8b` taucht hier **nicht mehr** auf, obwohl es gerade erst geantwortet hat - Ollama
hat es nach der Antwort automatisch wieder aus dem Speicher entfernt, weil beide Modelle
zusammen (2,6 GB + 6,2 GB) nicht gleichzeitig in die 6 GB VRAM passen. Auf den
Trainings-VMs mit 16 GB VRAM bleiben mehr bzw. groessere Modelle gleichzeitig geladen.

## Einordnung

Kein "richtiges" Ergebnis gefordert - das Ziel ist, den Zusammenhang zwischen Modellgroesse,
VRAM und Ladezeit selbst zu sehen. Das ist die Grundlage fuer Tag 3
(Performanceoptimierung, Quantisierung, Ressourcenmanagement).

# Abschlussprojekt: Minimales produktionsnahes RAG-System

## Ziel

Baut in Zweier- oder Dreiergruppen ein RAG-System, das die Themen aus Tag 2 und Tag 3
zusammenfuehrt. Es muss nicht "fertig" im Sinne von poliert sein - es muss die folgenden
Anforderungen nachweisbar erfuellen.

## Anforderungen

1. **RAG-Kern** (Tag 2): Eigene PDF-Dokumente werden in Chroma indexiert, eine Chat-Anfrage
   beantwortet auf Basis der abgerufenen Dokumentenausschnitte, mit Quellenangabe.
2. **Zugriffskontrolle** (siehe [sicherheit-und-zugriff.md](../produktion/sicherheit-und-zugriff.md)):
   Der Endpunkt ist nicht ohne Weiteres fuer jeden erreichbar - mindestens ein Token- oder
   Passwortschutz.
3. **Audit-Log** (siehe [logging-und-audit.md](../produktion/logging-und-audit.md)): Jede
   Anfrage wird mit Zeitstempel, (pseudonymisierter) Identitaet und angefragtem Modell
   protokolliert.
4. **Ressourcenkonfiguration** (siehe [ressourcenmanagement.md](../produktion/ressourcenmanagement.md)):
   bewusste Begruendung, welches Modell verwendet wird und mit welchem
   `OLLAMA_KEEP_ALIVE`/`OLLAMA_MAX_LOADED_MODELS` es betrieben werden soll - inklusive kurzer
   schriftlicher Begruendung (Qualitaet vs. VRAM vs. Antwortzeit, siehe
   [quantisierung-und-performance.md](../produktion/quantisierung-und-performance.md)).

## Nicht gefordert

- Kein huebsches Frontend - Kommandozeile oder ein einfaches `curl`-Beispiel reicht.
- Keine Skalierung auf mehrere Nutzer gleichzeitig.
- Keine TLS-Terminierung (wird nur als Leitfrage in der Sicherheits-Uebung diskutiert).

## Ablauf

- ca. 90 Minuten Bauzeit
- Abschluss: jede Gruppe zeigt kurz (5 Min.) eine echte Anfrage End-to-End - inklusive des
  401-Falls ohne Token und einem Blick ins Audit-Log.

## Bewertungsmassstab (informell)

Kein Punktesystem - Ziel ist der Transfer: Kann die Gruppe erklaeren, *warum* sie die
jeweilige Entscheidung getroffen hat (Modellwahl, Auth-Mechanismus, Keep-Alive-Wert)? Das ist
wichtiger als die Eleganz der Umsetzung.

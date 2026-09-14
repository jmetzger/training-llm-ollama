# Logging- und Audit-Aspekte

Voraussetzung: [sicherheit-und-zugriff.md](sicherheit-und-zugriff.md) - der Mini-Auth-Proxy
laeuft.

## Schritt 1: Eingebautes Ollama-Log ansehen

Ollama laeuft als systemd-Dienst und schreibt jede Anfrage ins Journal:

```
journalctl -u ollama --no-pager -n 20
```

Ihr seht pro Request eine Zeile im Format `[GIN] <Datum> | <Statuscode> | <Dauer> | <Client-IP> | <Methode> <Pfad>`, z.B.

```
[GIN] 2026/09/14 - 18:57:55 | 200 |  3.063640187s |       127.0.0.1 | POST     "/api/chat"
```

## Leitfrage 1

Reicht dieses Log fuer ein Audit im Sinne der DSGVO (siehe Tag 1, "Lokale Nutzung vs. Cloud")?
Was fehlt, um nachzuvollziehen, **wer** (welcher Nutzer/welches Team) **welchen Prompt** an
welches Modell geschickt hat? (Ollama kennt nur die IP-Adresse, keine Nutzeridentitaet, und
protokolliert standardmaessig nicht den Prompt-Inhalt.)

## Schritt 2: Audit-Log mit Identitaet ergaenzen

Der Proxy aus der vorigen Uebung ist der richtige Ort, um das zu ergaenzen - er sieht bereits
Token (=Identitaet) und Request. Erweitert `ollama-proxy.py` um eine Zeile, die pro Request in
eine Datei schreibt:

```
# in do_POST, nach der Token-Pruefung ergaenzen (vor dem Forward an Ollama):
import json, time
payload = json.loads(body)
prompt = payload["messages"][-1]["content"] if "messages" in payload else "?"
with open("audit.log", "a") as f:
    f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')}  token={auth[:14]}...  ip={self.client_address[0]}  modell={payload.get('model')}  prompt={prompt!r}\n")
```

Proxy neu starten, erneut eine Anfrage mit Token schicken (siehe vorige Uebung), dann:

```
cat audit.log
```

Getestet (14.09.2026): Anfrage `"Wie ist das Wetter in Kiel?"` erzeugt die Zeile

```
2026-09-14 19:03:54  token=Bearer geheime...  ip=127.0.0.1  modell=granite4.1:8b  prompt='Wie ist das Wetter in Kiel?'
```

## Leitfrage 2

Prompts koennen personenbezogene Daten enthalten (Namen, Kundendaten, interne Vorgaenge). Ein
Audit-Log, das den vollen Prompt-Text mitschreibt, ist selbst ein DSGVO-relevanter Datensatz.
Was folgt daraus fuer Aufbewahrungsdauer, Zugriffsschutz auf `audit.log` und ggf.
Pseudonymisierung?

## Einordnung

Zwei Ebenen im produktiven Betrieb:

| Ebene | Beispiel | Zeigt |
|---|---|---|
| Server-Log (Ollama selbst) | `journalctl -u ollama` | technische Sicht: Status, Dauer, IP |
| Audit-Log (Gateway/Proxy davor) | `audit.log` | fachliche Sicht: wer, was, wann - mit den DSGVO-Pflichten, die daraus folgen |

Beide Ebenen zusammen ergeben Nachvollziehbarkeit, ohne dass Ollama selbst etwas dafuer koennen
muss.

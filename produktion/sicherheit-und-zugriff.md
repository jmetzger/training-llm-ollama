# Sicherheits- und Zugriffskonzepte

## Hintergrund

Ollama hat standardmaessig **keine Authentifizierung**. Wer die API erreicht, kann jedes
geladene Modell nutzen - ohne Login, ohne Token, ohne Rate-Limit. In der Default-Konfiguration
bindet Ollama nur an `127.0.0.1:11434` (nicht `0.0.0.0`), das schuetzt also vor Zugriffen aus dem
Netzwerk, aber nicht vor anderen Prozessen/Usern auf derselben Maschine.

Pruefe die Bind-Adresse auf deiner VM:

```
ss -tlnp | grep 11434
```

`127.0.0.1:11434` bedeutet: nur lokal erreichbar. `0.0.0.0:11434` waere aus dem gesamten
Netzwerk erreichbar (z.B. wenn `OLLAMA_HOST=0.0.0.0` gesetzt ist) - fuer eine produktive
Umgebung ohne zusaetzlichen Schutz ein Risiko.

## Aufgabe: Mini-Auth-Proxy vor Ollama schalten

Da Ollama selbst kein Auth-Konzept mitbringt, ist das produktionsuebliche Muster ein
**API-Gateway/Reverse-Proxy davor**, der Requests pruefen (und protokollieren) kann, bevor sie
an Ollama gehen.

## Schritt 1: Proxy-Skript schreiben

```
# vi ollama-proxy.py
import urllib.request
from http.server import BaseHTTPRequestHandler, HTTPServer

TOKEN = "geheimes-training-token"
OLLAMA_URL = "http://127.0.0.1:11434"

class ProxyHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        auth = self.headers.get("Authorization", "")
        if auth != f"Bearer {TOKEN}":
            self.send_response(401)
            self.end_headers()
            self.wfile.write(b'{"error":"unauthorized"}')
            print(f"ABGELEHNT  {self.client_address[0]}  {self.path}")
            return

        length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(length)
        req = urllib.request.Request(f"{OLLAMA_URL}{self.path}", data=body, method="POST",
                                      headers={"Content-Type": "application/json"})
        with urllib.request.urlopen(req) as resp:
            self.send_response(resp.status)
            self.end_headers()
            self.wfile.write(resp.read())
        print(f"OK  {self.client_address[0]}  {self.path}")

HTTPServer(("127.0.0.1", 8091), ProxyHandler).serve_forever()
```

Port 8091 statt des ueblichen 8080 - letzterer ist auf vielen Maschinen schon belegt (z.B. von
anderen lokalen Diensten), das fuehrt sonst zu einem verwirrenden `Address already in use`.

## Schritt 2: Proxy starten und testen

```
python3 ollama-proxy.py
```

In einem zweiten Terminal, zuerst **ohne** Token:

```
curl -i http://localhost:8091/api/chat -d '{"model":"granite4.1:8b","messages":[{"role":"user","content":"Test"}],"stream":false}'
```

Erwartung: `401 Unauthorized`, im Proxy-Terminal erscheint `ABGELEHNT`.

Jetzt **mit** korrektem Token:

```
curl -i http://localhost:8091/api/chat -H "Authorization: Bearer geheimes-training-token" -d '{"model":"granite4.1:8b","messages":[{"role":"user","content":"Test"}],"stream":false}'
```

Erwartung: `200 OK` mit der Modellantwort, im Proxy-Terminal erscheint `OK`.

Getestet (14.09.2026, granite4.1:8b bereits geladen): ohne Token `401`, mit Token `200` und
Antwort `{"model":"granite4.1:8b",...,"message":{"role":"assistant","content":"OK"},"done":true,...}`.

## Leitfragen

- Was passiert, wenn ihr direkt gegen Port 11434 (Ollama selbst) testet, ohne den Proxy? Warum
  funktioniert das genauso, obwohl kein Token mitgeschickt wird?
- Wo wuerde in einer echten Produktivumgebung der Token herkommen (Secret-Store, Vault, Umgebungsvariable) statt hart codiert im Skript?
- Welche weiteren Zugriffskonzepte waeren sinnvoll (Rate-Limiting pro Nutzer, IP-Allowlist,
  TLS-Terminierung)?

## Einordnung

Der Proxy loest zwei Tag-3-Themen gleichzeitig: Zugriffskontrolle (dieser Abschnitt) und die
Grundlage fuer Audit-Logging (naechster Abschnitt: [logging-und-audit.md](logging-und-audit.md)) - die
`print()`-Zeilen sind der erste Schritt zu einem echten Audit-Trail.

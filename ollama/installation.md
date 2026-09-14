# Ollama installieren und konfigurieren

## Hintergrund

Ollama ist ein lokaler Server fuer Large Language Models. Er laeuft als Hintergrunddienst
(systemd), stellt eine REST-API auf Port 11434 bereit und erkennt vorhandene GPUs (NVIDIA/CUDA,
AMD/ROCm) automatisch. Modelle liegen als eigenstaendige Dateien vor und werden bei Bedarf
geladen.

Getestet auf: Ubuntu 24.04 LTS, NVIDIA-Treiber vorhanden.

## Schritt 1: Installation

```
curl -fsSL https://ollama.com/install.sh | sh
```

Das Skript braucht sudo-Rechte (legt einen `ollama`-Systembenutzer an, installiert den
systemd-Service). Falls Ollama schon installiert ist, aktualisiert das Skript einfach die
bestehende Installation - der Befehl ist gefahrlos mehrfach ausfuehrbar.

Erwartete Ausgabe (Ende):

```
>>> Adding ollama user to render group...
>>> Adding ollama user to video group...
>>> Adding current user to ollama group...
>>> Creating ollama systemd service...
>>> Enabling and starting ollama service...
>>> The Ollama API is now available at 127.0.0.1:11434.
>>> Install complete. Run "ollama" from the command line.
```

## Schritt 2: Installation pruefen

```
ollama --version
systemctl status ollama --no-pager
```

Erwartete Ausgabe (Auszug):

```
Active: active (running)
Main PID: ... (ollama)
```

Der Service startet automatisch bei jedem Boot (`enabled` in der systemctl-Ausgabe).

## Schritt 3: GPU-Erkennung pruefen

Ollama erkennt die GPU beim Start automatisch. Im Service-Log steht eine Zeile
`inference compute`:

```
sudo journalctl -u ollama --no-pager | grep "inference compute"
```

Erwartete Ausgabe (Beispiel):

```
msg="inference compute" id=0 library=CUDA compute=8.6 name=CUDA0 description="NVIDIA GeForce RTX 3060 Laptop GPU" total="6.0 GiB" available="5.0 GiB"
```

**Kein Eintrag gefunden?** Dann hat Ollama keine GPU erkannt und rechnet komplett auf der CPU
(deutlich langsamer). Ursachen pruefen:

```
nvidia-smi
```

Kommt hier ein Fehler statt einer GPU-Tabelle, fehlt der NVIDIA-Treiber - dann zuerst den
Treiber installieren, danach `sudo systemctl restart ollama`.

## Schritt 4: Erster Funktionstest

```
ollama run llama3.2 "Sag in einem Satz Hallo auf Deutsch"
```

Beim ersten Aufruf laedt Ollama das Modell automatisch herunter (siehe
[modelle.md](modelle.md) fuer Details zum Download). Antwortet das Modell, laeuft die
Installation. Mit `/bye` oder Strg+D den interaktiven Chat wieder verlassen.

## Wichtige Befehle im Ueberblick

| Befehl | Zweck |
|---|---|
| `ollama serve` | Server manuell im Vordergrund starten (nur zum Debuggen, laeuft normalerweise als Service) |
| `ollama list` | Installierte Modelle anzeigen |
| `ollama ps` | Aktuell geladene Modelle inkl. CPU/GPU-Aufteilung anzeigen |
| `systemctl restart ollama` | Service neu starten (z.B. nach Treiber-Update) |
| `journalctl -u ollama -f` | Service-Log live mitverfolgen |

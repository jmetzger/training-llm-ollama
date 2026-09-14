# LangChain-Agent mit Tools

Voraussetzung: `granite4.1:8b` ist installiert (siehe
[ollama/modelle.md](../ollama/modelle.md)) - das Standardmodell fuer Tool-Calling in diesem
Training.

## Hintergrund

Ein normales Chat-Modell kann nur Text generieren. Ein **Agent** kann zusaetzlich eigene
Funktionen ("Tools") aufrufen, wenn die Aufgabe das erfordert - z.B. eine Berechnung
durchfuehren oder eine externe API abfragen - und das Ergebnis in seine Antwort einbauen.
Technisch ist das dasselbe Tool-Calling, das wir in
[prompting/grundlagen.md](../prompting/grundlagen.md) schon direkt ueber die Ollama-API
gesehen haben - LangChain macht daraus nur eine Schleife: Modell entscheidet, Tool wird
ausgefuehrt, Ergebnis geht zurueck ans Modell, bis eine finale Antwort steht.

`create_agent` ist die aktuelle LangChain-1.x-API (basiert auf LangGraph). Die noch verbreitete
Kombination `create_tool_calling_agent` + `AgentExecutor` aus aelteren Tutorials liegt jetzt in
`langchain_classic` und wird nicht mehr weiterentwickelt.

## Schritt 1: Umgebung einrichten

```
pip install langchain langchain-ollama
```

## Schritt 2: Tools definieren

```
# vi agent.py
from langchain_core.tools import tool

@tool
def add_numbers(a: float, b: float) -> float:
    """Addiert zwei Zahlen."""
    return a + b

@tool
def get_weather(city: str) -> str:
    """Gibt das aktuelle Wetter fuer eine Stadt zurueck."""
    return f"In {city} sind es 18 Grad und bewoelkt."
```

Der Docstring ist nicht nur Dokumentation - das Modell bekommt ihn als Beschreibung, um zu
entscheiden, welches Tool zu einer Anfrage passt. `get_weather` liefert hier bewusst einen
festen Beispielwert statt eine echte Wetter-API abzufragen, um die Uebung offline lauffaehig
zu halten.

## Schritt 3: Agent erstellen und aufrufen

```
# an agent.py anhaengen
from langchain_ollama import ChatOllama
from langchain.agents import create_agent

llm = ChatOllama(model="granite4.1:8b", temperature=0)
agent = create_agent(llm, tools=[add_numbers, get_weather])

result = agent.invoke({
    "messages": [{"role": "user", "content": "Was ist 47 plus 128, und wie ist das Wetter in Kiel?"}]
})

for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content}")
    if hasattr(msg, "tool_calls") and msg.tool_calls:
        print("   tool_calls:", msg.tool_calls)
```

```
python3 agent.py
```

Tatsaechliche Ausgabe (getestet):

```
[human] Was ist 47 plus 128, und wie ist das Wetter in Kiel?
[ai]
   tool_calls: [{'name': 'add_numbers', 'args': {'a': 47, 'b': 128}, ...}, {'name': 'get_weather', 'args': {'city': 'Kiel'}, ...}]
[tool] 175.0
[tool] In Kiel sind es 18 Grad und bewoelkt.
[ai] Die Summe von 47 und 128 ist **175**.
Das aktuelle Wetter in Kiel ist **18 Grad und bewölkt**.
```

`granite4.1:8b` erkennt hier in einem Durchgang, dass zwei unabhaengige Tools gebraucht werden,
ruft beide auf (`tool_calls` mit zwei Eintraegen) und fasst die beiden Tool-Ergebnisse danach
in einer einzigen, zusammenhaengenden Antwort zusammen.

## Aufgabe zum Ausprobieren

Stelle eine Frage, die gar kein Tool braucht (z.B. `"Erklaer mir in einem Satz, was ein Agent
ist."`). Prueft der Agent trotzdem, ob ein Tool passt? Schau dir dazu die `messages`-Liste an -
gibt es einen `tool_calls`-Eintrag oder antwortet das `ai`-Message direkt mit Inhalt?

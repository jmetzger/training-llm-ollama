# Praxisprojekt: Ein funktionaler RAG-Chatbot

Voraussetzung: [rag/pdf-integration.md](../rag/pdf-integration.md) und
[langchain/agent-mit-tools.md](../langchain/agent-mit-tools.md) sind abgeschlossen.

## Aufgabe

Baue einen Chatbot, der Fragen zu einem eigenen PDF-Dokument beantwortet - und dabei ehrlich
zugibt, wenn die Antwort nicht im Dokument steht, statt zu halluzinieren. Das fasst alles aus
Tag 2 zusammen: PDF laden, Chunking, Embeddings, Chroma-Suche (Retrieval) und Antwortgenerierung
mit Grounding (Generation).

## Schritt 1: Wissensbasis aufbauen

Nutze dein PDF und den Aufbau-Code aus
[rag/pdf-integration.md](../rag/pdf-integration.md) (Laden, Chunking, Einbetten in Chroma mit
`persist_directory`). Wenn die Chroma-Datenbank schon auf der Platte liegt, muss sie nicht bei
jedem Chatbot-Start neu aufgebaut werden.

## Schritt 2: Retrieval und Generation verbinden

```
# vi rag_chatbot.py
from langchain_ollama import OllamaEmbeddings, ChatOllama
from langchain_chroma import Chroma

embeddings = OllamaEmbeddings(model="nomic-embed-text")
vectorstore = Chroma(collection_name="pdf_demo", embedding_function=embeddings, persist_directory="./chroma_pdf")

llm = ChatOllama(model="granite4.1:8b", temperature=0)

def frage_beantworten(frage):
    treffer = vectorstore.similarity_search(frage, k=2)
    kontext = "\n\n".join(d.page_content for d in treffer)
    messages = [
        {
            "role": "system",
            "content": (
                "Beantworte die Frage NUR auf Basis des folgenden Kontexts. "
                "Wenn die Antwort nicht im Kontext steht, sag das ehrlich.\n\n"
                f"Kontext:\n{kontext}"
            ),
        },
        {"role": "user", "content": frage},
    ]
    return llm.invoke(messages).content

while True:
    frage = input("Frage (leer = beenden): ")
    if not frage:
        break
    print(frage_beantworten(frage))
    print()
```

```
python3 rag_chatbot.py
```

## Schritt 3: Testen - auch das Scheitern

Stelle mindestens drei Fragen:

1. Eine Frage, deren Antwort klar im PDF steht.
2. Eine zweite Frage, deren Antwort ebenfalls im PDF steht, aber in einem anderen Abschnitt.
3. Eine Frage zu einem Thema, das im PDF **nicht** vorkommt.

Tatsaechliches Testergebnis mit dem Beispieldokument aus
[rag/pdf-integration.md](../rag/pdf-integration.md):

```
Frage: Wie lange darf ich im Homeoffice arbeiten?
Antwort: Gemaess dem gegebenen Kontext ist Homeoffice an bis zu 3 Tagen pro Woche
moeglich, nach Absprache mit der Teamleitung. Die Kernarbeitszeit betraegt von
10 bis 15 Uhr.

Frage: Wie lang muss mein Passwort mindestens sein?
Antwort: Gemaess dem gegebenen Kontext muss Ihr Passwort mindestens 14 Zeichen
lang sein.

Frage: Wie viele Urlaubstage habe ich pro Jahr?
Antwort: Der bereitgestellte Kontext enthaelt keine Informationen ueber die
Anzahl der Urlaubstage, die pro Jahr gewaehrt werden. Daher kann ich auf dieser
Grundlage keine Antwort auf die Frage geben.
```

Die dritte Antwort ist der eigentliche Erfolg der Uebung: Das Modell erfindet keine Urlaubstage,
sondern gibt zu, dass die Information fehlt - moeglich nur durch den einschraenkenden
System-Prompt ("NUR auf Basis des Kontexts"). Ohne diese Einschraenkung wuerde das Modell
wahrscheinlich aus allgemeinem Vorwissen antworten (vgl. Halluzinations-Beispiel in
[prompting/grundlagen.md](../prompting/grundlagen.md)).

## Erweiterung (optional)

Baue `frage_beantworten` als LangChain-Tool (siehe
[langchain/agent-mit-tools.md](../langchain/agent-mit-tools.md)) und gib es einem Agenten
zusammen mit weiteren Tools - der Agent entscheidet dann selbst, wann er im PDF nachschlagen
muss und wann nicht.

## Aufraeumen

```
rm -rf chroma_pdf beispiel.pdf
```

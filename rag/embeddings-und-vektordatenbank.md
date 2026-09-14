# Embeddings und Vektordatenbanken

Voraussetzung: Ollama laeuft (siehe [ollama/installation.md](../ollama/installation.md)),
Python 3.10+ ist vorhanden.

## Hintergrund

Ein **Embedding-Modell** wandelt Text in einen Vektor um (eine Liste von Zahlen), der die
*Bedeutung* des Texts abbildet. Semantisch aehnliche Saetze bekommen Vektoren, die im
Vektorraum nah beieinander liegen - auch wenn sie unterschiedliche Woerter benutzen. Eine
**Vektordatenbank** wie Chroma speichert diese Vektoren und findet zu einer Anfrage die
naechstgelegenen (=aehnlichsten) Eintraege.

Das ist die Grundlage von RAG (Retrieval Augmented Generation): Bevor ein Sprachmodell
antwortet, wird per Embedding-Aehnlichkeit der passende Kontext aus eigenen Dokumenten gesucht
und dem Modell mitgegeben (siehe [prompting/grundlagen.md](../prompting/grundlagen.md),
Abschnitt "Grounding").

Chroma ist Open Source (Apache-2.0) und laeuft hier komplett lokal im eigenen Prozess - keine
Cloud, kein Account, passt zur DSGVO-Anforderung aus Tag 1.

## Schritt 1: Umgebung einrichten

```
python3 -m venv venv
source venv/bin/activate
pip install langchain langchain-ollama langchain-chroma
```

## Schritt 2: Embedding-Modell laden

```
ollama pull nomic-embed-text
```

`nomic-embed-text` ist ein kleines, spezialisiertes Modell (274 MB) nur fuer Embeddings - kein
Chat-Modell.

## Schritt 3: Dokumente einbetten und speichern

```
# vi vektordatenbank.py
from langchain_ollama import OllamaEmbeddings
from langchain_chroma import Chroma
from langchain_core.documents import Document

embeddings = OllamaEmbeddings(model="nomic-embed-text")

docs = [
    Document(page_content="Ollama ist eine Software zum lokalen Ausfuehren von LLMs auf dem eigenen Rechner."),
    Document(page_content="Chroma ist eine Vektordatenbank fuer Embeddings."),
    Document(page_content="Die Hauptstadt von Deutschland ist Berlin."),
    Document(page_content="Granite ist ein Modell von IBM mit Tool-Calling-Unterstuetzung."),
]

vectorstore = Chroma.from_documents(docs, embeddings, collection_name="test", persist_directory="./chroma_db")

query = "Wie kann ich ein Sprachmodell lokal auf meinem Laptop betreiben, ohne Cloud?"
for doc, score in vectorstore.similarity_search_with_score(query, k=2):
    print(f"score={score:.4f}  {doc.page_content}")
```

```
python3 vektordatenbank.py
```

Erwartete Ausgabe (Beispiel, niedrigerer Score = aehnlicher):

```
score=0.6961  Ollama ist eine Software zum lokalen Ausfuehren von LLMs auf dem eigenen Rechner.
score=0.8007  Granite ist ein Modell von IBM mit Tool-Calling-Unterstuetzung.
```

Der Ollama-Satz gewinnt klar, obwohl in der Frage weder "Ollama" noch sonst ein identisches
Wort vorkommt - die Suche findet die Bedeutung, nicht nur Stichwoerter.

Durch `persist_directory="./chroma_db"` legt Chroma die Datenbank als lokale SQLite-Datei ab
(`chroma_db/chroma.sqlite3`) - beim naechsten Skriptstart bleiben die Embeddings erhalten,
ohne neu berechnet werden zu muessen. Ohne dieses Argument liefe Chroma rein im Arbeitsspeicher
und waere nach Skriptende weg.

## Aufgabe zum Ausprobieren

Frage stattdessen `"Was ist die deutsche Hauptstadt?"` - welcher Satz gewinnt jetzt, und mit
welchem Score-Abstand zum Zweitplatzierten?

## Aufraeumen

```
rm -rf chroma_db
```

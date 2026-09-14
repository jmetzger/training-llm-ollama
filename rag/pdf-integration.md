# Integration eigener PDF-Dokumente

Voraussetzung: [embeddings-und-vektordatenbank.md](embeddings-und-vektordatenbank.md) ist
abgeschlossen.

## Hintergrund

In der Praxis stehen die Dokumente, die ein RAG-System durchsuchen soll, meist als PDF vor.
PDFs muessen vor dem Einbetten in drei Schritten aufbereitet werden:

1. **Text extrahieren** (PDF ist ein Layout-Format, kein reines Textformat)
2. **In Chunks aufteilen** (ein ganzes Dokument ist meist zu lang fuer ein sinnvolles Embedding)
3. **Einbetten und in Chroma speichern** (wie in der letzten Uebung)

## Schritt 1: Umgebung ergaenzen

```
pip install pypdf fpdf2
```

`pypdf` liest PDFs, `fpdf2` brauchen wir nur, um uns ein Test-PDF zu erzeugen - im echten
Training bringen Teilnehmer eigene PDFs mit.

## Schritt 2: Test-PDF erzeugen

```
# vi erstelle_test_pdf.py
from fpdf import FPDF

pdf = FPDF()
pdf.add_page()
pdf.set_font("Helvetica", size=12)
text = """IT-Nutzungsrichtlinie (Beispieldokument, Beispiel AG)

Abschnitt 1: Zugangsdaten
Passwoerter muessen mindestens 14 Zeichen lang sein und alle 180 Tage geaendert
werden. Nach 5 Fehlversuchen wird der Account fuer 30 Minuten gesperrt.

Abschnitt 2: Ausstattung
Jeder Arbeitsplatz erhaelt einen Laptop mit mindestens 32 GB RAM. Der Support
ist werktags von 7 bis 19 Uhr unter der internen Nummer 4242 erreichbar.

Abschnitt 3: Homeoffice
Homeoffice ist an bis zu 3 Tagen pro Woche moeglich, nach Absprache mit der
Teamleitung. Die Kernarbeitszeit liegt zwischen 10 und 15 Uhr.
"""
pdf.multi_cell(0, 8, text)
pdf.output("beispiel.pdf")
```

```
python3 erstelle_test_pdf.py
```

## Schritt 3: PDF laden, aufteilen, einbetten

```
# vi pdf_rag.py
from pypdf import PdfReader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_ollama import OllamaEmbeddings
from langchain_chroma import Chroma

reader = PdfReader("beispiel.pdf")
raw_docs = [
    Document(page_content=page.extract_text(), metadata={"source": "beispiel.pdf", "page": i})
    for i, page in enumerate(reader.pages)
]
print(f"Seiten geladen: {len(raw_docs)}")

splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)
chunks = splitter.split_documents(raw_docs)
print(f"Chunks nach Splitting: {len(chunks)}")

embeddings = OllamaEmbeddings(model="nomic-embed-text")
vectorstore = Chroma.from_documents(chunks, embeddings, collection_name="pdf_demo", persist_directory="./chroma_pdf")

query = "Wie lange darf ich im Homeoffice arbeiten?"
print(f"\nFrage: {query}")
for doc, score in vectorstore.similarity_search_with_score(query, k=2):
    print(f"score={score:.4f}")
    print(doc.page_content)
    print()
```

```
python3 pdf_rag.py
```

Tatsaechliche Ausgabe (getestet):

```
Seiten geladen: 1
Chunks nach Splitting: 3

Frage: Wie lange darf ich im Homeoffice arbeiten?

score=0.5557
Abschnitt 2: Ausstattung
Jeder Arbeitsplatz erhaelt einen Laptop mit mindestens 32 GB RAM. Der Support
ist werktags von 7 bis 19 Uhr unter der internen Nummer 4242 erreichbar.
Abschnitt 3: Homeoffice
Homeoffice ist an bis zu 3 Tagen pro Woche moeglich, nach Absprache mit der

score=0.8062
Teamleitung. Die Kernarbeitszeit liegt zwischen 10 und 15 Uhr.
```

Die Antwort steht in einem frei erfundenen Beispieldokument - kein Sprachmodell "weiss" das
aus seinem Training. Der Treffer beweist, dass die Antwort tatsaechlich aus dem PDF kommt und
nicht aus vermeintlichem Vorwissen des Modells (vgl. Halluzinations-Beispiel in
[prompting/grundlagen.md](../prompting/grundlagen.md)).

**Chunk-Grenzen beachten:** Der Treffer mit dem besten Score enthaelt den Satz zum Homeoffice
nur, weil `chunk_size=300` die Ueberschrift "Abschnitt 3: Homeoffice" noch mit in denselben
Chunk wie "Abschnitt 2: Ausstattung" gepackt hat - Zufall der Textlaenge, kein Bug. Der zweite
Treffer ("Kernarbeitszeit liegt zwischen 10 und 15 Uhr") gehoert inhaltlich auch noch zum
Homeoffice-Abschnitt, wurde aber durch die Chunk-Grenze abgeschnitten. Genau das ist der
Praxis-Kompromiss beim Chunking: zu kleine Chunks reissen Saetze auseinander, zu grosse Chunks
vermischen Themen und verwaessern das Embedding.

## Aufgabe zum Ausprobieren

Ersetze `beispiel.pdf` durch ein eigenes PDF (z.B. eine Anleitung, ein Datenblatt) und passe
die Frage entsprechend an. Setze `chunk_size` auf 100 und dann auf 1000 - wie veraendern sich
Chunk-Anzahl, Scores und ob die Antwort ueberhaupt noch vollstaendig in einem Chunk steht?

## Aufraeumen

```
rm -rf chroma_pdf beispiel.pdf
```

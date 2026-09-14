# Deploying and Optimizing LLMs with Ollama

3-Tages-Training: Lokale LLMs mit Ollama betreiben, RAG-Systeme mit Chroma und LangChain bauen und produktionsnah absichern.

Verteilung: 30 % Theorie, 50 % Hands-on-Übungen, 20 % Projektarbeit & Transfer

## Agenda

### Tag 1 – Grundlagen & Setup

  1. Grundlagen
     * Einführung in LLM-Architekturen
     * Transformer-Prinzip (Überblick)
     * Lokale Nutzung vs. Cloud (DSGVO-Einordnung)

  1. Ollama
     * [Installation und Konfiguration von Ollama](ollama/installation.md)
     * [Download und Parametrisierung von Modellen](ollama/modelle.md)
     * [Praktische Übung: Lokale Modellinbetriebnahme](ollama/erste-inbetriebnahme.md)

  1. Prompting
     * [Prompt-Engineering-Grundlagen](prompting/grundlagen.md)

### Tag 2 – RAG & Integration

  1. RAG
     * Architektur von Retrieval Augmented Generation
     * Embeddings und Vektordatenbanken
     * Einrichtung eines lokalen Chroma Vector Stores
     * Integration eigener PDF-Dokumente
     * Vergleich RAG vs. Fine-Tuning

  1. LangChain
     * Einführung in LangChain (Chains & Agents – Demo)

  1. Praxisprojekt
     * Implementierung eines funktionalen RAG-Chatbots

### Tag 3 – Produktionsnahe Umsetzung & Transfer

  1. Produktion
     * Architekturentscheidungen
     * Logging- und Audit-Aspekte
     * Sicherheits- und Zugriffskonzepte
     * Performanceoptimierung (Modellgröße, Quantisierung)
     * Ressourcenmanagement

  1. Abschlussprojekt
     * Eigenständiger Aufbau eines minimalen produktionsnahen RAG-Systems

## Voraussetzungen

Pro Teilnehmer eine VM mit 4 vCPU, 64 GB RAM und einer GPU mit 16 GB VRAM. Ollama und die Python-Umgebung werden im Training auf der VM eingerichtet.

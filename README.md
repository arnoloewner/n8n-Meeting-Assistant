# n8n-Meeting-Assistant
[README_meeting_assistent.md](https://github.com/user-attachments/files/30179346/README_meeting_assistent.md)
# KI-Meeting-Assistent

Ein n8n-Workflow, der aus einem Meeting-Transkript automatisch eine strukturierte Zusammenfassung erzeugt, Entscheidungen und Aufgaben extrahiert und die Ergebnisse in zwei verknüpften Notion-Datenbanken ablegt.

**Kurz gesagt:** Transkript rein – dokumentiertes Meeting und angelegte Aufgaben raus.

---

## Das Problem

Nach Meetings entstehen Aufgaben, die niemand konsequent nachhält. Notizen werden nicht sauber übertragen, Zuständigkeiten bleiben unklar, Deadlines gehen unter. Die Nacharbeit kostet Zeit – und wird deshalb im Alltag oft übersprungen.

Dieser Workflow schließt die Lücke zwischen Gespräch und Nachverfolgung, ohne dass jemand manuell etwas überträgt.

## Der Ablauf

```
Formular (Titel · Datum · Transkript)
        ↓
Vorverarbeitung  →  Validierung, Spracherkennung, Chunking
        ↓
Claude API       →  strukturierte Zusammenfassung
        ↓
Claude API       →  Extraktion als JSON
                    (Entscheidungen, Aufgaben, Owner, Deadlines)
        ↓
Validierung      →  Parsing, Plausibilitätsprüfung, Defaults
        ↓
Notion           →  Meeting-Log-Eintrag
        ↓
Notion           →  je Aufgabe ein Task, verknüpft mit dem Meeting
```

**Im Detail:**

1. **Eingabe** über ein n8n-Formular: Meeting-Titel, Datum und Transkript.
2. **Vorverarbeitung** – Das Transkript wird auf Mindestlänge geprüft, die Sprache (Deutsch/Englisch) automatisch erkannt und bei Überlänge satzweise in Chunks mit Überlappung zerlegt.
3. **Zusammenfassung** über die Claude API – strukturiert nach Datum, Teilnehmern und Hauptthemen, in der erkannten Sprache.
4. **Extraktion** in einem zweiten API-Aufruf – gibt ausschließlich valides JSON zurück: getroffene Entscheidungen sowie Aufgaben mit Verantwortlichem, Deadline und Priorität. Auch implizite Fristen wie „nächste Woche" werden in konkrete Daten aufgelöst.
5. **Validierung** – Die Antwort wird geparst, die Struktur geprüft und fehlende Felder mit sinnvollen Standardwerten aufgefüllt, statt den Workflow abbrechen zu lassen.
6. **Ablage in Notion** – Ein Eintrag in der Datenbank *Meeting Log*, anschließend je erkannter Aufgabe ein Eintrag in der Datenbank *Tasks*, per Relation mit dem Meeting verknüpft.

## Technologie

| Baustein | Zweck |
|---|---|
| **n8n** (self-hosted, Docker) | Orchestrierung des Workflows |
| **Claude API** | Zusammenfassung und strukturierte Extraktion |
| **Notion API** | Ablage in zwei verknüpften Datenbanken |
| **JavaScript** (Code-Nodes) | Vorverarbeitung, Validierung, Fehlerbehandlung, Logging |

## Was den Workflow robust macht

Ein reiner Happy Path wäre schnell gebaut. Der Aufwand steckte darin, ihn belastbar zu machen:

- **Eigene Fehlerbehandlung pro API-Aufruf** – Beide Claude-Aufrufe haben einen dedizierten Error-Handler, der Statuscode, Fehlerquelle und Zeitstempel protokolliert, statt den Workflow stumm abbrechen zu lassen.
- **Validierung statt Vertrauen** – Die Modellantwort wird nicht direkt weitergereicht: JSON wird bereinigt und geparst, die erwartete Struktur geprüft, und einzelne Felder werden abgesichert (Deadline nur bei gültigem Datumsformat, Priorität nur aus einer erlaubten Werteliste, Fallback auf „mittel").
- **Sprachunabhängigkeit** – Eine einfache Worthäufigkeitsanalyse erkennt Deutsch oder Englisch und steuert damit die Sprache der Ausgabe.
- **Umgang mit langen Transkripten** – Überlange Texte werden an Satzgrenzen in überlappende Abschnitte zerlegt, statt am Kontextlimit zu scheitern.
- **Abschluss-Logging** – Am Ende wird protokolliert, wie viele Entscheidungen und Aufgaben erzeugt wurden – nachvollziehbar statt Blackbox.

## Herausforderungen

**Verwertbare Daten statt Fließtext**
Eine Zusammenfassung als Prosa lässt sich nicht in Datenbankfelder überführen. Die Lösung war die Trennung in zwei Aufrufe: erst zusammenfassen (für Menschen), dann strikt strukturiert extrahieren (für die Weiterverarbeitung).

**Verlässliches JSON aus einem Sprachmodell**
Modelle neigen dazu, Antworten mit Erklärtext oder Markdown-Backticks zu umrahmen. Gelöst über eine klare Systemanweisung plus nachgelagerte Bereinigung und Parsing mit Fehlerabfang.

**Datenübergabe zwischen Nodes**
Das gezielte Referenzieren von Ergebnissen aus vorherigen Nodes war der aufwendigste Teil der Fehlersuche – und der größte Lerneffekt im Umgang mit n8n.

**Relationen in Notion**
Aufgaben sollen nicht lose danebenstehen, sondern mit ihrem Meeting verknüpft sein. Das erforderte sauberes Mapping von Feldnamen, Typen und Relation zwischen beiden Datenbanken.

## Setup

> **Hinweis:** Dieses Repository enthält ausschließlich den Workflow-Export. Zugangsdaten sind durch Platzhalter ersetzt und müssen selbst hinterlegt werden.

1. JSON-Datei herunterladen und in n8n über **Import from File** einlesen
2. Eigene Credentials für Claude API und Notion hinterlegen
3. Zwei Notion-Datenbanken anlegen:
   - **Meeting Log** – Titel, Datum, Zusammenfassung, Entscheidungen, Task_Count
   - **Tasks** – Titel, Verantwortlich, Deadline, Priorität, Status, Relation zum Meeting Log
4. Die jeweiligen Datenbank-IDs in den Notion-Nodes eintragen
5. Workflow aktivieren und das Formular aufrufen

## Status

Funktionsfähiger Prototyp – der Workflow läuft vollständig durch, von der Formulareingabe bis zu den angelegten Notion-Einträgen.

Entstanden als Lernprojekt zum Aufbau praktischer Automatisierungskompetenz. Der Schwerpunkt lag bewusst nicht auf einem möglichst kurzen Weg zum Ergebnis, sondern auf Fehlerbehandlung, Validierung und nachvollziehbarer Struktur.

**Mögliche Erweiterungen:**
- Vorgeschaltete Transkription, damit auch der Schritt von der Aufnahme zum Text entfällt
- Benachrichtigung bei neuen Aufgaben
- Auswertung der Chunks bei sehr langen Transkripten in einem zusammenführenden Schritt

## Kontext

Entstanden aus einem eigenen Bedarf im Vertriebsalltag: Aus Gesprächen entstehen laufend Aufgaben, die in Notizen versanden. Das Projekt ist Teil einer Reihe von Automatisierungsprojekten, mit denen ich meine Vertriebserfahrung mit KI-gestützter Prozessautomatisierung verbinde.

---

**Arno Löwner** · [arno.loewner@icloud.com](mailto:arno.loewner@icloud.com)

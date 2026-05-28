Rollen:
- Admin
- Editor
- Guest

Features:
MVP
- login (username & password)
- startpage (apprenticeship, wenn eingeloggt)
- add apprenticeship (admin)
- edit apprenticeship (admin)
- delete apprenticeship (admin)
- add module (admin)
- edit module (admin)
- delete module (admin)
- add catagory (admin)
- edit catagory (admin)
- delete catagory (admin)
- add entries (admin and creator)
- edit entries (admin and creator)
- delete entries (admin and creator)
- read entries (for all users)
- search with filter feature (for all users)


NOT MVP
- iserv sso
- changelog (not mvp)
- review/change request for entries
- ticket system (user request new module → admin approve)


	                		admin                	    user
structure: apprenticeship→ lernfelder →kategorien→ entries
entry:

usecases:
- login
- manage entries
- read entries

languages:
backend: java
frontend: javascript & react

Woraus besteht ein Eintrag?
- title
- maincontent
- author
- last edited Date


 

1. Projektübersicht
Ziel des Projekts ist die Entwicklung eines Wissensmanagement-Systems für Ausbildungsberufe. Die Software strukturiert Informationen hierarchisch und ermöglicht es verschiedenen Benutzergruppen, Inhalte effizient zu verwalten und abzurufen.
Technologiestack:
•	Backend: Java, Spring Boot
•	Frontend: React
•	Datenstruktur: Ausbildungsberuf (Apprenticeship) → Lernfelder (Modules) → Kategorien → Einträge

2. Rollendefinition
Rolle	Beschreibung
Sys-Admin	Vollzugriff auf alles, kann Rolle geben und entnehmen, kann alles verändern, diese Rolle kann nicht gelöscht werden
Admin	Vollzugriff auf alle Systemkomponenten, Stammdaten (Berufe, Module) und Inhalte.
Editor	Kann fachspezifische Einträge erstellen, bearbeiten und löschen (wenn er selber den Eintrag erstellt hat).
Guest	Kann Inhalte über die Suche finden und lesen.

3. Funktionale Anforderungen
3.1 MVP (Minimum Viable Product)
Benutzerverwaltung & Zugriff:
•	Login: Authentifizierung via Benutzername und Passwort.
•	Startseite: Nach dem Login erfolgt die Weiterleitung zur Übersicht der Ausbildungsberufe.
•	Rechteprüfung: Zugriffsbeschränkung basierend auf den Rollen (Admin, Editor, Guest).

Stammdatenverwaltung (Admin only):
•	Verwaltung (CRUD: Create, Read, Update, Delete) von:
•	Apprenticeships (Ausbildungsberufe)
•	Modules (Lernfelder)
•	Categories (Kategorien)
Inhaltsverwaltung:
•	Entries (Einträge): Erstellen, Bearbeiten und Löschen durch Admins und Editoren.
•	Struktur eines Eintrags:
•	Titel
•	Hauptinhalt (Maincontent)
•	Autor (automatisch/manuell)
•	Letztes Änderungsdatum (Zeitstempel)
Informationsabruf (Alle User):
•	Lesezugriff: Ansicht aller validen Einträge.
•	Suche & Filter: Suchfunktion zum Auffinden von Einträgen inklusive Filteroptionen (z.B. nach Kategorie oder Modul).
3.2 Erweitertes Backlog (Post-MVP)
•	IServ SSO: Anbindung an das IServ-Identity-Management für Single Sign-On.
•	Changelog: Historisierung aller Änderungen an Einträgen.
•	Review-Prozess: Workflow für Änderungsanträge (Change Requests) an bestehenden Einträgen.
•	Ticket-System: Prozess für Nutzer, um neue Module bei Admins anzufragen.

4. Datenmodell & Hierarchie
Die Software folgt einer strikten Top-Down-Hierarchie zur Organisation des Wissens:
1.	Apprenticeship: Die oberste Ebene (z.B. "Fachinformatiker für Anwendungsentwicklung").
2.	Module (Lernfelder): Unterteilung des Berufs in fachliche Einheiten.
3.	Kategorien: Thematische Gruppierung innerhalb eines Moduls.
4.	Entries: Die eigentlichen Informationseinheiten mit Fachinhalten.

5. Use Cases (Anwendungsfälle)
UC1: Login
•	Akteur: Alle Rollen
•	Beschreibung: Der Nutzer gibt seine Credentials ein und erhält bei Erfolg Zugriff auf die für ihn freigeschalteten Bereiche.
UC2: Einträge verwalten (Manage Entries)
•	Akteur: Admin, Editor
•	Beschreibung: Erstellen neuer Wissensartikel oder Aktualisieren bestehender Inhalte innerhalb der definierten Kategorien.
UC3: Einträge lesen (Read Entries)
•	Akteur: Admin, Editor, Guest
•	Beschreibung: Suchen nach spezifischen Informationen und Navigation durch die Baumstruktur (Beruf -> Modul -> Kategorie), um den Zielinhalt zu lesen.

6. Nicht-funktionale Anforderungen
•	Benutzerfreundlichkeit: Intuitive Navigation durch die vier Hierarchieebenen.
•	Sicherheit: Passwortgeschützter Bereich; Schutz der Admin-Funktionen vor unbefugtem Zugriff.
•	Skalierbarkeit: Das Backend (Java) muss in der Lage sein, eine wachsende Anzahl an Einträgen und Modulen performant zu filtern.
 


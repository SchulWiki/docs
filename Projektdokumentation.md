# SchulWiki – Projektdokumentation

> Interne Wissensdatenbank für Schulen | Spring Boot 3 + React + PostgreSQL | Deployment auf Render

---

## 1. Projektüberblick

**SchulWiki** ist eine webbasierte, interne Wissensdatenbank, die speziell für den schulischen Einsatz entwickelt wurde. Lehrkräfte und Administratoren können Einträge (Records) strukturiert in Verzeichnissen (Directories) ablegen, bearbeiten und verwalten. Die Plattform unterstützt Markdown-Inhalte, rollenbasierte Zugriffssteuerung und sichere Authentifizierung per JWT.

### Kernziele

- **Strukturierte Wissensverwaltung**: Hierarchische Verzeichnisstruktur, ähnlich einem Dateisystem
- **Rollenbasierter Zugriff**: Feingranulare Berechtigungen je nach Nutzerrolle
- **Sicherheit**: JWT mit Refresh-Token, Rate Limiting, Soft-Delete für Nutzer
- **Einfache Bedienung**: Modernes React-Frontend mit Markdown-Editor

---

## 2. Funktionsübersicht

| Funktion | Beschreibung |
|---|---|
| Benutzerregistrierung & Login | JWT-basierte Authentifizierung mit Access + Refresh Token |
| Verzeichnisverwaltung | Erstellen, Umbenennen, Löschen von Verzeichnissen (hierarchisch) |
| Eintrags-CRUD | Einträge anlegen, bearbeiten (Titel & Inhalt getrennt), löschen |
| Markdown-Editor | Echtzeit-Vorschau mit react-markdown + remark/rehype |
| Volltextsuche | Backend-seitige Suche mit PostgreSQL ILIKE |
| Benutzerverwaltung (Admin) | Rollen zuweisen, Nutzer soft-deleten, gelöschte anzeigen |
| Profilverwaltung | Name, Benutzername, E-Mail, Passwort ändern |
| Rate Limiting | Schutz sensibler Endpunkte (Login, Register, Refresh) |
| SYS_ADMIN-Schutz | System-Administrator kann nicht verändert oder gelöscht werden |

---

## 3. Systemarchitektur

```
┌─────────────────────────────────────────────────────┐
│                    CLIENT (Browser)                  │
│  React 18 + Vite + TypeScript + TanStack Query       │
│  Tailwind CSS + shadcn/ui + react-markdown           │
└────────────────────┬────────────────────────────────┘
                     │ HTTPS / REST API
                     │ JWT in Authorization Header
                     │ X-Device-Fingerprint Header
┌────────────────────▼────────────────────────────────┐
│                  BACKEND (Render)                    │
│  Spring Boot 3.x + Spring Security + JPA/Hibernate  │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ Auth Module │  │  Wiki Module │  │Admin Module│  │
│  │ JWT Filter  │  │  Records API │  │ Users API  │  │
│  │ Rate Limit  │  │  Directory   │  │ Role Mgmt  │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
│                                                      │
│  AOP: @RequireRole Aspect (Rollengewicht-Prüfung)    │
└────────────────────┬────────────────────────────────┘
                     │ JDBC / Hibernate
┌────────────────────▼────────────────────────────────┐
│              PostgreSQL (Render Managed)              │
│  users, records, directories, refresh_tokens         │
└─────────────────────────────────────────────────────┘
```

---

## 4. Deployment-Topologie

```
GitHub (SchulWiki Org)
  ├── /backend   → Render Web Service (Java, port 8080)
  ├── /frontend  → Render Static Site (Vite build → /dist)
  └── /docs      → Projektdokumentation (dieses Repo)

Render Free Tier:
  - Backend:  512 MB RAM, spin-down nach Inaktivität
  - Frontend: CDN-gehostet, Vite-Build-Artefakt
  - DB:       PostgreSQL Managed, 1 GB Storage

Environment Variables (Render):
  DB_URL, DB_USERNAME, DB_PASSWORD
  JWT_SECRET, ADMIN_PASSWORD
  CORS_ALLOWED_ORIGINS
  PORT (default: 8080)
```

---

## 5. Datenmodell

```
users
  id (PK), username (unique), email (unique)
  password_hash, first_name, last_name
  role: GUEST | EDITOR | ADMIN | SYS_ADMIN
  deleted (boolean, soft-delete)

directories
  id (PK), name
  parent_id (FK → directories, nullable = root)
  created_by (FK → users)

records
  id (PK), title
  content (TEXT, Markdown)
  directory_id (FK → directories)
  created_by (FK → users)
  created_at, updated_at

refresh_tokens
  id (PK), token (hashed)
  user_id (FK → users)
  device_fingerprint
  expires_at, revoked
```

### Wichtige Designentscheidungen im Datenmodell

- **`content` als `TEXT`**: Hibernate 6 mapped `@Lob String` auf PostgreSQL `OID` (Binary Large Object), was zu Typfehlern beim Speichern führt. Lösung: `@Column(columnDefinition = "TEXT")` erzwingt den richtigen Spaltentyp.
- **Soft-Delete**: `@SQLDelete` + `@SQLRestriction("deleted = false")` — normale Queries sehen gelöschte Nutzer nicht; Admin-Queries nutzen Native SQL mit explizitem `includeDeleted`-Flag.
- **Refresh Token Fingerprint**: Jeder Token ist an einen `X-Device-Fingerprint`-Header gebunden, der bei der Ausgabe gespeichert und bei jedem Refresh verifiziert wird.

---

## 6. Rollenmodell & Berechtigungen

### Rollengewichte

| Rolle | Gewicht | Beschreibung |
|---|---|---|
| GUEST | 1 | Nur Lesezugriff (Standard für neue Nutzer) |
| EDITOR | 2 | Erstellen und Bearbeiten eigener Einträge |
| ADMIN | 3 | Verzeichnisverwaltung, Nutzerrollen ändern |
| SYS_ADMIN | 4 | Vollzugriff, unantastbar (kein Delete, kein Rolechange) |

### Berechtigungsmatrix

| Aktion | GUEST | EDITOR | ADMIN | SYS_ADMIN |
|---|---|---|---|---|
| Einträge lesen | ✓ | ✓ | ✓ | ✓ |
| Eintrag erstellen | ✗ | ✓ | ✓ | ✓ |
| Eintrag bearbeiten | ✗ | eigene | ✓ | ✓ |
| Eintrag löschen | ✗ | eigene | ✓ | ✓ |
| Verzeichnis erstellen | ✗ | ✗ | ✓ | ✓ |
| Verzeichnis löschen (leer) | ✗ | ✗ | ✓ | ✓ |
| Verzeichnis löschen (nicht leer) | ✗ | ✗ | ✗ | ✓ |
| Nutzerrollen ändern | ✗ | ✗ | ✓ (bis ADMIN) | ✓ |
| Nutzer löschen | ✗ | ✗ | ✓ | ✓ |
| Benutzerverwaltung | ✗ | ✗ | ✗ | ✓ (UI) |

### Implementierung

```java
// AOP-Aspekt: @RequireRole(Role.EDITOR) auf Service-Methoden
@Aspect @Component
public class RequireRoleAspect {
    // Prüft ob user.role.weight >= required.weight
    // Wirft ForbiddenException → HTTP 403
}
```

---

## 7. Sicherheitsarchitektur

### JWT-Flow

```
Login Request
    │
    ▼
JwtService.generateAccessToken()   → 15 Min., nur im Memory (kein localStorage)
JwtService.generateRefreshToken()  → 31 Tage, in localStorage + DB gespeichert
    │
    ▼
Jede API-Anfrage:
  Authorization: Bearer <accessToken>
  X-Device-Fingerprint: <fingerprint>
    │
    ▼
JwtFilter → Token validieren → SecurityContext setzen
    │
    ▼
401 → Axios Interceptor → POST /api/auth/refresh
  (mit refreshToken aus localStorage + Fingerprint)
    │
    ▼
Neuer accessToken → Request wiederholen
```

### Rate Limiting

```
Strict Paths (10 req/min):
  /api/auth/login
  /api/auth/register
  /api/auth/refresh
  /api/auth/valid-credentials

Default (120 req/min):
  Alle anderen Endpunkte

Antwort bei Überschreitung: HTTP 429 Too Many Requests
```

### CORS-Konfiguration

- Allowed Origins: via `${CORS_ALLOWED_ORIGINS}` (Environment Variable)
- Credentials: true
- Allowed Headers: Authorization, X-Device-Fingerprint, Content-Type
- Exposed Headers: Authorization

### Bekanntes Problem: 403 ohne CORS-Header

Spring Security kann 403 zurückgeben **bevor** der CORS-Filter greift. Browser zeigt dann "CORS Error" an — tatsächliche Ursache ist aber eine fehlende Berechtigung. Diagnose: HTTP-Status im Network Tab prüfen, nicht die Browser-Fehlermeldung.

---

## 8. Backend-Paketstruktur

```
com.schulwiki.backend
  ├── auth/
  │   ├── controller/    AuthController (login, register, refresh, logout)
  │   ├── service/       AuthService, JwtService
  │   ├── security/      SecurityConfig, JwtFilter, RateLimitFilter
  │   ├── entity/        UserEntity, RefreshTokenEntity
  │   └── dto/           LoginRequest, RegisterRequest, TokenResponse, ...
  ├── wiki/
  │   ├── controller/    RecordController, DirectoryController
  │   ├── service/       RecordService, DirectoryService
  │   ├── entity/        RecordEntity (@Column TEXT), DirectoryEntity
  │   └── dto/           RecordDto, DirectoryDto, ...
  ├── admin/
  │   ├── controller/    AdminController
  │   └── service/       AdminService (getUsers mit includeDeleted)
  └── common/
      ├── aop/           RequireRoleAspect, @RequireRole
      ├── exception/     ForbiddenException, GlobalExceptionHandler
      └── config/        FilterConfig (FilterRegistrationBean)
```

---

## 9. Technische Entscheidungen

### Spring Boot vs. Node.js/Express

**Entscheidung: Spring Boot**

Spring Boot bietet durch JPA/Hibernate, Spring Security und AOP eine vollständige, typsichere Lösung. Die Rollenlogik via AOP-Annotationen (`@RequireRole`) ist deutlich eleganter als manuelle Middleware-Ketten. Nachteil: höherer Speicherverbrauch auf dem Render Free Tier.

### PostgreSQL TEXT vs. VARCHAR

**Entscheidung: `columnDefinition = "TEXT"`**

Hibernate 6 mappt `@Lob String` zu PostgreSQL `OID` (Binary Large Object in separater Tabelle). Das führt zu Typ-Mismatch-Fehlern beim Speichern von Markdown-Inhalten. `@Column(columnDefinition = "TEXT")` erzwingt den korrekten Typ direkt in der DDL. Nach der Änderung war `ALTER TABLE records ALTER COLUMN content TYPE TEXT USING content::text;` notwendig.

### Vite Code Splitting

**Entscheidung: `manualChunks` (Funktionsform)**

Render Free Tier hat 512 MB RAM. Ein einzelner 800 KB+ JS-Chunk führte beim Minifizieren zu OOM → leere `dist/` → 404 für alle Assets. Die Lösung: `manualChunks(id)` Funktion in `vite.config.ts` teilt das Bundle in 5 Chunks ≤ 330 KB auf. Wichtig: TypeScript 6 akzeptiert nur die Funktionsform, nicht die Objekt-Syntax.

### TanStack Query vs. Redux

**Entscheidung: TanStack Query**

Für eine CRUD-Wiki-Applikation ist Server State Management (Caching, Invalidierung, Refetching) wichtiger als Client State. TanStack Query löst genau das mit minimalem Boilerplate. Redux wäre Over-Engineering für diesen Use Case.

### Soft-Delete für Nutzer

**Entscheidung: `@SQLDelete` + `@SQLRestriction`**

Echtes Löschen würde Fremdschlüssel-Constraints auf `records.created_by` brechen und die Audit-Trail löschen. Soft-Delete (`deleted = true`) bewahrt die Datenintegrität. Die Hibernate-Annotation `@SQLRestriction("deleted = false")` stellt sicher, dass gelöschte Nutzer unsichtbar sind, ohne jeden Query anzupassen.

---

## 10. Frontend-Architektur

```
src/
  ├── features/
  │   ├── auth/
  │   │   ├── useAuth.tsx        React Context + Provider
  │   │   ├── authApi.ts         Axios-Wrapper für Auth-Endpunkte
  │   │   └── auth.types.ts      User, LoginRequest, TokenResponse
  │   ├── wiki/
  │   │   ├── wikiApi.ts         CRUD für Records + Directories
  │   │   ├── useRole.ts         Berechtigungs-Hook (canEditRecord etc.)
  │   │   └── wiki.types.ts
  │   └── admin/
  │       └── adminApi.ts        getUsers, updateUserRole, deleteUser
  ├── pages/
  │   ├── LoginPage.tsx
  │   ├── DashboardPage.tsx      Verzeichnis-Browser
  │   ├── RecordPage.tsx         Markdown-Anzeige
  │   ├── RecordFormPage.tsx     Erstellen/Bearbeiten mit Editor
  │   ├── ProfilePage.tsx        Name, Konto, Passwort
  │   ├── AdminUsersPage.tsx     Benutzerverwaltung
  │   └── SearchPage.tsx         Suchergebnisse
  ├── components/
  │   ├── layout/
  │   │   ├── Navbar.tsx         Suche, Navigation, User-Avatar
  │   │   └── Sidebar.tsx        Verzeichnisbaum
  │   └── ui/                    shadcn/ui Komponenten
  └── lib/
      ├── axios.ts               Interceptor (401 → Token Refresh)
      └── routes.ts              Zentralisierte Route-Konstanten
```

### Axios-Interceptor (Token Refresh)

```typescript
// failedQueue-Pattern verhindert Race Conditions bei parallelen 401-Antworten
let isRefreshing = false
let failedQueue: Array<{resolve, reject}> = []

axiosInstance.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject })
        })
      }
      isRefreshing = true
      // ... refresh token, retry queue
    }
  }
)
```

---

## 11. API-Endpunkte

### Authentifizierung

| Method | Path | Auth | Beschreibung |
|---|---|---|---|
| POST | /api/auth/register | Nein | Registrierung (→ GUEST) |
| POST | /api/auth/login | Nein | Login, gibt Access + Refresh Token |
| POST | /api/auth/refresh | Nein | Access Token erneuern |
| POST | /api/auth/logout | Ja | Refresh Token revoken |
| GET | /api/auth/me | Ja | Aktuellen Nutzer abrufen |
| PUT | /api/auth/me/profile | Ja | Name ändern |
| PUT | /api/auth/me/identity | Ja | Username/E-Mail ändern |
| PUT | /api/auth/me/password | Ja | Passwort ändern |

### Wiki

| Method | Path | Min. Rolle | Beschreibung |
|---|---|---|---|
| GET | /api/wiki/directories | GUEST | Verzeichnisbaum |
| POST | /api/wiki/directories | ADMIN | Verzeichnis erstellen |
| DELETE | /api/wiki/directories/{id} | ADMIN | Verzeichnis löschen |
| GET | /api/wiki/records | GUEST | Alle Einträge |
| GET | /api/wiki/records/{id} | GUEST | Eintrag abrufen |
| POST | /api/wiki/records | EDITOR | Eintrag erstellen |
| PATCH | /api/wiki/records/{id}/title | EDITOR | Titel ändern |
| PATCH | /api/wiki/records/{id}/content | EDITOR | Inhalt ändern |
| DELETE | /api/wiki/records/{id} | EDITOR | Eintrag löschen |
| GET | /api/wiki/search?q= | GUEST | Volltextsuche |

### Administration

| Method | Path | Min. Rolle | Beschreibung |
|---|---|---|---|
| GET | /api/admin/users | ADMIN | Nutzerliste (mit ?deleted=true) |
| PUT | /api/admin/users/{id}/role | ADMIN | Rolle ändern |
| DELETE | /api/admin/users/{id} | ADMIN | Nutzer soft-deleten |

---

## 12. Entwicklungsprozess

### Lokale Entwicklung

```bash
# Backend (Port 8080)
cd backend
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Frontend (Port 8081)
cd frontend
npm install
npm run dev

# Tests
npm run test          # Vitest Unit Tests
npm run test:coverage # Mit Coverage-Report
```

### Projektstruktur (Monorepo)

```
SchulWiki/
  ├── backend/    Spring Boot Projekt
  ├── frontend/   Vite + React Projekt
  └── docs/       Diese Dokumentation
```

### Testabdeckung

- **Frontend**: Vitest + React Testing Library
  - `Navbar.test.tsx`: Suche, Navigation, Rollenbasierte Anzeige
  - `ProfilePage.test.tsx`: Name-Sektion, Avatar, Rollenanzeige
  - `ProfilePage.konto.test.tsx`: Konto & Passwort-Sektion
- **Backend**: Spring Boot Test + JUnit 5

---

## 13. Herausforderungen & Lösungen

### Problem 1: Große Inhalte konnten nicht gespeichert werden

**Symptom**: Einträge mit langem Markdown-Inhalt ließen sich nicht speichern.

**Ursache**: Zweifaches Problem:
1. `@Lob String` in Hibernate 6 → PostgreSQL `OID` Typ → Typfehler beim Speichern
2. Frontend rief immer zuerst den Titel-PATCH auf; wenn keine Titel-Änderung → 400 "No changes detected" → Content-Update wurde nie ausgeführt

**Lösung**:
1. `@Column(columnDefinition = "TEXT")` + `ALTER TABLE records ALTER COLUMN content TYPE TEXT USING content::text;`
2. Frontend prüft `titleChanged` und `contentChanged` separat und ruft nur die nötigen Endpunkte auf

### Problem 2: 403-Fehler wurden als CORS-Fehler angezeigt

**Symptom**: Browser zeigte "CORS Error (strict-origin-when-cross-origin)" für Content-PATCH-Requests.

**Ursache**: Spring Security gab 403 zurück (fehlende Berechtigung durch `@RequireRole`) **bevor** der CORS-Filter ausgeführt wurde. Browser interpretiert fehlende CORS-Header auf einer 403-Antwort als CORS-Problem.

**Diagnose**: Im Network Tab war tatsächlich HTTP 403, kein CORS-Fehler. Ursache war ein neu angelegter Nutzer mit Standard-Rolle `GUEST`, der keine Schreibberechtigung hatte.

**Lösung**: Nutzerrolle von GUEST auf EDITOR erhöht.

### Problem 3: Deployment-Fehler (Blank Page, 404 für Assets)

**Symptom**: Produktiv-Deployment zeigte nur ein leeres `<div id="root">`. Browser-Konsole: 404 für JS/CSS-Dateien, MIME-Type-Fehler.

**Ursache**: Vite-Build erzeugte einen einzelnen 800 KB+ Bundle. Render Free Tier (512 MB RAM) lief beim Minifizieren aus Speicher → unvollständiges `dist/`-Verzeichnis. Renders `/* → index.html`-Rewrite servierte dann `index.html` für Asset-Requests → falscher MIME-Type.

**Lösung**: `manualChunks(id)` in `vite.config.ts` teilt Bundle auf:
- `vendor-markdown`: react-markdown, remark, rehype (~328 KB)
- `vendor-react`: React, React DOM, React Router (~140 KB)
- `vendor-query`: TanStack Query (~85 KB)
- `vendor-ui`: Lucide, Tailwind utilities (~50 KB)
- `index`: App-Code (~80 KB)

### Problem 4: Double-Registration von Spring Filters

**Symptom**: `JwtFilter` und `RateLimitFilter` wurden doppelt aufgerufen (einmal von Spring, einmal von Spring Security).

**Ursache**: Spring Boot registriert alle `@Component`-Filter automatisch in der Servlet-Filter-Chain **zusätzlich** zur Security-Chain.

**Lösung**: `FilterRegistrationBean.setEnabled(false)` für beide Filter — sie werden nur noch in der Security-Chain ausgeführt.

---

## 14. Deployment-Konfiguration

### Render – Backend (Web Service)

```yaml
Build Command:  ./mvnw clean package -DskipTests
Start Command:  java -jar target/backend-*.jar
Environment:    Java 21
Health Check:   /api/health
```

### Render – Frontend (Static Site)

```yaml
Build Command:  npm install && npm run build
Publish Dir:    dist
Redirects:      /* → /index.html (SPA Routing)
```

### Environment Variables (Backend)

```
DB_URL=jdbc:postgresql://...
DB_USERNAME=...
DB_PASSWORD=...
JWT_SECRET=<min. 256-bit random>
ADMIN_PASSWORD=<sicheres Passwort>
CORS_ALLOWED_ORIGINS=https://frontend-6h00.onrender.com
```

---

## 15. Demo-Ablauf (15-Minuten-Präsentation)

### Gliederung

| Zeit | Inhalt |
|---|---|
| 0–2 min | Projektvorstellung: Was ist SchulWiki? Warum? |
| 2–4 min | Live-Demo: Login, Navigation, Eintrag lesen |
| 4–6 min | Live-Demo: Eintrag erstellen (Markdown-Editor) |
| 6–8 min | Architektur erklären (Diagramm Folie 3) |
| 8–10 min | Sicherheit: JWT-Flow, Rate Limiting, Rollen |
| 10–12 min | Technische Highlights: TEXT-Fix, Code Splitting, AOP |
| 12–14 min | Admin-Ansicht: Benutzerverwaltung live zeigen |
| 14–15 min | Fazit, Ausblick, Fragen |

### Demo-Benutzer für Präsentation

- **system_administrator** (SYS_ADMIN): Vollzugriff, Benutzerverwaltung
- **demo_editor** (EDITOR): Erstellen und Bearbeiten von Einträgen
- **demo_guest** (GUEST): Nur Lesezugriff — zeigt Berechtigungsgrenzen

### Empfohlene Demo-Sequenz

1. Als GUEST einloggen → versuchen, Eintrag zu erstellen → Fehler zeigen
2. Als EDITOR einloggen → Eintrag mit langem Markdown erstellen/bearbeiten
3. Als SYS_ADMIN einloggen → Benutzerverwaltung → Rolle ändern → Nutzer löschen und wiederfinden

---

## Anhang: Technologie-Stack

| Bereich | Technologie | Version |
|---|---|---|
| Backend Framework | Spring Boot | 3.x |
| Sprache (Backend) | Java | 21 |
| ORM | Hibernate / JPA | 6.x |
| Security | Spring Security | 6.x |
| JWT | jjwt | 0.12.6 |
| Datenbank | PostgreSQL | 16 |
| Build (Backend) | Maven | 3.9 |
| Frontend Framework | React | 18 |
| Sprache (Frontend) | TypeScript | 5/6 |
| Build (Frontend) | Vite | 5 |
| Styling | Tailwind CSS | 4 |
| UI-Komponenten | shadcn/ui | - |
| State Management | TanStack Query | v5 |
| HTTP Client | Axios | - |
| Markdown | react-markdown + remark | - |
| Tests (Frontend) | Vitest + RTL | - |
| Hosting | Render | Free Tier |

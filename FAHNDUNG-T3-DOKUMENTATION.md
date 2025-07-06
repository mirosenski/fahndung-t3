# Fahndung-T3 – Komplette Dokumentation

**Stand:** 6. Juli 2025 · System Timezone Europe/Berlin

## 1. Projektziele

- Aufbau eines TypeScript-First Full-Stack-Projekts (T3-Stack)
- Stack-Komponenten: Next.js 15, tRPC ^12, Prisma 5, Tailwind CSS ^3, NextAuth ^5, Bun 1.2 als Package-Manager/Runtime
- PostgreSQL 16 in einem Docker-Container (Port 5434) → Entwicklungs- & später Produktions-DB
- GitHub OAuth Integration für Authentifizierung

## 2. System-Anforderungen

| Komponente | Version | Installation / Quelle |
|------------|---------|---------------------|
| Ubuntu 22.04 (WSL 2) | - | bereits vorhanden |
| Node.js | 20 LTS | `curl -fsSL https://deb.nodesource.com/setup_20.x \| sudo -E bash -` |
| Bun | 1.2.18 | `curl -fsSL https://bun.sh/install \| bash` |
| Docker Engine | 24.x | `sudo apt-get install docker.io docker-compose` |
| Git | 2.43+ | `sudo apt-get install git` |

**Pfad-Aktualisierung nach Bun-Setup:**
```bash
source ~/.bashrc   # oder neues Terminal öffnen
which bun          # /home/<user>/.bun/bin/bun
```

## 3. Verzeichnis-Struktur anlegen

```bash
mkdir -p ~/projects/miro/github && cd ~/projects/miro/github
```

## 4. T3-App generieren (interaktiv)

```bash
rm -rf fahndung-t3            # falls Reste vorhanden
bun create t3-app@latest fahndung-t3
```

**CLI-Antworten:**

| Frage | Antwort |
|-------|---------|
| TypeScript? | Yes |
| Tailwind CSS? | Yes |
| tRPC? | Yes |
| Authentication | NextAuth.js |
| ORM | Prisma |
| App Router? | Yes |
| DB Provider | PostgreSQL |
| Lint/Format | ESLint / Prettier |
| Git init? | Yes |
| bun install? | Yes |
| Import Alias | ~/ |

**Nach dem Scaffold:**
```
✔ Dependencies installed
✔ eslint & prettier executed
✔ git repo initialised
Next steps:
  cd fahndung-t3
  bun run dev
```

## 5. GitHub-Repo verbinden

```bash
cd fahndung-t3
git remote add origin git@github.com:mirosenski/fahndung-t3.git
git commit -m "feat: initial T3 stack"   # falls noch nicht
git push -u origin main
```

## 6. PostgreSQL-Container (Port 5434)

### 6.1 docker-compose.yml

```yaml
version: '3.8'
services:
  db:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: Vv8XeY7#pP!daB03hQts
      POSTGRES_DB: fahndung
    ports:
      - "5434:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
volumes:
  db_data:
```

### 6.2 Container starten

```bash
docker compose up -d db
docker ps                  # Status check
```

## 7. Sichere Passwort-Konfiguration

### 7.1 Datenbank-Passwort manuell setzen

**Ziel:** .env und docker-compose.yml sollen ein sicheres, festes Passwort enthalten.

**Aktion 1:** Neues Passwort wählen
```
Vv8XeY7#pP!daB03hQts
```

**Aktion 2:** In .env ändern
```env
POSTGRES_PASSWORD=Vv8XeY7#pP!daB03hQts
DATABASE_URL="postgresql://postgres:Vv8XeY7%23pP%21daB03hQts@localhost:5434/fahndung"
```

**Aktion 3:** In docker-compose.yml ändern
```yaml
environment:
  POSTGRES_DB: fahndung
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: Vv8XeY7#pP!daB03hQts
```

**Aktion 4:** Container neu starten
```bash
docker compose down
docker compose up -d db
```

**Aktion 5:** Verbindung testen
```bash
bunx prisma db push
```

## 8. GitHub OAuth App erstellen

### 8.1 GitHub OAuth App konfigurieren

Gehe zu: https://github.com/settings/developers

**Neue OAuth App erstellen:**

| Feld | Wert |
|------|------|
| Application name | Fahndung App |
| Homepage URL | http://localhost:3004 |
| Authorization callback URL | http://localhost:3004/api/auth/callback/github |

**Erhaltene Werte:**
- Client ID: `Ov23lijQlvWgGHytyG2G`
- Client Secret: `601efbf1b3d390c3e225f133daf4f16eb7b1b193`

### 8.2 .env Datei erweitern

```env
# GitHub OAuth (Alternative Namen)
GITHUB_ID="Ov23lijQlvWgGHytyG2G"
GITHUB_SECRET="601efbf1b3d390c3e225f133daf4f16eb7b1b193"

# NextAuth Configuration
NEXTAUTH_SECRET="EQ9RO6aIPbADdDr3weKAkt3TJEDh8fdd6ipHBB2nO44="
NEXTAUTH_URL="http://localhost:3004"
```

**Secret generieren:**
```bash
openssl rand -base64 32
```

## 9. NextAuth konfigurieren

### 9.1 GitHub Provider hinzufügen

**Datei:** `src/server/auth/config.ts`

```typescript
import { PrismaAdapter } from "@auth/prisma-adapter";
import { type DefaultSession, type NextAuthConfig } from "next-auth";
import DiscordProvider from "next-auth/providers/discord";
import GitHubProvider from "next-auth/providers/github";

import { db } from "~/server/db";

export const authConfig = {
  providers: [
    DiscordProvider,
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
  adapter: PrismaAdapter(db),
  callbacks: {
    session: ({ session, user }) => ({
      ...session,
      user: {
        ...session.user,
        id: user.id,
      },
    }),
  },
} satisfies NextAuthConfig;
```

### 9.2 Umgebungsvariablen registrieren

**Datei:** `src/env.js`

```javascript
export const env = createEnv({
  server: {
    AUTH_SECRET:
      process.env.NODE_ENV === "production"
        ? z.string()
        : z.string().optional(),
    AUTH_DISCORD_ID: z.string().optional(),
    AUTH_DISCORD_SECRET: z.string().optional(),
    GITHUB_ID: z.string().optional(),
    GITHUB_SECRET: z.string().optional(),
    NEXTAUTH_SECRET: z.string().optional(),
    NEXTAUTH_URL: z.string().url().optional(),
    DATABASE_URL: z.string().url(),
    POSTGRES_PASSWORD: z.string().optional(),
    NODE_ENV: z
      .enum(["development", "test", "production"])
      .default("development"),
  },
  runtimeEnv: {
    AUTH_SECRET: process.env.AUTH_SECRET,
    AUTH_DISCORD_ID: process.env.AUTH_DISCORD_ID,
    AUTH_DISCORD_SECRET: process.env.AUTH_DISCORD_SECRET,
    GITHUB_ID: process.env.GITHUB_ID,
    GITHUB_SECRET: process.env.GITHUB_SECRET,
    NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET,
    NEXTAUTH_URL: process.env.NEXTAUTH_URL,
    DATABASE_URL: process.env.DATABASE_URL,
    POSTGRES_PASSWORD: process.env.POSTGRES_PASSWORD,
    NODE_ENV: process.env.NODE_ENV,
  },
});
```

## 10. Datenbank-Schema und Migrationen

### 10.1 Prisma Schema (bereits vorhanden)

```prisma
// Necessary for Next auth
model Account {
    id                       String  @id @default(cuid())
    userId                   String
    type                     String
    provider                 String
    providerAccountId        String
    refresh_token            String?
    access_token             String?
    expires_at               Int?
    token_type               String?
    scope                    String?
    id_token                 String?
    session_state            String?
    user                     User    @relation(fields: [userId], references: [id], onDelete: Cascade)
    refresh_token_expires_in Int?

    @@unique([provider, providerAccountId])
}

model Session {
    id           String   @id @default(cuid())
    sessionToken String   @unique
    userId       String
    expires      DateTime
    user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
    id            String    @id @default(cuid())
    name          String?
    email         String?   @unique
    emailVerified DateTime?
    image         String?
    accounts      Account[]
    sessions      Session[]
    posts         Post[]
}

model VerificationToken {
    identifier String
    token      String   @unique
    expires    DateTime

    @@unique([identifier, token])
}
```

### 10.2 Datenbank-Tabellen erstellen

```bash
bunx prisma db push
```

## 11. Port-Konfiguration

### 11.1 Port-Konflikte lösen

**Problem:** Port 3000 ist belegt (Gitea und andere Server)

**Lösung:** Explizit Port 3004 verwenden

```bash
PORT=3004 bun run dev
```

**Finale Konfiguration:**
- **App**: http://localhost:3004
- **NEXTAUTH_URL**: http://localhost:3004
- **Datenbank**: localhost:5434

## 12. Entwicklungs-Server

```bash
PORT=3004 bun run dev
```

**URL:** http://localhost:3004

**Features:**
- Hot Reload über Bun & Next.js
- GitHub OAuth Integration
- Prisma Database Connection
- tRPC API

## 13. Problemlösungen & Fallstricke

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `Command 'bun' not found` | PATH noch nicht neu geladen | `source ~/.bashrc` oder Terminal neu öffnen |
| `Port 3000 is in use` | Gitea belegt Standardport | `PORT=3004 bun run dev` |
| `Invalid environment variables` | POSTGRES_PASSWORD Validierung | `z.string().optional()` in src/env.js |
| `The table 'public.Account' does not exist` | Datenbank-Tabellen fehlen | `bunx prisma db push` |
| `Invalid url` in DATABASE_URL | Sonderzeichen im Passwort | URL-Encoding: `%23` für `#`, `%21` für `!` |

## 14. Vollständige .env Konfiguration

```env
# When adding additional environment variables, the schema in "/src/env.js"
# should be updated accordingly.

# Next Auth
# You can generate a new secret on the command line with:
# npx auth secret
# https://next-auth.js.org/configuration/options#secret
AUTH_SECRET="7o9GYbkr5ufoZzwO9jkYiu/EYPYa6xUL3bON5d48Py0=" # Generated by create-t3-app.

# Next Auth Discord Provider
AUTH_DISCORD_ID=""
AUTH_DISCORD_SECRET=""

# Prisma
# https://www.prisma.io/docs/reference/database-reference/connection-urls#env
DATABASE_URL="postgresql://postgres:Vv8XeY7%23pP%21daB03hQts@localhost:5434/fahndung"
POSTGRES_PASSWORD=Vv8XeY7#pP!daB03hQts

# Next Auth GitHub Provider
AUTH_GITHUB_ID="Ov23lijQlvWgGHytyG2G"
AUTH_GITHUB_SECRET="601efbf1b3d390c3e225f133daf4f16eb7b1b193"

# GitHub OAuth (Alternative Namen)
GITHUB_ID="Ov23lijQlvWgGHytyG2G"
GITHUB_SECRET="601efbf1b3d390c3e225f133daf4f16eb7b1b193"

# NextAuth Configuration
NEXTAUTH_SECRET="EQ9RO6aIPbADdDr3weKAkt3TJEDh8fdd6ipHBB2nO44="
NEXTAUTH_URL="http://localhost:3004"
```

## 15. Nächste Schritte

### 15.1 Sofortige Verbesserungen
- [ ] **Session Management**: Automatische Session-Erneuerung
- [ ] **Error Handling**: Benutzerfreundliche Fehlermeldungen
- [ ] **Loading States**: Spinner während OAuth-Prozess
- [ ] **User Profile**: Benutzerprofil-Seite erstellen

### 15.2 Testing & Qualität
- [ ] **Vitest + React Testing Library** integrieren
  ```bash
  bun add -D vitest @testing-library/react @testing-library/jest-dom
  ```
- [ ] **E2E Tests** mit Playwright
  ```bash
  bun add -D @playwright/test
  ```
- [ ] **TypeScript Strict Mode** aktivieren

### 15.3 CI/CD Pipeline
- [ ] **GitHub Actions** – Install → Lint → Test → Build → Deploy
- [ ] **Docker-Image** für Produktion (multi-stage Dockerfile)
- [ ] **CSP Header** konfigurieren
- [ ] **Security Headers** implementieren

### 15.4 Performance & UX
- [ ] **tRPC Batching** für bessere Performance
- [ ] **Tailwind content Paths** optimieren
- [ ] **next/image** für optimierte Bilder
- [ ] **Accessibility Audit** durchführen
- [ ] **PWA Features** hinzufügen

### 15.5 Erweiterte Features
- [ ] **Role-based Access Control** (RBAC)
- [ ] **API Rate Limiting**
- [ ] **Real-time Features** mit WebSockets
- [ ] **File Upload** Integration
- [ ] **Email Notifications**

## 16. Cheat Sheet – wichtigste Befehle

```bash
# Container
docker compose up -d db          # startet Postgres (Port 5434)
docker compose down              # stoppt & entfernt Container
docker ps                        # Container Status

# Prisma
bunx prisma migrate dev --name <msg>   # Migration
bunx prisma db push                    # Schema pushen
bunx prisma studio                     # DB GUI

# Dev-Server
PORT=3004 bun run dev                  # App starten
curl http://localhost:3004             # App testen

# Qualität
bunx eslint . --max-warnings=0        # Linting
bunx prettier --check .               # Formatierung prüfen
bunx tsc --noEmit                     # TypeScript prüfen

# Git
git add -A && git commit -m "<msg>" && git push

# Troubleshooting
lsof -i :3004                         # Port-Belegung prüfen
docker logs fahndung-t3-db-1          # DB-Logs anzeigen
```

## 17. Projektstatus

**✅ Abgeschlossen:**
- [x] T3-Stack Setup
- [x] PostgreSQL Docker Container
- [x] Sichere Passwort-Konfiguration
- [x] GitHub OAuth App erstellt
- [x] NextAuth mit GitHub Provider konfiguriert
- [x] Datenbank-Tabellen erstellt
- [x] Port-Konflikte gelöst (Port 3004)
- [x] Umgebungsvariablen validiert
- [x] GitHub OAuth Authentifizierung funktioniert

**🚀 Aktueller Status:**
- **App läuft:** http://localhost:3004
- **Datenbank:** localhost:5434
- **GitHub OAuth:** ✅ Funktioniert
- **NextAuth:** ✅ Konfiguriert
- **Prisma:** ✅ Verbunden

## 18. Kontakt & Support

Bei Fragen zu weiteren Schritten (Auth-Flows, Tests, Deployment, Accessibility-Audits, Performance-Optimierung, Security-Hardening, etc.) einfach im Chat melden – ich helfe gern weiter!

---

**Dokumentation erstellt:** 6. Juli 2025  
**Letzte Aktualisierung:** 6. Juli 2025  
**Version:** 1.0.0 
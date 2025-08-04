# Padelio Serve (NestJS)

Backend API for running **Americano-style padel events** (Americano, Mixed, Team Americano, Mexicano, Mixicano, Team Mexicano, Super Mexicano).
Includes pairing generation with **max 2-round rest rule**, standings, scheduling, and scoring.

## Features

- 7 formats (Americano & Mexicano variants, including **Super Mexicano** bonus).
- Pairing engine enforces **no player rests > 2 rounds** if courts allow.
- Players, Teams, Tournaments, Rounds, Matches, Standings.
- Postgres + Prisma ORM.
- OpenAPI (Swagger) docs.
- Dockerized local dev.

---

## Tech Stack

- **Node.js 20+**, **NestJS 10+**
- **PostgreSQL 14+**, **Prisma**
- **Jest** for tests, **ESLint/Prettier**
- **Docker Compose** for local infra

---

## Quick Start

### 1) Prerequisites

- Node.js 20+ and npm
- Docker & Docker Compose

### 2) Clone & Install

```bash
git clone https://github.com/<your-org>/padelio-serve.git
cd padelio-serve
npm ci
```

### 3) Environment

Create `.env` from the template:

```bash
cp .env.example .env
```

**`.env.example`**

```env
# App
PORT=3000
NODE_ENV=development
CORS_ORIGINS=http://localhost:5173,http://localhost:3000

# Database (Prisma expects this exact var)
DATABASE_URL="postgresql://padelio:padelio@localhost:5432/padelio?schema=public"

# Pairing behavior
MAX_REST_STREAK=2

# (Optional) Auth
JWT_SECRET=change-me
```

### 4) Start Postgres

```bash
docker compose up -d db
```

### 5) Generate & Migrate DB

```bash
npm run prisma:generate
npm run prisma:migrate:dev
# optional: seed demo data
npm run seed
```

### 6) Run the API

```bash
npm run start:dev
# Swagger UI: http://localhost:3000/docs
# Health:     http://localhost:3000/health
```

---

## NPM Scripts

```json
{
  "scripts": {
    "start": "nest start",
    "start:dev": "nest start --watch",
    "build": "nest build",
    "lint": "eslint \"src/**/*.{ts,js}\"",
    "format": "prettier --write \"**/*.{ts,js,json,md}\"",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:e2e": "jest --config ./test/jest-e2e.json",

    "prisma:generate": "prisma generate",
    "prisma:migrate:dev": "prisma migrate dev",
    "prisma:studio": "prisma studio",

    "seed": "ts-node prisma/seed.ts"
  }
}
```

---

## Project Structure

```
src/
  app.module.ts
  main.ts
  health/health.module.ts
  common/
    filters/ ...
    interceptors/ ...
    dto/ ...
  db/
    prisma.module.ts
    prisma.service.ts
  tournaments/
    tournaments.module.ts
    tournaments.controller.ts
    tournaments.service.ts
    dto/ ...
  players/
  teams/
  rounds/
  matches/
  standings/
  pairing/
    pairing.module.ts
    pairing.service.ts     # 7-format pairing algorithms (+ rest rule)
    strategies/
      americano.strategy.ts
      mixed-americano.strategy.ts
      team-americano.strategy.ts
      mexicano.strategy.ts
      mixicano.strategy.ts
      team-mexicano.strategy.ts
      super-mexicano.strategy.ts
prisma/
  schema.prisma
  seed.ts
docker/
  docker-compose.yml
```

---

## Database (Prisma)

**`prisma/schema.prisma` (essential excerpt)**

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client { provider = "prisma-client-js" }

model Player {
  id        String   @id @default(cuid())
  name      String
  gender    String   // "M" | "F"
  rating    Int?
  createdAt DateTime @default(now())
  TournamentPlayers TournamentPlayer[]
  TeamMembers       TeamMember[]
}

model Tournament {
  id          String   @id @default(cuid())
  name        String
  format      TournamentFormat
  startDate   DateTime
  courtCount  Int      @default(1)
  createdAt   DateTime @default(now())
  Rounds      Round[]
  Teams       Team[]
  Players     TournamentPlayer[]
  SuperConfig SuperMexicanoConfig?
}

enum TournamentFormat {
  americano
  mixed
  team_americano
  mexicano
  mixicano
  team_mexicano
  super_mexicano
}

model TournamentPlayer {
  tournamentId String
  playerId     String
  tournament   Tournament @relation(fields: [tournamentId], references: [id])
  player       Player     @relation(fields: [playerId], references: [id])

  @@id([tournamentId, playerId])
}

model Team {
  id           String     @id @default(cuid())
  tournamentId String
  name         String
  tournament   Tournament @relation(fields: [tournamentId], references: [id])
  members      TeamMember[]
}

model TeamMember {
  teamId   String
  playerId String
  team     Team   @relation(fields: [teamId], references: [id])
  player   Player @relation(fields: [playerId], references: [id])

  @@id([teamId, playerId])
}

model Round {
  id           String     @id @default(cuid())
  tournamentId String
  roundNumber  Int
  scheduledAt  DateTime?
  tournament   Tournament @relation(fields: [tournamentId], references: [id])
  matches      Match[]
}

model Match {
  id        String  @id @default(cuid())
  roundId   String
  courtIdx  Int
  startTime DateTime?
  metadata  Json?
  round     Round   @relation(fields: [roundId], references: [id])
  participants MatchParticipant[]
}

model MatchParticipant {
  matchId   String
  side      Int
  playerId  String?
  teamId    String?
  points    Int      @default(0)
  match     Match    @relation(fields: [matchId], references: [id])

  @@id([matchId, side])
}

model SuperMexicanoConfig {
  tournamentId String  @id
  bonusPoints  Int     @default(1)
  bonusFromRound Int   @default(2)
  tournament   Tournament @relation(fields: [tournamentId], references: [id])
}
```

Run:

```bash
npm run prisma:generate
npm run prisma:migrate:dev
```

---

## Seeding

**`prisma/seed.ts`** (minimal example)

```ts
import { PrismaClient, TournamentFormat } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  // players
  const players = await Promise.all(
    ['Ana', 'Ben', 'Caro', 'Dan', 'Ella', 'Finn', 'Gio', 'Hana'].map(
      (name, i) =>
        prisma.player.create({
          data: { name, gender: i % 2 === 0 ? 'F' : 'M' },
        }),
    ),
  );

  // tournament
  const t = await prisma.tournament.create({
    data: {
      name: 'Friday Social',
      format: 'mixicano',
      startDate: new Date(),
      courtCount: 2,
    },
  });

  await prisma.tournamentPlayer.createMany({
    data: players.map((p) => ({ tournamentId: t.id, playerId: p.id })),
  });

  console.log('Seed done:', { tournamentId: t.id });
}

main().finally(() => prisma.$disconnect());
```

Run: `npm run seed`

---

## Pairing Engine (7 Formats)

The engine enforces **MAX_REST_STREAK** (default 2). It selects who must play (those resting ≥2 rounds), fills remaining slots by longest rest, then pairs per format constraints.

**`src/pairing/pairing.service.ts` (excerpt)**

```ts
import { Injectable } from '@nestjs/common';

type Match = { players: [string, string]; metadata?: Record<string, any> };
type RoundHistory = Match[][];
type GenderMap = Record<string, 'M' | 'F'>;

@Injectable()
export class PairingService {
  constructor() {}

  private computeRestStreaks(ids: string[], history: RoundHistory) {
    const m: Record<string, number> = {};
    for (const id of ids) {
      let s = 0;
      for (let i = history.length - 1; i >= 0; i--) {
        const played = history[i].some((match) => match.players.includes(id));
        if (played) break;
        s++;
      }
      m[id] = s;
    }
    return m;
  }

  private selectActive(
    ids: string[],
    history: RoundHistory,
    courts: number,
    maxRest = 2,
  ) {
    const slots = courts * 2;
    const streaks = this.computeRestStreaks(ids, history);
    const must = ids.filter((id) => streaks[id] >= maxRest);
    const others = ids.filter((id) => streaks[id] < maxRest);

    if (must.length >= slots) {
      return must.sort((a, b) => streaks[b] - streaks[a]).slice(0, slots);
    }
    const need = slots - must.length;
    return must.concat(
      others.sort((a, b) => streaks[b] - streaks[a]).slice(0, need),
    );
  }

  private pair(ids: string[]): Match[] {
    const arr = ids.slice().sort(() => Math.random() - 0.5);
    const out: Match[] = [];
    for (let i = 0; i + 1 < arr.length; i += 2)
      out.push({ players: [arr[i], arr[i + 1]] });
    return out;
  }

  americano(ids: string[], history: RoundHistory, courts: number) {
    const active = this.selectActive(ids, history, courts);
    return this.pair(active);
  }

  mixedAmericano(
    ids: string[],
    genders: GenderMap,
    history: RoundHistory,
    courts: number,
  ) {
    const active = this.selectActive(ids, history, courts);
    const men = active.filter((id) => genders[id] === 'M');
    const women = active.filter((id) => genders[id] === 'F');
    const n = Math.min(men.length, women.length, courts);
    const matches: Match[] = [];
    for (let i = 0; i < n; i++) matches.push({ players: [men[i], women[i]] });
    return matches;
  }

  teamAmericano(teamIds: string[], history: RoundHistory, courts: number) {
    const active = this.selectActive(teamIds, history, courts);
    return this.pair(active);
  }

  mexicano(
    ids: string[],
    scores: Record<string, number>,
    history: RoundHistory,
    courts: number,
  ) {
    const active = this.selectActive(ids, history, courts);
    const sorted = active.sort((a, b) => (scores[b] || 0) - (scores[a] || 0));
    return this.pair(sorted);
  }

  mixicano(
    ids: string[],
    genders: GenderMap,
    scores: Record<string, number>,
    history: RoundHistory,
    courts: number,
  ) {
    const active = this.selectActive(ids, history, courts);
    const sorted = active.sort((a, b) => (scores[b] || 0) - (scores[a] || 0));
    const used = new Set<string>();
    const matches: Match[] = [];
    for (let i = 0; i < sorted.length; i++) {
      const a = sorted[i];
      if (used.has(a)) continue;
      for (let j = i + 1; j < sorted.length; j++) {
        const b = sorted[j];
        if (!used.has(b) && genders[a] !== genders[b]) {
          matches.push({ players: [a, b] });
          used.add(a);
          used.add(b);
          break;
        }
      }
    }
    return matches.slice(0, courts);
  }

  teamMexicano(
    teamIds: string[],
    teamScores: Record<string, number>,
    history: RoundHistory,
    courts: number,
  ) {
    const active = this.selectActive(teamIds, history, courts);
    const sorted = active.sort(
      (a, b) => (teamScores[b] || 0) - (teamScores[a] || 0),
    );
    return this.pair(sorted);
  }

  superMexicano(
    ids: string[],
    genders: GenderMap,
    scores: Record<string, number>,
    history: RoundHistory,
    courts: number,
    bonusPoints = 1,
    fromRound = 2,
  ) {
    const base = this.mixicano(ids, genders, scores, history, courts);
    return base.map((m) => ({
      players: m.players,
      metadata: { bonusPoints, fromRound },
    }));
  }
}
```

---

## Endpoints (Example)

- `POST /tournaments` – create tournament
- `POST /tournaments/:id/players` – add players
- `POST /tournaments/:id/teams` – add teams & members
- `POST /tournaments/:id/rounds/generate` – **generate next round pairings**
- `POST /matches/:id/result` – submit score
- `GET  /tournaments/:id/standings` – standings
- `GET  /health` – liveness check
- `GET  /docs` – Swagger

**Generate Next Round (example)**

```bash
curl -X POST http://localhost:3000/tournaments/<id>/rounds/generate \
  -H 'Content-Type: application/json' \
  -d '{ "force": false }'
```

---

## Controllers/Services (Example)

**`src/tournaments/tournaments.controller.ts` (excerpt)**

```ts
@Post(':id/rounds/generate')
async generate(@Param('id') id: string) {
  return this.tournamentsService.generateNextRound(id);
}
```

**`src/tournaments/tournaments.service.ts` (excerpt)**

```ts
async generateNextRound(tournamentId: string) {
  // 1) Load tournament, participants, history, scores, genders
  const ctx = await this.contextRepo.load(tournamentId); // implement with Prisma

  // 2) Dispatch to PairingService by format
  const { format, courtCount } = ctx.tournament;
  const pairs = this.pairing.dispatch(format, ctx, courtCount);

  // 3) Persist Round + Matches + Participants
  return this.roundWriter.createRoundWithMatches(tournamentId, pairs);
}
```

---

## Testing

```bash
npm test
npm run test:e2e
```

---

## Swagger

- Enabled at **`/docs`** (configure in `main.ts`):

```ts
const config = new DocumentBuilder()
  .setTitle('Padelio Serve')
  .setDescription('Americano/Mexicano Padel API')
  .setVersion('1.0.0')
  .build();
const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document);
```

---

## Notes

- Pairing fairness improves if you also track **used pair combinations** and deprioritize repeats—add a `pair_history` table or keep in `Match` metadata.
- Adjust **`MAX_REST_STREAK`** via env to change the “no more than 2-round break” behavior.
- For **auth**/multi-tenant, add `org_id` on core tables and JWT guards.

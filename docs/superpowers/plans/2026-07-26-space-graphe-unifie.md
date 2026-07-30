# Space en graphe unifié (Node/Edge) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remplacer le modèle Space/Quest/Step (arbres isolés) par un graphe unifié Space/Node/Edge où un Node (Objectif ou Étape) peut avoir plusieurs parents et contribuer à plusieurs objectifs, avec un canvas front où les Cards sont librement déplaçables et reliables — comme dans la spec [`docs/superpowers/specs/2026-07-26-space-graphe-unifie-design.md`](../specs/2026-07-26-space-graphe-unifie-design.md).

**Architecture:** Côté API (NestJS/Prisma), `Quest`/`Step` fusionnent en `Node` (`type: OBJECTIF|ETAPE`), les liens deviennent une table `Edge` autorisant plusieurs parents (DAG). Côté Web (Next.js/React Flow), le canvas affiche tout le graphe d'un Space, les Cards sont draggables avec position persistée, et les liens se créent/suppriment directement sur le canvas.

**Tech Stack:** NestJS, Prisma, PostgreSQL, Next.js, React, `@xyflow/react`, Jest, Vitest.

---

## Orchestration des sous-agents (à lire avant d'exécuter ce plan)

Ce plan sera exécuté via `superpowers:subagent-driven-development` : un sous-agent frais par tâche. Pour rester économe en tokens sans sacrifier la qualité là où ça compte, applique cette règle de routage de modèle à **chaque** dispatch d'agent (paramètre `model` du tool `Agent`) :

- **Modèle `haiku`** — tâches mécaniques, à spec fermée, sans jugement visuel : schéma Prisma, DTOs, services CRUD, contrôleurs, seed, tests unitaires/e2e API, mise à jour de fichiers de config/markdown/CLAUDE.md, suppression de fichiers obsolètes, renommages de types. Ce sont les Tasks 1 à 9 (Phase API) et les tâches non-visuelles de la Phase Web (10, 11, 16, 17, 18).
- **Modèle par défaut de la session (Sonnet, ne PAS passer `haiku`)** — toute tâche touchant le rendu, l'interaction, le drag & drop, les animations, ou le comportement du canvas React Flow. Une régression visuelle ici (Cards qui sautent, liens qui ne suivent pas le drag, HUD qui bloque le pan) est le problème exact que cette refonte doit corriger — pas de place pour l'à-peu-près d'un modèle économique. Ce sont les Tasks 12, 13, 14, 15.
- En cas de doute sur une tâche à la frontière (ex: Task 11 qui touche des types front mais pas de rendu), reste sur `haiku` si la tâche ne produit aucun pixel à l'écran ; bascule sur le modèle par défaut dès qu'un composant JSX/CSS est modifié pour un rendu visuel.
- Le **Test Runner** deux-passes de `subagent-driven-development` (implémentation puis revue) suit la même règle : la revue d'une tâche UI doit elle aussi tourner sur le modèle par défaut, jamais sur haiku.

**Master prompt à réutiliser pour chaque dispatch d'agent d'implémentation** (adapter `<N>` et `<résumé>`) :

```
Tu exécutes la Task <N> ("<résumé>") du plan
docs/superpowers/plans/2026-07-26-space-graphe-unifie.md.
Lis uniquement la section "Task <N>" de ce fichier — n'exécute pas les
autres tâches. Suis les étapes dans l'ordre exact (TDD : test d'abord,
vérifier qu'il échoue, implémenter, vérifier qu'il passe, committer).
N'invente aucun comportement hors de ce qui est écrit dans la tâche.
Si une commande échoue, corrige et reformule la même étape avant de
passer à la suivante — ne saute jamais une étape de vérification.
Rends compte en fin de tâche : fichiers modifiés, commandes exécutées,
résultat des tests.
```

---

## Phase A — API (`api/`)

### Task 1: Schéma Prisma — Space/Node/Edge

**Files:**
- Modify: `api/prisma/schema.prisma`

- [ ] **Step 1: Remplacer le schéma existant**

Remplacer tout le contenu de `api/prisma/schema.prisma` par :

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum NodeType {
  OBJECTIF
  ETAPE
}

enum NodeStatus {
  active
  completed
}

model Space {
  id        String   @id @default(uuid()) @db.Uuid
  name      String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  nodes     Node[]
}

model Node {
  id          String     @id @default(uuid()) @db.Uuid
  spaceId     String     @db.Uuid
  type        NodeType
  title       String
  description String?
  status      NodeStatus @default(active)
  validatedAt DateTime?
  positionX   Float      @default(0)
  positionY   Float      @default(0)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  space         Space  @relation(fields: [spaceId], references: [id], onDelete: Cascade)
  outgoingEdges Edge[] @relation("EdgeSource")
  incomingEdges Edge[] @relation("EdgeTarget")

  @@index([spaceId])
  @@unique([spaceId, title])
}

model Edge {
  id           String   @id @default(uuid()) @db.Uuid
  sourceNodeId String   @db.Uuid
  targetNodeId String   @db.Uuid
  createdAt    DateTime @default(now())

  sourceNode Node @relation("EdgeSource", fields: [sourceNodeId], references: [id], onDelete: Cascade)
  targetNode Node @relation("EdgeTarget", fields: [targetNodeId], references: [id], onDelete: Cascade)

  @@unique([sourceNodeId, targetNodeId])
  @@index([sourceNodeId])
  @@index([targetNodeId])
}
```

- [ ] **Step 2: Générer la migration**

Run: `cd api && DATABASE_URL="${DATABASE_URL:-postgresql://prisma:prisma@localhost:5432/prisma?schema=public}" npx prisma migrate dev --name unified_node_graph`
Expected: la migration est créée sous `api/prisma/migrations/`, applique le drop de `Quest`/`Step` et la création de `Node`/`Edge`, et se termine par "Your database is now in sync with your schema."

- [ ] **Step 3: Générer le client Prisma**

Run: `cd api && npx prisma generate`
Expected: "Generated Prisma Client" sans erreur.

- [ ] **Step 4: Commit**

```bash
git add api/prisma/schema.prisma api/prisma/migrations
git commit -m "feat(api): remplace Quest/Step par un graphe unifié Node/Edge"
```

---

### Task 2: Seed du graphe unifié

**Files:**
- Modify: `api/prisma/seed.ts`
- Modify: `api/test/seed.e2e-spec.ts`

- [ ] **Step 1: Lire le test e2e existant pour connaître le format attendu**

Run: `cat api/test/seed.e2e-spec.ts`
Expected: ce test interroge probablement `prisma.quest`/`prisma.step` — il sera réécrit à l'étape suivante pour interroger `prisma.node`/`prisma.edge`.

- [ ] **Step 2: Réécrire `api/prisma/seed.ts`**

```typescript
import { NodeStatus, NodeType, Prisma, PrismaClient, Space } from '@prisma/client';

type SeedClient = Prisma.TransactionClient;

type SeedNode = {
  key: string;
  type: NodeType;
  title: string;
  status: NodeStatus;
  feedsInto: string[];
};

const graph: SeedNode[] = [
  { key: 'main', type: NodeType.OBJECTIF, title: 'Gagner beaucoup d’argent', status: NodeStatus.active, feedsInto: [] },
  { key: 'freelance', type: NodeType.OBJECTIF, title: 'Développer mon activité freelance', status: NodeStatus.active, feedsInto: ['main'] },
  { key: 'cloudbreak', type: NodeType.OBJECTIF, title: 'Lancer Cloudbreak', status: NodeStatus.active, feedsInto: ['main'] },
  { key: 'achat-revente', type: NodeType.OBJECTIF, title: 'Démarrer l’achat-revente de voitures', status: NodeStatus.active, feedsInto: ['main'] },

  { key: 'clarifier-offres', type: NodeType.ETAPE, title: 'Clarifier mes offres', status: NodeStatus.completed, feedsInto: ['freelance'] },
  { key: 'signer-clients', type: NodeType.ETAPE, title: 'Signer 3 clients récurrents', status: NodeStatus.active, feedsInto: ['freelance'] },
  { key: 'atteindre-5000', type: NodeType.ETAPE, title: 'Atteindre 5 000 € / mois', status: NodeStatus.active, feedsInto: ['freelance'] },

  { key: 'lancer-app', type: NodeType.ETAPE, title: 'Lancer l’application', status: NodeStatus.active, feedsInto: ['cloudbreak'] },
  { key: 'premiers-users', type: NodeType.ETAPE, title: 'Obtenir mes premiers utilisateurs', status: NodeStatus.active, feedsInto: ['cloudbreak'] },

  {
    key: 'epargne-3000',
    type: NodeType.ETAPE,
    title: 'Épargner 3 000 € de trésorerie',
    status: NodeStatus.active,
    // Une même étape alimente deux objectifs à la fois — c'est le cas
    // d'usage central de ce graphe unifié (cf. spec Décision 2).
    feedsInto: ['cloudbreak', 'achat-revente'],
  },
  { key: 'obd', type: NodeType.ETAPE, title: 'Acheter un lecteur OBD', status: NodeStatus.completed, feedsInto: ['achat-revente'] },
  { key: 'premier-vehicule', type: NodeType.ETAPE, title: 'Acheter le premier véhicule', status: NodeStatus.active, feedsInto: ['achat-revente'] },
];

async function findOrCreateSpace(client: SeedClient): Promise<Space> {
  return client.space.upsert({
    where: { name: 'Revenus' },
    update: {},
    create: { name: 'Revenus' },
  });
}

async function findOrCreateNode(client: SeedClient, spaceId: string, seedNode: SeedNode, index: number) {
  return client.node.upsert({
    where: { spaceId_title: { spaceId, title: seedNode.title } },
    update: { type: seedNode.type, status: seedNode.status },
    create: {
      spaceId,
      type: seedNode.type,
      title: seedNode.title,
      status: seedNode.status,
      positionX: (index % 4) * 320,
      positionY: Math.floor(index / 4) * 220,
    },
  });
}

export async function seedDatabase(prisma: PrismaClient) {
  await prisma.$transaction(async (transaction) => {
    await transaction.$executeRaw(
      Prisma.sql`SELECT pg_advisory_xact_lock(hashtextextended('quest:initial-objective-seed', 0));`,
    );
    const space = await findOrCreateSpace(transaction);

    const idByKey = new Map<string, string>();
    for (const [index, seedNode] of graph.entries()) {
      const created = await findOrCreateNode(transaction, space.id, seedNode, index);
      idByKey.set(seedNode.key, created.id);
    }

    for (const seedNode of graph) {
      const sourceNodeId = idByKey.get(seedNode.key)!;
      for (const targetKey of seedNode.feedsInto) {
        const targetNodeId = idByKey.get(targetKey)!;
        await transaction.edge.upsert({
          where: { sourceNodeId_targetNodeId: { sourceNodeId, targetNodeId } },
          update: {},
          create: { sourceNodeId, targetNodeId },
        });
      }
    }
  });
}

async function main() {
  const prisma = new PrismaClient();

  try {
    await seedDatabase(prisma);
  } finally {
    await prisma.$disconnect();
  }
}

if (require.main === module) {
  void main();
}
```

- [ ] **Step 3: Réécrire `api/test/seed.e2e-spec.ts`**

```typescript
import { PrismaClient } from '@prisma/client';
import { seedDatabase } from '../prisma/seed';

describe('seedDatabase', () => {
  const prisma = new PrismaClient();

  afterAll(async () => {
    await prisma.$disconnect();
  });

  it('is idempotent and produces the expected unified graph', async () => {
    await seedDatabase(prisma);
    await seedDatabase(prisma);

    const space = await prisma.space.findUniqueOrThrow({ where: { name: 'Revenus' } });
    const nodes = await prisma.node.findMany({ where: { spaceId: space.id } });
    const edges = await prisma.edge.findMany({ where: { sourceNode: { spaceId: space.id } } });

    expect(nodes).toHaveLength(12);
    expect(edges).toHaveLength(12);

    const sharedStep = nodes.find((node) => node.title === 'Épargner 3 000 € de trésorerie');
    expect(sharedStep).toBeDefined();
    const sharedStepEdges = edges.filter((edge) => edge.sourceNodeId === sharedStep!.id);
    expect(sharedStepEdges).toHaveLength(2);
  });
});
```

- [ ] **Step 4: Lancer le seed et le test**

Run: `cd api && npm run db:seed && npm run test:e2e -- seed.e2e-spec`
Expected: le seed s'exécute sans erreur, le test `seedDatabase` passe.

- [ ] **Step 5: Commit**

```bash
git add api/prisma/seed.ts api/test/seed.e2e-spec.ts
git commit -m "feat(api): seed du graphe unifié avec une étape partagée entre deux objectifs"
```

---

### Task 3: Module `nodes` — CRUD

**Files:**
- Create: `api/src/nodes/dto/create-node.dto.ts`
- Create: `api/src/nodes/dto/update-node.dto.ts`
- Create: `api/src/nodes/nodes.service.ts`
- Create: `api/src/nodes/nodes.controller.ts`
- Create: `api/src/nodes/nodes.module.ts`
- Create: `api/src/nodes/nodes.service.spec.ts`
- Modify: `api/src/app.module.ts`

- [ ] **Step 1: Write the failing unit test**

```typescript
// api/src/nodes/nodes.service.spec.ts
import { ConflictException, NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { NodeStatus, NodeType } from '@prisma/client';
import { PrismaService } from '../prisma/prisma.service';
import { NodesService } from './nodes.service';

describe('NodesService', () => {
  let service: NodesService;
  const prisma = {
    space: { findUnique: jest.fn() },
    node: { create: jest.fn(), update: jest.fn(), delete: jest.fn(), findMany: jest.fn(), findUnique: jest.fn() },
  };

  beforeEach(async () => {
    jest.resetAllMocks();
    const moduleRef = await Test.createTestingModule({
      providers: [NodesService, { provide: PrismaService, useValue: prisma }],
    }).compile();
    service = moduleRef.get(NodesService);
  });

  it('throws NotFoundException when the space does not exist', async () => {
    prisma.space.findUnique.mockResolvedValue(null);
    await expect(
      service.create('missing-space', { type: NodeType.ETAPE, title: 'Test' }),
    ).rejects.toThrow(NotFoundException);
  });

  it('creates a node scoped to its space', async () => {
    prisma.space.findUnique.mockResolvedValue({ id: 'space-1' });
    prisma.node.create.mockResolvedValue({ id: 'node-1', spaceId: 'space-1', type: NodeType.ETAPE, title: 'Test', status: NodeStatus.active });

    const result = await service.create('space-1', { type: NodeType.ETAPE, title: 'Test' });

    expect(prisma.node.create).toHaveBeenCalledWith({
      data: { spaceId: 'space-1', type: NodeType.ETAPE, title: 'Test', description: undefined, positionX: 0, positionY: 0 },
    });
    expect(result).toEqual(expect.objectContaining({ id: 'node-1' }));
  });

  it('wraps a duplicate title into a ConflictException', async () => {
    prisma.space.findUnique.mockResolvedValue({ id: 'space-1' });
    prisma.node.create.mockRejectedValue({ code: 'P2002', constructor: { name: 'PrismaClientKnownRequestError' } });
    await expect(service.create('space-1', { type: NodeType.ETAPE, title: 'Test' })).rejects.toThrow(ConflictException);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd api && npx jest src/nodes/nodes.service.spec.ts`
Expected: FAIL — `Cannot find module './nodes.service'`.

- [ ] **Step 3: Write DTOs**

```typescript
// api/src/nodes/dto/create-node.dto.ts
import { Transform, Type } from 'class-transformer';
import { IsEnum, IsNotEmpty, IsNumber, IsOptional, IsString } from 'class-validator';
import { NodeType } from '@prisma/client';

function trim(value: unknown) {
  return typeof value === 'string' ? value.trim() : value;
}

export class CreateNodeDto {
  @IsEnum(NodeType)
  type!: NodeType;

  @Transform(({ value }) => trim(value))
  @IsString()
  @IsNotEmpty()
  title!: string;

  @Transform(({ value }) => trim(value))
  @IsOptional()
  @IsString()
  description?: string;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  positionX?: number;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  positionY?: number;
}
```

```typescript
// api/src/nodes/dto/update-node.dto.ts
import { Transform, Type } from 'class-transformer';
import { IsEnum, IsNotEmpty, IsNumber, IsOptional, IsString } from 'class-validator';
import { NodeStatus } from '@prisma/client';
import { AtLeastOneOf } from '../../common/at-least-one-of.decorator';

function trim(value: unknown) {
  return typeof value === 'string' ? value.trim() : value;
}

export class UpdateNodeDto {
  @Transform(({ value }) => trim(value))
  @IsOptional()
  @IsString()
  @IsNotEmpty()
  title?: string;

  @Transform(({ value }) => trim(value))
  @IsOptional()
  @IsString()
  description?: string;

  @IsOptional()
  @IsEnum(NodeStatus)
  status?: NodeStatus;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  positionX?: number;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  positionY?: number;

  @AtLeastOneOf(['title', 'description', 'status', 'positionX', 'positionY'])
  private readonly _atLeastOneField?: never;
}
```

- [ ] **Step 4: Write `NodesService`**

```typescript
// api/src/nodes/nodes.service.ts
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import { PrismaService } from '../prisma/prisma.service';
import { CreateNodeDto } from './dto/create-node.dto';
import { UpdateNodeDto } from './dto/update-node.dto';

@Injectable()
export class NodesService {
  constructor(private readonly prisma: PrismaService) {}

  async findAllForSpace(spaceId: string) {
    const space = await this.prisma.space.findUnique({ where: { id: spaceId }, select: { id: true } });
    if (space === null) {
      throw new NotFoundException(`Space with id "${spaceId}" was not found.`);
    }
    return this.prisma.node.findMany({ where: { spaceId }, orderBy: { createdAt: 'asc' } });
  }

  async create(spaceId: string, createNodeDto: CreateNodeDto) {
    const space = await this.prisma.space.findUnique({ where: { id: spaceId }, select: { id: true } });
    if (space === null) {
      throw new NotFoundException(`Space with id "${spaceId}" was not found.`);
    }

    try {
      return await this.prisma.node.create({
        data: {
          spaceId,
          type: createNodeDto.type,
          title: createNodeDto.title,
          description: createNodeDto.description,
          positionX: createNodeDto.positionX ?? 0,
          positionY: createNodeDto.positionY ?? 0,
        },
      });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2002') {
        throw new ConflictException('A node with this title already exists in this space.');
      }
      throw error;
    }
  }

  async update(nodeId: string, updateNodeDto: UpdateNodeDto) {
    try {
      return await this.prisma.node.update({ where: { id: nodeId }, data: updateNodeDto });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError) {
        if (error.code === 'P2025') {
          throw new NotFoundException(`Node with id "${nodeId}" was not found.`);
        }
        if (error.code === 'P2002') {
          throw new ConflictException('A node with this title already exists in this space.');
        }
      }
      throw error;
    }
  }

  async remove(nodeId: string) {
    try {
      await this.prisma.node.delete({ where: { id: nodeId } });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2025') {
        throw new NotFoundException(`Node with id "${nodeId}" was not found.`);
      }
      throw error;
    }
  }

  async validate(nodeId: string) {
    const node = await this.prisma.node.findUnique({ where: { id: nodeId } });
    if (node === null) {
      throw new NotFoundException(`Node with id "${nodeId}" was not found.`);
    }
    if (node.type !== 'OBJECTIF') {
      throw new ConflictException('Only an OBJECTIF node can be validated.');
    }
    return this.prisma.node.update({
      where: { id: nodeId },
      data: { status: 'completed', validatedAt: new Date() },
    });
  }

  async progress(nodeId: string) {
    const node = await this.prisma.node.findUnique({ where: { id: nodeId } });
    if (node === null) {
      throw new NotFoundException(`Node with id "${nodeId}" was not found.`);
    }

    const spaceEdges = await this.prisma.edge.findMany({
      where: { targetNode: { spaceId: node.spaceId } },
      select: { sourceNodeId: true, targetNodeId: true },
    });
    const incomingBySource = new Map<string, string[]>();
    for (const edge of spaceEdges) {
      const sources = incomingBySource.get(edge.targetNodeId) ?? [];
      sources.push(edge.sourceNodeId);
      incomingBySource.set(edge.targetNodeId, sources);
    }

    const ancestorIds = new Set<string>();
    const pending = [...(incomingBySource.get(nodeId) ?? [])];
    while (pending.length > 0) {
      const currentId = pending.pop()!;
      if (ancestorIds.has(currentId)) continue;
      ancestorIds.add(currentId);
      pending.push(...(incomingBySource.get(currentId) ?? []));
    }

    if (ancestorIds.size === 0) {
      return { nodeId, totalAncestors: 0, completedAncestors: 0, percent: 0 };
    }

    const ancestors = await this.prisma.node.findMany({
      where: { id: { in: [...ancestorIds] } },
      select: { status: true },
    });
    const completedAncestors = ancestors.filter((ancestor) => ancestor.status === 'completed').length;

    return {
      nodeId,
      totalAncestors: ancestors.length,
      completedAncestors,
      percent: Math.round((completedAncestors / ancestors.length) * 100),
    };
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd api && npx jest src/nodes/nodes.service.spec.ts`
Expected: PASS (3 tests).

- [ ] **Step 6: Write controller and module**

```typescript
// api/src/nodes/nodes.controller.ts
import { Body, Controller, Delete, Get, HttpCode, Param, ParseUUIDPipe, Patch, Post } from '@nestjs/common';
import { CreateNodeDto } from './dto/create-node.dto';
import { UpdateNodeDto } from './dto/update-node.dto';
import { NodesService } from './nodes.service';

@Controller()
export class NodesController {
  constructor(private readonly nodesService: NodesService) {}

  @Get('spaces/:spaceId/nodes')
  findAllForSpace(@Param('spaceId', ParseUUIDPipe) spaceId: string) {
    return this.nodesService.findAllForSpace(spaceId);
  }

  @Post('spaces/:spaceId/nodes')
  create(@Param('spaceId', ParseUUIDPipe) spaceId: string, @Body() createNodeDto: CreateNodeDto) {
    return this.nodesService.create(spaceId, createNodeDto);
  }

  @Patch('nodes/:nodeId')
  update(@Param('nodeId', ParseUUIDPipe) nodeId: string, @Body() updateNodeDto: UpdateNodeDto) {
    return this.nodesService.update(nodeId, updateNodeDto);
  }

  @Delete('nodes/:nodeId')
  @HttpCode(204)
  async remove(@Param('nodeId', ParseUUIDPipe) nodeId: string) {
    await this.nodesService.remove(nodeId);
  }

  @Post('nodes/:nodeId/validate')
  validate(@Param('nodeId', ParseUUIDPipe) nodeId: string) {
    return this.nodesService.validate(nodeId);
  }

  @Get('nodes/:nodeId/progress')
  progress(@Param('nodeId', ParseUUIDPipe) nodeId: string) {
    return this.nodesService.progress(nodeId);
  }
}
```

```typescript
// api/src/nodes/nodes.module.ts
import { Module } from '@nestjs/common';
import { PrismaModule } from '../prisma/prisma.module';
import { NodesController } from './nodes.controller';
import { NodesService } from './nodes.service';

@Module({
  imports: [PrismaModule],
  controllers: [NodesController],
  providers: [NodesService],
})
export class NodesModule {}
```

- [ ] **Step 7: Wire the module into `AppModule`**

Modify `api/src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PrismaModule } from './prisma/prisma.module';
import { SpacesModule } from './spaces/spaces.module';
import { NodesModule } from './nodes/nodes.module';
import { EdgesModule } from './edges/edges.module';

@Module({
  imports: [PrismaModule, SpacesModule, NodesModule, EdgesModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

(`EdgesModule` n'existe pas encore — c'est normal, il est créé à la Task 4. Le build échouera jusque-là ; c'est attendu, ne pas lancer `npm run build` avant la fin de la Task 4.)

- [ ] **Step 8: Commit**

```bash
git add api/src/nodes api/src/app.module.ts
git commit -m "feat(api): ajoute le module nodes (CRUD, validation, progression)"
```

---

### Task 4: Module `edges` — création/suppression de liens

**Files:**
- Create: `api/src/edges/dto/create-edge.dto.ts`
- Create: `api/src/edges/edges.service.ts`
- Create: `api/src/edges/edges.controller.ts`
- Create: `api/src/edges/edges.module.ts`
- Create: `api/src/edges/edges.service.spec.ts`

- [ ] **Step 1: Write the failing unit test**

```typescript
// api/src/edges/edges.service.spec.ts
import { ConflictException, NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { PrismaService } from '../prisma/prisma.service';
import { EdgesService } from './edges.service';

describe('EdgesService', () => {
  let service: EdgesService;
  const prisma = {
    node: { findMany: jest.fn() },
    edge: { create: jest.fn(), delete: jest.fn(), findMany: jest.fn() },
  };

  beforeEach(async () => {
    jest.resetAllMocks();
    const moduleRef = await Test.createTestingModule({
      providers: [EdgesService, { provide: PrismaService, useValue: prisma }],
    }).compile();
    service = moduleRef.get(EdgesService);
  });

  it('rejects an edge whose nodes do not both exist', async () => {
    prisma.node.findMany.mockResolvedValue([{ id: 'a', spaceId: 'space-1' }]);
    await expect(service.create('space-1', { sourceNodeId: 'a', targetNodeId: 'missing' }))
      .rejects.toThrow(NotFoundException);
  });

  it('rejects a self-loop', async () => {
    prisma.node.findMany.mockResolvedValue([{ id: 'a', spaceId: 'space-1' }]);
    await expect(service.create('space-1', { sourceNodeId: 'a', targetNodeId: 'a' }))
      .rejects.toThrow(ConflictException);
  });

  it('rejects an edge that would create a cycle', async () => {
    prisma.node.findMany.mockResolvedValue([
      { id: 'a', spaceId: 'space-1' },
      { id: 'b', spaceId: 'space-1' },
    ]);
    // b -> a already exists; creating a -> b would close a cycle.
    prisma.edge.findMany.mockResolvedValue([{ sourceNodeId: 'b', targetNodeId: 'a' }]);
    await expect(service.create('space-1', { sourceNodeId: 'a', targetNodeId: 'b' }))
      .rejects.toThrow(ConflictException);
  });

  it('creates the edge when everything is valid', async () => {
    prisma.node.findMany.mockResolvedValue([
      { id: 'a', spaceId: 'space-1' },
      { id: 'b', spaceId: 'space-1' },
    ]);
    prisma.edge.findMany.mockResolvedValue([]);
    prisma.edge.create.mockResolvedValue({ id: 'edge-1', sourceNodeId: 'a', targetNodeId: 'b' });

    const result = await service.create('space-1', { sourceNodeId: 'a', targetNodeId: 'b' });

    expect(prisma.edge.create).toHaveBeenCalledWith({ data: { sourceNodeId: 'a', targetNodeId: 'b' } });
    expect(result).toEqual(expect.objectContaining({ id: 'edge-1' }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd api && npx jest src/edges/edges.service.spec.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the DTO**

```typescript
// api/src/edges/dto/create-edge.dto.ts
import { IsUUID } from 'class-validator';

export class CreateEdgeDto {
  @IsUUID()
  sourceNodeId!: string;

  @IsUUID()
  targetNodeId!: string;
}
```

- [ ] **Step 4: Write `EdgesService`**

```typescript
// api/src/edges/edges.service.ts
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import { PrismaService } from '../prisma/prisma.service';
import { CreateEdgeDto } from './dto/create-edge.dto';

@Injectable()
export class EdgesService {
  constructor(private readonly prisma: PrismaService) {}

  findAllForSpace(spaceId: string) {
    return this.prisma.edge.findMany({ where: { sourceNode: { spaceId } } });
  }

  async create(spaceId: string, createEdgeDto: CreateEdgeDto) {
    if (createEdgeDto.sourceNodeId === createEdgeDto.targetNodeId) {
      throw new ConflictException('A node cannot link to itself.');
    }

    const nodes = await this.prisma.node.findMany({
      where: { id: { in: [createEdgeDto.sourceNodeId, createEdgeDto.targetNodeId] } },
      select: { id: true, spaceId: true },
    });
    if (nodes.length !== 2 || nodes.some((node) => node.spaceId !== spaceId)) {
      throw new NotFoundException('Both nodes must exist and belong to this space.');
    }

    const wouldCycle = await this.createsCycle(spaceId, createEdgeDto.sourceNodeId, createEdgeDto.targetNodeId);
    if (wouldCycle) {
      throw new ConflictException('This link would create a cycle in the graph.');
    }

    try {
      return await this.prisma.edge.create({
        data: { sourceNodeId: createEdgeDto.sourceNodeId, targetNodeId: createEdgeDto.targetNodeId },
      });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2002') {
        throw new ConflictException('This link already exists.');
      }
      throw error;
    }
  }

  async remove(edgeId: string) {
    try {
      await this.prisma.edge.delete({ where: { id: edgeId } });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2025') {
        throw new NotFoundException(`Edge with id "${edgeId}" was not found.`);
      }
      throw error;
    }
  }

  // A new edge source -> target would create a cycle if target can already
  // reach source through existing edges (adding source -> target would then
  // close the loop). Node.progress() assumes a DAG, so this must be rejected
  // at write time rather than handled defensively at read time.
  private async createsCycle(spaceId: string, sourceNodeId: string, targetNodeId: string): Promise<boolean> {
    const edges = await this.prisma.edge.findMany({
      where: { sourceNode: { spaceId } },
      select: { sourceNodeId: true, targetNodeId: true },
    });
    const outgoingByNode = new Map<string, string[]>();
    for (const edge of edges) {
      const targets = outgoingByNode.get(edge.sourceNodeId) ?? [];
      targets.push(edge.targetNodeId);
      outgoingByNode.set(edge.sourceNodeId, targets);
    }

    const visited = new Set<string>();
    const pending = [targetNodeId];
    while (pending.length > 0) {
      const currentId = pending.pop()!;
      if (currentId === sourceNodeId) return true;
      if (visited.has(currentId)) continue;
      visited.add(currentId);
      pending.push(...(outgoingByNode.get(currentId) ?? []));
    }
    return false;
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd api && npx jest src/edges/edges.service.spec.ts`
Expected: PASS (4 tests).

- [ ] **Step 6: Write controller and module**

```typescript
// api/src/edges/edges.controller.ts
import { Body, Controller, Delete, Get, HttpCode, Param, ParseUUIDPipe, Post } from '@nestjs/common';
import { CreateEdgeDto } from './dto/create-edge.dto';
import { EdgesService } from './edges.service';

@Controller()
export class EdgesController {
  constructor(private readonly edgesService: EdgesService) {}

  @Get('spaces/:spaceId/edges')
  findAllForSpace(@Param('spaceId', ParseUUIDPipe) spaceId: string) {
    return this.edgesService.findAllForSpace(spaceId);
  }

  @Post('spaces/:spaceId/edges')
  create(@Param('spaceId', ParseUUIDPipe) spaceId: string, @Body() createEdgeDto: CreateEdgeDto) {
    return this.edgesService.create(spaceId, createEdgeDto);
  }

  @Delete('edges/:edgeId')
  @HttpCode(204)
  async remove(@Param('edgeId', ParseUUIDPipe) edgeId: string) {
    await this.edgesService.remove(edgeId);
  }
}
```

```typescript
// api/src/edges/edges.module.ts
import { Module } from '@nestjs/common';
import { PrismaModule } from '../prisma/prisma.module';
import { EdgesController } from './edges.controller';
import { EdgesService } from './edges.service';

@Module({
  imports: [PrismaModule],
  controllers: [EdgesController],
  providers: [EdgesService],
})
export class EdgesModule {}
```

- [ ] **Step 7: Build to confirm `AppModule` now resolves**

Run: `cd api && npm run build`
Expected: build succeeds (this closes out the `EdgesModule` forward reference from Task 3, Step 7).

- [ ] **Step 8: Commit**

```bash
git add api/src/edges
git commit -m "feat(api): ajoute le module edges (création/suppression de liens, garde anti-cycle)"
```

---

### Task 5: Suppression des anciens modules `quests`/`steps`/`spaces` (routes obsolètes)

**Files:**
- Delete: `api/src/quests/`
- Delete: `api/src/steps/`
- Modify: `api/src/spaces/spaces.controller.ts`
- Modify: `api/src/spaces/spaces.service.ts`
- Delete: `api/src/spaces/dto/create-quest.dto.ts`
- Delete: `api/test/quests-steps.e2e-spec.ts`
- Create: `api/test/nodes-edges.e2e-spec.ts`

- [ ] **Step 1: Supprimer les modules obsolètes**

Run: `cd api && rm -rf src/quests src/steps src/spaces/dto/create-quest.dto.ts test/quests-steps.e2e-spec.ts`
Expected: aucune sortie ; ces chemins n'existent plus.

- [ ] **Step 2: Réécrire `SpacesService`**

```typescript
// api/src/spaces/spaces.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class SpacesService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.space.findMany({ orderBy: { name: 'asc' } });
  }

  async findGraph(spaceId: string) {
    const space = await this.prisma.space.findUnique({
      where: { id: spaceId },
      include: {
        nodes: { orderBy: { createdAt: 'asc' } },
      },
    });
    if (space === null) {
      throw new NotFoundException(`Space with id "${spaceId}" was not found.`);
    }

    const edges = await this.prisma.edge.findMany({ where: { sourceNode: { spaceId } } });

    return { id: space.id, name: space.name, nodes: space.nodes, edges };
  }
}
```

- [ ] **Step 3: Réécrire `SpacesController`**

```typescript
// api/src/spaces/spaces.controller.ts
import { Controller, Get, Param, ParseUUIDPipe } from '@nestjs/common';
import { SpacesService } from './spaces.service';

@Controller('spaces')
export class SpacesController {
  constructor(private readonly spacesService: SpacesService) {}

  @Get()
  findAll() {
    return this.spacesService.findAll();
  }

  @Get(':spaceId/graph')
  findGraph(@Param('spaceId', ParseUUIDPipe) spaceId: string) {
    return this.spacesService.findGraph(spaceId);
  }
}
```

- [ ] **Step 4: Retirer les imports morts de `app.module.ts`**

Vérifier que `api/src/app.module.ts` (déjà réécrit Task 3 Step 7) ne référence plus `QuestsModule`/`StepsModule` — c'est déjà le cas, ne rien changer ici.

- [ ] **Step 5: Écrire le test e2e du parcours complet**

```typescript
// api/test/nodes-edges.e2e-spec.ts
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Nodes & Edges (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;
  let spaceId: string;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ transform: true, whitelist: true, forbidNonWhitelisted: true }));
    await app.init();
    prisma = moduleRef.get(PrismaService);

    const space = await prisma.space.upsert({
      where: { name: 'e2e-nodes-edges' },
      update: {},
      create: { name: 'e2e-nodes-edges' },
    });
    spaceId = space.id;
  });

  afterAll(async () => {
    await prisma.node.deleteMany({ where: { spaceId } });
    await prisma.space.delete({ where: { id: spaceId } });
    await app.close();
  });

  it('creates two nodes, links them, and computes progress', async () => {
    const objectif = await request(app.getHttpServer())
      .post(`/spaces/${spaceId}/nodes`)
      .send({ type: 'OBJECTIF', title: 'Objectif e2e' })
      .expect(201);

    const etape = await request(app.getHttpServer())
      .post(`/spaces/${spaceId}/nodes`)
      .send({ type: 'ETAPE', title: 'Étape e2e' })
      .expect(201);

    await request(app.getHttpServer())
      .post(`/spaces/${spaceId}/edges`)
      .send({ sourceNodeId: etape.body.id, targetNodeId: objectif.body.id })
      .expect(201);

    let progress = await request(app.getHttpServer())
      .get(`/nodes/${objectif.body.id}/progress`)
      .expect(200);
    expect(progress.body).toEqual({ nodeId: objectif.body.id, totalAncestors: 1, completedAncestors: 0, percent: 0 });

    await request(app.getHttpServer())
      .patch(`/nodes/${etape.body.id}`)
      .send({ status: 'completed' })
      .expect(200);

    progress = await request(app.getHttpServer())
      .get(`/nodes/${objectif.body.id}/progress`)
      .expect(200);
    expect(progress.body).toEqual({ nodeId: objectif.body.id, totalAncestors: 1, completedAncestors: 1, percent: 100 });

    await request(app.getHttpServer()).post(`/nodes/${objectif.body.id}/validate`).expect(201);

    const graph = await request(app.getHttpServer()).get(`/spaces/${spaceId}/graph`).expect(200);
    const validated = graph.body.nodes.find((node: { id: string }) => node.id === objectif.body.id);
    expect(validated.status).toBe('completed');
    expect(validated.validatedAt).not.toBeNull();
  });

  it('deleting a node only removes its edges, not its former neighbors', async () => {
    const a = await request(app.getHttpServer()).post(`/spaces/${spaceId}/nodes`).send({ type: 'ETAPE', title: 'A' }).expect(201);
    const b = await request(app.getHttpServer()).post(`/spaces/${spaceId}/nodes`).send({ type: 'ETAPE', title: 'B' }).expect(201);
    await request(app.getHttpServer()).post(`/spaces/${spaceId}/edges`).send({ sourceNodeId: a.body.id, targetNodeId: b.body.id }).expect(201);

    await request(app.getHttpServer()).delete(`/nodes/${a.body.id}`).expect(204);

    const remaining = await request(app.getHttpServer()).get(`/spaces/${spaceId}/nodes`).expect(200);
    expect(remaining.body.map((node: { id: string }) => node.id)).toContain(b.body.id);
    const edges = await request(app.getHttpServer()).get(`/spaces/${spaceId}/edges`).expect(200);
    expect(edges.body.some((edge: { sourceNodeId: string }) => edge.sourceNodeId === a.body.id)).toBe(false);
  });
});
```

- [ ] **Step 6: Run all API tests**

Run: `cd api && npm run test && npm run test:e2e`
Expected: toutes les suites passent ; aucune référence résiduelle à `quest`/`step`.

- [ ] **Step 7: Commit**

```bash
git add -A api/src/quests api/src/steps api/src/spaces api/test
git commit -m "refactor(api): supprime les modules Quest/Step obsolètes, expose /spaces/:id/graph"
```

---

### Task 6: Mettre à jour `api/CLAUDE.md`

**Files:**
- Modify: `api/CLAUDE.md`

- [ ] **Step 1: Remplacer la règle de suppression en cascade**

Dans `api/CLAUDE.md`, remplacer :

```
- Un module NestJS par domaine : `spaces`, `quests`, `steps`.
- Controllers fins ; services responsables de la logique ; DTOs pour validation ; Prisma uniquement pour la persistence.
- Une suppression de Step supprime explicitement ses descendants dans une transaction.
- Chaque story API crée `docs/story-<id>-<slug>.md` et sa recette manuelle racine.
```

par :

```
- Un module NestJS par domaine : `spaces`, `nodes`, `edges`.
- Controllers fins ; services responsables de la logique ; DTOs pour validation ; Prisma uniquement pour la persistence.
- Une suppression de Node ne supprime que ses Edges (`onDelete: Cascade` côté `Edge`, jamais côté `Node`) : les nœuds voisins restent dans le graphe, non reconnectés, à reconnecter manuellement depuis l'UI.
- Toute création d'Edge doit être rejetée si elle fermerait un cycle (le calcul de progression suppose un DAG).
- Chaque story API crée `docs/story-<id>-<slug>.md` et sa recette manuelle racine.
```

- [ ] **Step 2: Commit**

```bash
git add api/CLAUDE.md
git commit -m "docs(api): met à jour la règle de suppression pour le graphe Node/Edge"
```

---

## Phase B — Web (`web/`)

### Task 7: Types et client API

**Files:**
- Modify: `web/src/lib/map/graph.ts`
- Modify: `web/src/lib/map/fallback.ts`
- Modify: `web/src/lib/api/client.ts`
- Modify: `web/src/lib/api/client.test.ts`
- Delete: `web/src/lib/map/placement.ts`
- Delete: `web/src/lib/map/placement.test.ts`

- [ ] **Step 1: Lire le test client existant pour connaître le style attendu**

Run: `cat web/src/lib/api/client.test.ts`
Expected: confirme le pattern (mock de `fetch`, assertions sur l'URL/verbe) à reproduire pour les nouvelles méthodes.

- [ ] **Step 2: Réécrire `web/src/lib/map/graph.ts`**

Le calcul de layout complexe (`branchLayout`, `projectPosition`, `findOpenPosition`) disparaît : la position de chaque Card vient désormais de la base (`positionX`/`positionY`), plus besoin de la recalculer côté front. Seule la position par défaut d'un nœud tout juste créé reste à déterminer côté front, dans le hook (Task 9), pas ici.

```typescript
export type NodeType = 'OBJECTIF' | 'ETAPE';
export type NodeStatus = 'active' | 'completed';

export interface SpaceNode {
  id: string;
  type: NodeType;
  title: string;
  description?: string | null;
  status: NodeStatus;
  validatedAt?: string | null;
  positionX: number;
  positionY: number;
}

export interface SpaceEdge {
  id: string;
  sourceNodeId: string;
  targetNodeId: string;
}

export interface SpaceGraph {
  id: string;
  name: string;
  nodes: SpaceNode[];
  edges: SpaceEdge[];
}
```

- [ ] **Step 3: Réécrire `web/src/lib/map/fallback.ts`**

```typescript
import type { SpaceGraph } from './graph';

export const fallbackSpace: SpaceGraph = {
  id: 'local-atlas',
  name: 'L’atlas d’Alex',
  nodes: [
    { id: 'main', type: 'OBJECTIF', title: 'Gagner beaucoup d’argent', status: 'active', positionX: 640, positionY: 80 },
    { id: 'freelance', type: 'OBJECTIF', title: 'Développer mon activité freelance', status: 'active', positionX: 160, positionY: 360 },
    { id: 'signer-clients', type: 'ETAPE', title: 'Signer 3 clients récurrents', status: 'active', positionX: 160, positionY: 600 },
    { id: 'cloudbreak', type: 'OBJECTIF', title: 'Lancer Cloudbreak', status: 'active', positionX: 640, positionY: 360 },
    { id: 'epargne-3000', type: 'ETAPE', title: 'Épargner 3 000 € de trésorerie', status: 'active', positionX: 900, positionY: 600 },
    { id: 'achat-revente', type: 'OBJECTIF', title: 'Démarrer l’achat-revente de voitures', status: 'active', positionX: 1120, positionY: 360 },
  ],
  edges: [
    { id: 'e-freelance-main', sourceNodeId: 'freelance', targetNodeId: 'main' },
    { id: 'e-cloudbreak-main', sourceNodeId: 'cloudbreak', targetNodeId: 'main' },
    { id: 'e-achat-main', sourceNodeId: 'achat-revente', targetNodeId: 'main' },
    { id: 'e-signer-freelance', sourceNodeId: 'signer-clients', targetNodeId: 'freelance' },
    { id: 'e-epargne-cloudbreak', sourceNodeId: 'epargne-3000', targetNodeId: 'cloudbreak' },
    { id: 'e-epargne-achat', sourceNodeId: 'epargne-3000', targetNodeId: 'achat-revente' },
  ],
};
```

- [ ] **Step 4: Réécrire `web/src/lib/api/client.ts`**

```typescript
import type { NodeType, SpaceGraph, SpaceNode } from '@/lib/map/graph';

export type CreateNodeInput = {
  type: NodeType;
  title: string;
  description?: string;
  positionX?: number;
  positionY?: number;
};

export type UpdateNodeInput = {
  title?: string;
  description?: string;
  status?: SpaceNode['status'];
  positionX?: number;
  positionY?: number;
};

export type CreateEdgeInput = {
  sourceNodeId: string;
  targetNodeId: string;
};

export class QuestApiClient {
  constructor(private readonly baseUrl: string) {}

  async loadFirstSpace(): Promise<SpaceGraph> {
    const spaces = await this.getJson<Array<Pick<SpaceGraph, 'id' | 'name'>>>('/spaces');
    if (spaces.length === 0) {
      throw new Error('Aucun espace disponible');
    }
    return this.getJson<SpaceGraph>(`/spaces/${spaces[0]!.id}/graph`);
  }

  createNode(spaceId: string, input: CreateNodeInput): Promise<SpaceNode> {
    return this.sendJson<SpaceNode>(`/spaces/${spaceId}/nodes`, input);
  }

  updateNode(nodeId: string, input: UpdateNodeInput): Promise<SpaceNode> {
    return this.patchJson<SpaceNode>(`/nodes/${nodeId}`, input);
  }

  async deleteNode(nodeId: string): Promise<void> {
    await this.deleteRequest(`/nodes/${nodeId}`);
  }

  validateObjectif(nodeId: string): Promise<SpaceNode> {
    return this.sendJson<SpaceNode>(`/nodes/${nodeId}/validate`, {});
  }

  progressFor(nodeId: string): Promise<{ percent: number; totalAncestors: number; completedAncestors: number }> {
    return this.getJson(`/nodes/${nodeId}/progress`);
  }

  createEdge(spaceId: string, input: CreateEdgeInput): Promise<{ id: string; sourceNodeId: string; targetNodeId: string }> {
    return this.sendJson(`/spaces/${spaceId}/edges`, input);
  }

  async deleteEdge(edgeId: string): Promise<void> {
    await this.deleteRequest(`/edges/${edgeId}`);
  }

  private async getJson<T>(path: string): Promise<T> {
    const response = await fetch(`${this.baseUrl.replace(/\/$/, '')}${path}`, { cache: 'no-store' });
    if (!response.ok) {
      throw new Error(`API indisponible (${response.status})`);
    }
    return response.json() as Promise<T>;
  }

  private async sendJson<T>(path: string, body: object): Promise<T> {
    const response = await fetch(`${this.baseUrl.replace(/\/$/, '')}${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    if (!response.ok) {
      throw new Error(`Création impossible (${response.status})`);
    }
    return response.json() as Promise<T>;
  }

  private async patchJson<T>(path: string, body: object): Promise<T> {
    const response = await fetch(`${this.baseUrl.replace(/\/$/, '')}${path}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    if (!response.ok) {
      throw new Error(`Mise à jour impossible (${response.status})`);
    }
    return response.json() as Promise<T>;
  }

  private async deleteRequest(path: string): Promise<void> {
    const response = await fetch(`${this.baseUrl.replace(/\/$/, '')}${path}`, { method: 'DELETE' });
    if (!response.ok) {
      throw new Error(`Suppression impossible (${response.status})`);
    }
  }
}
```

- [ ] **Step 5: Réécrire `web/src/lib/api/client.test.ts`**

```typescript
import { describe, expect, it, vi } from 'vitest';
import { QuestApiClient } from './client';

describe('QuestApiClient', () => {
  it('creates a node against the space nodes endpoint', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true, json: async () => ({ id: 'node-1' }) });
    vi.stubGlobal('fetch', fetchMock);

    const client = new QuestApiClient('http://api.test');
    await client.createNode('space-1', { type: 'ETAPE', title: 'Test' });

    expect(fetchMock).toHaveBeenCalledWith(
      'http://api.test/spaces/space-1/nodes',
      expect.objectContaining({ method: 'POST' }),
    );
  });

  it('creates an edge against the space edges endpoint', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true, json: async () => ({ id: 'edge-1' }) });
    vi.stubGlobal('fetch', fetchMock);

    const client = new QuestApiClient('http://api.test');
    await client.createEdge('space-1', { sourceNodeId: 'a', targetNodeId: 'b' });

    expect(fetchMock).toHaveBeenCalledWith(
      'http://api.test/spaces/space-1/edges',
      expect.objectContaining({ method: 'POST' }),
    );
  });

  it('deletes an edge with a DELETE request', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true });
    vi.stubGlobal('fetch', fetchMock);

    const client = new QuestApiClient('http://api.test');
    await client.deleteEdge('edge-1');

    expect(fetchMock).toHaveBeenCalledWith(
      'http://api.test/edges/edge-1',
      expect.objectContaining({ method: 'DELETE' }),
    );
  });
});
```

- [ ] **Step 6: Supprimer le module de placement devenu inutile**

Run: `cd web && rm -f src/lib/map/placement.ts src/lib/map/placement.test.ts`

- [ ] **Step 7: Run tests**

Run: `cd web && npm run test`
Expected: FAIL attendu à ce stade sur `use-quest-map`/`quest-map`/`use-branch-drag` (pas encore migrés — Tasks 8-10) ; les nouveaux tests `client.test.ts` doivent, eux, passer. Confirmer qu'ils passent en isolant : `npx vitest run src/lib/api/client.test.ts`.

- [ ] **Step 8: Commit**

```bash
git add web/src/lib/map/graph.ts web/src/lib/map/fallback.ts web/src/lib/api/client.ts web/src/lib/api/client.test.ts
git rm web/src/lib/map/placement.ts web/src/lib/map/placement.test.ts
git commit -m "refactor(web): remplace les types Quest/Step par SpaceNode/SpaceEdge et le client API associé"
```

---

### Task 8: Supprimer l'interaction "branche + menu" devenue obsolète

**Files:**
- Delete: `web/src/components/branch-menu.tsx`
- Delete: `web/src/components/node-anchor.tsx`
- Delete: `web/src/hooks/use-branch-drag.ts`
- Delete: `web/src/hooks/use-branch-drag.test.ts`
- Delete: `web/src/lib/map/edge-geometry.ts`
- Delete: `web/src/lib/map/edge-geometry.test.ts`

Cette interaction (glisser depuis une ancre pour ouvrir un menu de choix) reposait sur la contrainte "un step = un seul parent". Le graphe DAG la remplace par la création de lien native de React Flow (glisser d'un Handle à un autre, Task 10) et un bouton "+" de création de nœud dans le HUD (Task 11) — plus simple, et c'est le point précis où l'ancienne UI était fragile ("comme actuellement" dans la demande d'origine).

- [ ] **Step 1: Confirmer qu'aucun autre fichier n'importe ces modules avant suppression**

Run: `cd web && grep -rl "branch-menu\|node-anchor\|use-branch-drag\|edge-geometry" src --include="*.tsx" --include="*.ts" | grep -v "\.test\."`
Expected: seuls `quest-map.tsx` et `quest-edge.tsx` apparaissent — ils sont réécrits aux Tasks 9-10, donc ces références disparaîtront avec eux. Si un autre fichier apparaît, l'ouvrir et vérifier avant de continuer.

- [ ] **Step 2: Supprimer les fichiers**

```bash
cd web
rm -f src/components/branch-menu.tsx src/components/node-anchor.tsx \
      src/hooks/use-branch-drag.ts src/hooks/use-branch-drag.test.ts \
      src/lib/map/edge-geometry.ts src/lib/map/edge-geometry.test.ts
```

- [ ] **Step 3: Commit**

```bash
git add -A web/src/components/branch-menu.tsx web/src/components/node-anchor.tsx web/src/hooks/use-branch-drag.ts web/src/hooks/use-branch-drag.test.ts web/src/lib/map/edge-geometry.ts web/src/lib/map/edge-geometry.test.ts
git commit -m "refactor(web): retire l'interaction ancre+menu, remplacée par le drag & connect natif React Flow"
```

(Le build restera cassé — `quest-map.tsx`/`quest-edge.tsx`/`use-quest-map.ts` référencent encore ces fichiers — jusqu'à la fin de la Task 10. C'est attendu.)

---

### Task 9: Hook `use-space-map`

**Files:**
- Create: `web/src/hooks/use-space-map.ts`
- Create: `web/src/hooks/use-space-map.test.ts`
- Delete: `web/src/hooks/use-quest-map.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// web/src/hooks/use-space-map.test.ts
import { act, renderHook, waitFor } from '@testing-library/react';
import { describe, expect, it, vi } from 'vitest';
import { useSpaceMap } from './use-space-map';

vi.mock('@/lib/api/client', () => {
  const graph = {
    id: 'space-1',
    name: 'Test Space',
    nodes: [{ id: 'n1', type: 'OBJECTIF', title: 'Objectif', status: 'active', positionX: 0, positionY: 0 }],
    edges: [],
  };
  return {
    QuestApiClient: vi.fn().mockImplementation(() => ({
      loadFirstSpace: vi.fn().mockResolvedValue(graph),
      createNode: vi.fn().mockResolvedValue({ id: 'n2', type: 'ETAPE', title: 'Étape', status: 'active', positionX: 100, positionY: 100 }),
      updateNode: vi.fn().mockResolvedValue({ id: 'n1', positionX: 50, positionY: 50 }),
      createEdge: vi.fn().mockResolvedValue({ id: 'e1', sourceNodeId: 'n2', targetNodeId: 'n1' }),
      deleteEdge: vi.fn().mockResolvedValue(undefined),
    })),
  };
});

describe('useSpaceMap', () => {
  it('loads the space graph from the API', async () => {
    const { result } = renderHook(() => useSpaceMap());
    await waitFor(() => expect(result.current.source).toBe('api'));
    expect(result.current.graph.nodes).toHaveLength(1);
  });

  it('creates a node and adds it to the graph', async () => {
    const { result } = renderHook(() => useSpaceMap());
    await waitFor(() => expect(result.current.source).toBe('api'));

    await act(async () => {
      await result.current.createNode({ type: 'ETAPE', title: 'Étape', positionX: 100, positionY: 100 });
    });

    expect(result.current.graph.nodes.map((node) => node.id)).toContain('n2');
  });

  it('creates an edge and adds it to the graph', async () => {
    const { result } = renderHook(() => useSpaceMap());
    await waitFor(() => expect(result.current.source).toBe('api'));

    await act(async () => {
      await result.current.linkNodes('n2', 'n1');
    });

    expect(result.current.graph.edges.map((edge) => edge.id)).toContain('e1');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && npx vitest run src/hooks/use-space-map.test.ts`
Expected: FAIL — `Cannot find module './use-space-map'`.

- [ ] **Step 3: Implement the hook**

```typescript
// web/src/hooks/use-space-map.ts
'use client';

import { useCallback, useEffect, useState } from 'react';

import { QuestApiClient, type CreateNodeInput } from '@/lib/api/client';
import { fallbackSpace } from '@/lib/map/fallback';
import type { SpaceGraph, SpaceNode } from '@/lib/map/graph';

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001';

export function useSpaceMap() {
  const [graph, setGraph] = useState<SpaceGraph>(fallbackSpace);
  const [source, setSource] = useState<'local' | 'api'>('local');
  const [selectedNodeId, setSelectedNodeId] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let active = true;
    new QuestApiClient(API_URL)
      .loadFirstSpace()
      .then((apiGraph) => {
        if (!active) return;
        setGraph(apiGraph);
        setSelectedNodeId(apiGraph.nodes[0]?.id ?? null);
        setSource('api');
      })
      .catch(() => {
        // The interactive demo remains useful before the Nest API is running.
      });
    return () => {
      active = false;
    };
  }, []);

  const createNode = useCallback(async (input: CreateNodeInput): Promise<SpaceNode | null> => {
    const title = input.title.trim();
    if (!title) return null;

    if (source === 'api') {
      try {
        const node = await new QuestApiClient(API_URL).createNode(graph.id, { ...input, title });
        setGraph((current) => ({ ...current, nodes: [...current.nodes, node] }));
        setSelectedNodeId(node.id);
        setError(null);
        return node;
      } catch {
        setError('Le nœud n’a pas pu être créé.');
        return null;
      }
    }

    const node: SpaceNode = { id: `local-${Date.now()}`, type: input.type, title, status: 'active', positionX: input.positionX ?? 0, positionY: input.positionY ?? 0 };
    setGraph((current) => ({ ...current, nodes: [...current.nodes, node] }));
    setSelectedNodeId(node.id);
    return node;
  }, [graph.id, source]);

  const updateNodePosition = useCallback(async (nodeId: string, positionX: number, positionY: number) => {
    setGraph((current) => ({
      ...current,
      nodes: current.nodes.map((node) => node.id === nodeId ? { ...node, positionX, positionY } : node),
    }));
    if (source === 'api') {
      try {
        await new QuestApiClient(API_URL).updateNode(nodeId, { positionX, positionY });
      } catch {
        setError('La position n’a pas pu être enregistrée.');
      }
    }
  }, [source]);

  const updateNodeStatus = useCallback(async (nodeId: string, status: SpaceNode['status']) => {
    setGraph((current) => ({
      ...current,
      nodes: current.nodes.map((node) => node.id === nodeId ? { ...node, status } : node),
    }));
    if (source === 'api') {
      try {
        await new QuestApiClient(API_URL).updateNode(nodeId, { status });
      } catch {
        setError('Le statut n’a pas pu être enregistré.');
      }
    }
  }, [source]);

  const validateObjectif = useCallback(async (nodeId: string) => {
    if (source !== 'api') {
      setError('La validation nécessite l’API.');
      return;
    }
    try {
      const node = await new QuestApiClient(API_URL).validateObjectif(nodeId);
      setGraph((current) => ({
        ...current,
        nodes: current.nodes.map((existing) => existing.id === nodeId ? node : existing),
      }));
      setError(null);
    } catch {
      setError('La validation a échoué.');
    }
  }, [source]);

  const linkNodes = useCallback(async (sourceNodeId: string, targetNodeId: string) => {
    if (source !== 'api') {
      setError('La création de lien nécessite l’API.');
      return;
    }
    try {
      const edge = await new QuestApiClient(API_URL).createEdge(graph.id, { sourceNodeId, targetNodeId });
      setGraph((current) => ({ ...current, edges: [...current.edges, edge] }));
      setError(null);
    } catch {
      setError('Le lien n’a pas pu être créé.');
    }
  }, [graph.id, source]);

  const unlinkEdge = useCallback(async (edgeId: string) => {
    setGraph((current) => ({ ...current, edges: current.edges.filter((edge) => edge.id !== edgeId) }));
    if (source === 'api') {
      try {
        await new QuestApiClient(API_URL).deleteEdge(edgeId);
      } catch {
        setError('La suppression du lien a échoué.');
      }
    }
  }, [source]);

  const removeNode = useCallback(async (nodeId: string) => {
    setGraph((current) => ({
      ...current,
      nodes: current.nodes.filter((node) => node.id !== nodeId),
      edges: current.edges.filter((edge) => edge.sourceNodeId !== nodeId && edge.targetNodeId !== nodeId),
    }));
    setSelectedNodeId((current) => current === nodeId ? null : current);
    if (source === 'api') {
      try {
        await new QuestApiClient(API_URL).deleteNode(nodeId);
      } catch {
        setError('La suppression a échoué.');
      }
    }
  }, [source]);

  return {
    graph,
    source,
    error,
    selectedNodeId,
    selectNode: setSelectedNodeId,
    createNode,
    updateNodePosition,
    updateNodeStatus,
    validateObjectif,
    linkNodes,
    unlinkEdge,
    removeNode,
  };
}
```

- [ ] **Step 4: Supprimer l'ancien hook**

Run: `cd web && rm -f src/hooks/use-quest-map.ts`

- [ ] **Step 5: Run test to verify it passes**

Run: `cd web && npx vitest run src/hooks/use-space-map.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 6: Commit**

```bash
git add web/src/hooks/use-space-map.ts web/src/hooks/use-space-map.test.ts
git rm web/src/hooks/use-quest-map.ts
git commit -m "feat(web): remplace use-quest-map par use-space-map (nœuds/liens, statut, validation)"
```

---

### Task 10: Canvas `SpaceCanvas` — drag & drop, connexions, HUD

> **Modèle recommandé : ne pas utiliser `haiku` pour cette tâche** (cf. section Orchestration). C'est le cœur de la régression UX que ce plan corrige — le drag & drop, le suivi des liens et le HUD non bloquant doivent être vérifiés avec attention.

**Files:**
- Create: `web/src/components/space-canvas.tsx`
- Modify: `web/src/components/quest-edge.tsx`
- Modify: `web/src/app/page.tsx`
- Delete: `web/src/components/quest-map.tsx`

- [ ] **Step 1: Adapter `quest-edge.tsx` au nouveau modèle**

Le tracé de courbe reste identique ; seule la clé qui détermine la taille du rectangle change (`isObjective` → `type === 'OBJECTIF'`). Remplacer dans `web/src/components/quest-edge.tsx` :

```typescript
function nodeRect(node: ReturnType<typeof useInternalNode>): Rect | null {
  if (!node) return null;
  const isObjectif = (node.data as { type?: string }).type === 'OBJECTIF';
  const size = isObjectif ? OBJECTIVE_SIZE : STEP_SIZE;
  return {
    x: node.internals.positionAbsolute.x,
    y: node.internals.positionAbsolute.y,
    width: size.width,
    height: size.height,
  };
}
```

(le reste du fichier — imports, `rectCenter`, le composant `QuestEdge` — ne change pas.)

- [ ] **Step 2: Écrire `space-canvas.tsx`**

```tsx
// web/src/components/space-canvas.tsx
'use client';

import { useState, type FormEvent } from 'react';
import { useTranslation } from 'react-i18next';
import {
  Background,
  Controls,
  Handle,
  Position,
  ReactFlow,
  ReactFlowProvider,
  type Connection,
  type Edge,
  type EdgeChange,
  type Node,
  type NodeChange,
  type NodeProps,
  applyEdgeChanges,
  applyNodeChanges,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';

import { useSpaceMap } from '@/hooks/use-space-map';
import { useMediaQuery } from '@/hooks/use-media-query';
import { useTheme, THEME_CLASS } from '@/hooks/use-theme';
import { useLocale } from '@/hooks/use-locale';
import { QuestEdge } from '@/components/quest-edge';
import { ThemeSwitcher } from '@/components/theme-switcher';
import { LanguageToggle } from '@/components/language-toggle';
import type { NodeType, SpaceNode } from '@/lib/map/graph';

type SpaceFlowNode = Node<SpaceNode & Record<string, unknown>>;

function SpaceNodeCard({ data, selected }: NodeProps<SpaceFlowNode>) {
  const { t } = useTranslation();
  const isObjectif = data.type === 'OBJECTIF';
  return (
    <div
      className={`quest-node status-${data.status} ${isObjectif ? 'is-objective' : ''} ${selected ? 'is-selected' : ''}`}
      tabIndex={0}
      role="button"
      aria-label={data.title}
    >
      <Handle id="target" type="target" position={Position.Left} className="node-handle" />
      <Handle id="source" type="source" position={Position.Right} className="node-handle" />
      <span className="node-status">
        {isObjectif ? t('node.objectiveEyebrow') : data.status === 'completed' ? t('node.statusDone') : t('node.statusActive')}
      </span>
      <strong>{data.title}</strong>
    </div>
  );
}

const nodeTypes = { spaceNode: SpaceNodeCard };
const edgeTypes = { quest: QuestEdge };

export function SpaceCanvas() {
  return (
    <ReactFlowProvider>
      <SpaceCanvasInner />
    </ReactFlowProvider>
  );
}

function SpaceCanvasInner() {
  const {
    graph, source, error, selectedNodeId, selectNode,
    createNode, updateNodePosition, updateNodeStatus, validateObjectif, linkNodes, unlinkEdge, removeNode,
  } = useSpaceMap();
  const { themeId, setThemeId } = useTheme();
  const { locale, setLocale } = useLocale();
  const { t } = useTranslation();
  const isMobile = useMediaQuery('(max-width: 760px)');
  const [newNodeType, setNewNodeType] = useState<NodeType>('ETAPE');
  const [newNodeTitle, setNewNodeTitle] = useState('');
  const [isCreating, setIsCreating] = useState(false);

  const flowNodes: SpaceFlowNode[] = graph.nodes.map((node) => ({
    id: node.id,
    type: 'spaceNode',
    position: { x: node.positionX, y: node.positionY },
    selected: node.id === selectedNodeId,
    data: node,
  }));
  const flowEdges: Edge[] = graph.edges.map((edge) => ({
    id: edge.id,
    source: edge.sourceNodeId,
    target: edge.targetNodeId,
    type: 'quest',
    style: { stroke: 'var(--q-conn)', strokeWidth: 1.4, strokeDasharray: '7 9', opacity: 0.8 },
  }));

  function handleNodesChange(changes: NodeChange<SpaceFlowNode>[]) {
    for (const change of changes) {
      if (change.type === 'position' && change.dragging === false && change.position) {
        void updateNodePosition(change.id, change.position.x, change.position.y);
      }
    }
    applyNodeChanges(changes, flowNodes);
  }

  function handleEdgesChange(changes: EdgeChange<Edge>[]) {
    for (const change of changes) {
      if (change.type === 'remove') {
        void unlinkEdge(change.id);
      }
    }
    applyEdgeChanges(changes, flowEdges);
  }

  function handleConnect(connection: Connection) {
    if (!connection.source || !connection.target) return;
    void linkNodes(connection.source, connection.target);
  }

  async function submitCreate(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setIsCreating(true);
    const centerX = 400 + Math.round(Math.random() * 200);
    const centerY = 300 + Math.round(Math.random() * 200);
    await createNode({ type: newNodeType, title: newNodeTitle, positionX: centerX, positionY: centerY });
    setIsCreating(false);
    setNewNodeTitle('');
  }

  const selectedNode = graph.nodes.find((node) => node.id === selectedNodeId) ?? null;

  return (
    <main className={`quest-shell ${THEME_CLASS[themeId]} atlas-calm`} aria-label={t('map.ariaLabel')}>
      <ReactFlow
        className="quest-flow"
        nodes={flowNodes}
        edges={flowEdges}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        onNodesChange={handleNodesChange}
        onEdgesChange={handleEdgesChange}
        onConnect={handleConnect}
        onNodeClick={(_, node) => selectNode(node.id)}
        fitView
        fitViewOptions={{ padding: 0.18 }}
        minZoom={0.3}
        maxZoom={1.8}
        panOnDrag
        panOnScroll
        zoomOnScroll
        zoomOnPinch
        nodesDraggable
        nodesFocusable
        deleteKeyCode={['Backspace', 'Delete']}
        proOptions={{ hideAttribution: true }}
      >
        <Background />
        <Controls showInteractive={false} />
      </ReactFlow>

      <header className="topbar" data-map-overlay>
        <div className="brand-mark">Q</div>
        <div>
          <p className="eyebrow">{t('topbar.eyebrow')}</p>
          <h1>{graph.name}</h1>
        </div>
        <span className={`connection-pill ${source === 'api' ? 'api' : ''}`}>
          <i /> {source === 'api' ? t('topbar.apiConnected') : t('topbar.localMode')}
        </span>
        <ThemeSwitcher activeTheme={themeId} onSelect={setThemeId} />
        <LanguageToggle activeLocale={locale} onSelect={setLocale} />
      </header>

      <aside className={`creation-panel glass-panel ${isMobile ? 'bottom-sheet' : ''}`} data-map-overlay>
        <p className="eyebrow">{t('creationPanel.eyebrow')}</p>
        <h2>{t('creationPanel.title')}</h2>
        <form onSubmit={submitCreate}>
          <label htmlFor="node-type">{t('creationPanel.nodeTypeLabel')}</label>
          <select id="node-type" value={newNodeType} onChange={(event) => setNewNodeType(event.target.value as NodeType)}>
            <option value="OBJECTIF">{t('creationPanel.nodeTypeObjectif')}</option>
            <option value="ETAPE">{t('creationPanel.nodeTypeEtape')}</option>
          </select>
          <label htmlFor="node-title">{t('creationPanel.nodeTitleLabel')}</label>
          <input id="node-title" value={newNodeTitle} onChange={(event) => setNewNodeTitle(event.target.value)} placeholder={t('creationPanel.nodeTitlePlaceholder')} />
          <button type="submit" disabled={isCreating || newNodeTitle.trim() === ''}>
            {isCreating ? t('creationPanel.submitPending') : t('creationPanel.submit')} <span>→</span>
          </button>
        </form>
        {error && <p className="form-error" role="alert">{error}</p>}
      </aside>

      {selectedNode && (
        <aside className={`inspector glass-panel ${isMobile ? 'bottom-sheet' : ''}`} data-map-overlay>
          <div className="inspector-title">
            <h2>{selectedNode.title}</h2>
          </div>
          <span className={`state-badge ${selectedNode.status}`}>
            {selectedNode.status === 'completed' ? t('inspector.statusDone') : t('inspector.statusActive')}
          </span>
          {selectedNode.type === 'ETAPE' && (
            <button
              type="button"
              onClick={() => void updateNodeStatus(selectedNode.id, selectedNode.status === 'completed' ? 'active' : 'completed')}
            >
              {selectedNode.status === 'completed' ? t('inspector.markActive') : t('inspector.markDone')}
            </button>
          )}
          {selectedNode.type === 'OBJECTIF' && selectedNode.status !== 'completed' && (
            <button type="button" onClick={() => void validateObjectif(selectedNode.id)}>
              {t('inspector.validateObjectif')}
            </button>
          )}
          <button type="button" className="danger" onClick={() => void removeNode(selectedNode.id)}>
            {t('inspector.deleteNode')}
          </button>
        </aside>
      )}

      <p className="map-hint" data-map-overlay>{t('map.hint')}</p>
    </main>
  );
}
```

- [ ] **Step 3: Mettre à jour `page.tsx`**

```tsx
// web/src/app/page.tsx
import { SpaceCanvas } from '@/components/space-canvas';

export default function Home() {
  return <SpaceCanvas />;
}
```

- [ ] **Step 4: Supprimer l'ancien composant**

Run: `cd web && rm -f src/components/quest-map.tsx`

- [ ] **Step 5: Lancer le serveur de dev et vérifier visuellement**

Utiliser le tool de preview du navigateur (pas Bash) pour démarrer `web` (`npm run dev`), ouvrir la page, et vérifier :
- tous les nœuds du seed sont visibles simultanément sur un seul canvas (plus de filtre par objectif) ;
- glisser une Card déplace la Card ET les liens qui y sont attachés suivent en temps réel ;
- relâcher la Card persiste la position (recharger la page : la Card reste à sa nouvelle place, `source: api` requis) ;
- glisser depuis le Handle d'une Card vers une autre crée un lien visible ;
- sélectionner un lien puis appuyer sur Suppr le retire ;
- le panneau de création et l'inspecteur (HUD) restent cliquables sans jamais déplacer le canvas en dessous.

Expected: chaque point ci-dessus se vérifie sans erreur console. Corriger le code source si un point échoue, puis rejouer la vérification — ne pas continuer tant que le drag & drop et le suivi des liens ne sont pas visuellement corrects.

- [ ] **Step 6: Run unit/lint/build**

Run: `cd web && npm run test && npm run lint && npm run build`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add web/src/components/space-canvas.tsx web/src/components/quest-edge.tsx web/src/app/page.tsx
git rm web/src/components/quest-map.tsx
git commit -m "feat(web): remplace quest-map par space-canvas — drag & drop libre, connexions natives, HUD non bloquant"
```

---

### Task 11: Nettoyage i18n et CSS

**Files:**
- Modify: `web/src/i18n/locales/fr.json`
- Modify: `web/src/i18n/locales/en.json`
- Modify: `web/src/app/globals.css`

> **Modèle recommandé : ne pas utiliser `haiku` pour la partie CSS de cette tâche** (Step 2) — c'est un ajustement visuel direct sur les Cards.

- [ ] **Step 1: Ajouter/renommer les clés i18n**

Dans `web/src/i18n/locales/fr.json` et `en.json`, s'assurer que ces clés existent (ajouter celles qui manquent, adapter celles qui référençaient l'ancien vocabulaire Quest/Step) :

```json
{
  "creationPanel": {
    "eyebrow": "Nouveau nœud",
    "title": "Ajouter au Space",
    "nodeTypeLabel": "Type",
    "nodeTypeObjectif": "Objectif",
    "nodeTypeEtape": "Étape",
    "nodeTitleLabel": "Titre",
    "nodeTitlePlaceholder": "Ex : Signer 3 clients récurrents",
    "submit": "Ajouter",
    "submitPending": "Ajout…"
  },
  "inspector": {
    "statusDone": "Complété",
    "statusActive": "Actif",
    "markDone": "Marquer complété",
    "markActive": "Remettre actif",
    "validateObjectif": "Valider l’objectif",
    "deleteNode": "Supprimer"
  },
  "node": {
    "objectiveEyebrow": "Objectif",
    "statusDone": "Complété",
    "statusActive": "Actif"
  },
  "topbar": {
    "eyebrow": "Ton atlas",
    "apiConnected": "Connecté à l’API",
    "localMode": "Mode local"
  },
  "map": {
    "ariaLabel": "Carte du Space",
    "hint": "Glisse une Card pour la déplacer, relie deux Cards depuis leurs poignées."
  }
}
```

(Traduire l'équivalent en anglais dans `en.json` avec les mêmes clés.)

- [ ] **Step 2: Adapter les sélecteurs CSS au nouveau vocabulaire de statut**

Dans `web/src/app/globals.css`, chercher les sélecteurs `.status-done` et les remplacer par `.status-completed` (le statut `done` n'existe plus, remplacé par `completed`) :

Run: `cd web && grep -rn "status-done" src/app/globals.css`

Remplacer chaque occurrence trouvée de `.status-done` par `.status-completed`, et chaque `.status-locked` restante (le statut `locked` n'existe plus) par une suppression de la règle correspondante si elle n'est plus utilisée ailleurs — vérifier avec `grep -rn "status-locked" src` avant de supprimer.

- [ ] **Step 3: Run tests i18n dédiés**

Run: `cd web && npx vitest run src/i18n`
Expected: PASS — notamment `no-hardcoded-strings.test.ts` et `locales.test.ts`, qui vérifient que toutes les clés utilisées existent dans les deux locales.

- [ ] **Step 4: Commit**

```bash
git add web/src/i18n/locales/fr.json web/src/i18n/locales/en.json web/src/app/globals.css
git commit -m "chore(web): aligne les traductions et les classes CSS de statut sur le vocabulaire Node"
```

---

### Task 12: Recette manuelle et suivi de chantier

**Files:**
- Create: `docs/manual-tests/story-graphe-unifie-space-node-edge.md`
- Modify: `TODO.md`
- Modify: `_bmad-output/implementation-artifacts/sprint-status.yaml`

- [ ] **Step 1: Écrire la recette manuelle**

```markdown
# Recette manuelle — Graphe unifié Space/Node/Edge

## Prérequis

- `docker compose up --build` (Postgres + API) depuis `api/`.
- `npm run db:seed` dans `api/` pour charger le graphe d'exemple.
- `npm run dev` dans `web/`.

## Scénario 1 — Vue d'ensemble

**Given** le seed a été chargé
**When** j'ouvre la page d'accueil
**Then** je vois tous les objectifs (Gagner beaucoup d'argent, Freelance, Cloudbreak, achat-revente) et leurs étapes sur un seul Space, y compris l'étape "Épargner 3 000 € de trésorerie" reliée à la fois à Cloudbreak et à achat-revente.

## Scénario 2 — Déplacer une Card

**Given** le Space est affiché
**When** je clique-maintiens une Card et la déplace
**Then** les liens connectés suivent la Card en continu pendant le déplacement
**And** après un rechargement de page, la Card reste à sa nouvelle position.

## Scénario 3 — Créer un lien

**Given** deux Cards existent sans lien entre elles
**When** je glisse depuis la poignée d'une Card vers une autre
**Then** un nouveau lien apparaît immédiatement et persiste après rechargement.

## Scénario 4 — Compléter une étape et voir la progression

**Given** une étape reliée à un objectif est "active"
**When** je la marque "complétée" depuis l'inspecteur
**Then** la progression de l'objectif augmente en conséquence (vérifiable via `GET /nodes/:id/progress`).

## Scénario 5 — Valider un objectif

**Given** un objectif affiche 100% de progression
**When** je clique sur "Valider l'objectif"
**Then** son statut passe à "complété" (avant ce clic, il restait "actif" malgré les 100%).

## Cas limite — Supprimer un nœud partagé

**Given** l'étape "Épargner 3 000 € de trésorerie" est reliée à deux objectifs
**When** je la supprime
**Then** les deux objectifs restent sur le Space, simplement sans lien entrant de cette étape (pas de suppression en cascade).

## Checklist finale

- [ ] Tous les nœuds du seed sont visibles simultanément
- [ ] Drag & drop fluide, liens qui suivent
- [ ] Création de lien par glisser-déposer
- [ ] Suppression de lien (sélection + Suppr)
- [ ] Calcul de progression correct pour une étape partagée
- [ ] Validation manuelle distincte du 100%
- [ ] Suppression de nœud sans cascade
```

- [ ] **Step 2: Mettre à jour `TODO.md`**

Remplacer la section "Sprint actif" et "En cours" pour refléter que les stories 1.1/1.2 sont remplacées par cette refonte du modèle de données (à adapter avec le contenu réel de `TODO.md` au moment de l'exécution — lire le fichier avant d'éditer).

- [ ] **Step 3: Mettre à jour `sprint-status.yaml`**

Lire `_bmad-output/implementation-artifacts/sprint-status.yaml` et marquer les anciennes stories Quest/Step comme remplacées par la nouvelle direction Node/Edge, en ajoutant une entrée pour ce chantier.

- [ ] **Step 4: Commit**

```bash
git add docs/manual-tests/story-graphe-unifie-space-node-edge.md TODO.md _bmad-output/implementation-artifacts/sprint-status.yaml
git commit -m "docs: recette manuelle et suivi de chantier pour le graphe unifié Space/Node/Edge"
```

---

## Self-Review Notes (déjà appliqué à ce document)

- Chaque décision de la spec (nœud unique, DAG multi-parents, Space léger, pas de verrouillage, progression par ratio, validation manuelle, suppression = coupe les liens uniquement, position persistée, HUD non bloquant) a une tâche correspondante.
- Les noms (`SpaceNode`, `SpaceEdge`, `SpaceGraph`, `useSpaceMap`, `SpaceCanvas`, `NodesService`, `EdgesService`) sont réutilisés à l'identique entre les tâches back et front.
- Aucune étape ne renvoie à "Task N" sans réécrire le code concerné.

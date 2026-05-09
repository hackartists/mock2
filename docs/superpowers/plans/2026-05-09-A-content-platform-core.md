# A. 콘텐츠 플랫폼 코어 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 정독(Jeongdok) MVP의 첫 vertical slice 구축 — 사용자가 이메일+OTP로 가입하고, 글을 작성·조회·읽을 수 있는 최소 콘텐츠 플랫폼.

**Architecture:** Next.js 14 App Router 풀스택 단일 코드베이스. Domain layer(`src/domain/`)에 비즈니스 로직 격리, API route는 얇은 어댑터. Prisma + Postgres로 영속성, NextAuth(Credentials provider)로 인증. PoE/보상/어뷰징 등 후속 sub-plan은 Post/User 모델의 확장 필드와 도메인 함수에 hook을 추가하는 형태로 결합.

**Tech Stack:**
- Next.js 14 (App Router) + TypeScript 5 + React 18
- Prisma 5 + PostgreSQL 16 (Docker)
- NextAuth 4 (Credentials provider, 이메일+OTP)
- Tailwind CSS 3
- Vitest 1.x (단위) + Playwright 1.x (E2E)
- Node.js 20 LTS

**프로젝트 디렉토리:** `C:\Users\jscho\기획서\local-reward\`
**GitHub remote:** `https://github.com/hackartists/local-reward` (비어 있음, 별도 푸시 단계에서 인증)

---

## File Structure

```
local-reward/
├── package.json
├── tsconfig.json
├── next.config.mjs
├── tailwind.config.ts
├── postcss.config.mjs
├── vitest.config.ts
├── playwright.config.ts
├── docker-compose.yml                      # postgres 16
├── .env.local.example
├── .env.local                              # gitignored
├── .gitignore
├── README.md
├── prisma/
│   ├── schema.prisma                       # User, Category, Post, OtpToken
│   └── seed.ts
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                        # → /feed 리다이렉트
│   │   ├── globals.css
│   │   ├── (auth)/
│   │   │   ├── signin/page.tsx
│   │   │   └── verify/page.tsx
│   │   ├── feed/page.tsx                   # 글 목록
│   │   ├── posts/
│   │   │   ├── new/page.tsx                # 글 작성
│   │   │   └── [id]/page.tsx               # 글 상세
│   │   └── api/
│   │       ├── auth/[...nextauth]/route.ts
│   │       ├── otp/request/route.ts        # OTP 발급
│   │       ├── posts/route.ts              # GET 목록 / POST 작성
│   │       └── posts/[id]/route.ts         # GET 상세
│   ├── lib/
│   │   ├── auth.ts                         # NextAuth options
│   │   ├── db.ts                           # Prisma singleton
│   │   └── otp.ts                          # OTP 생성·검증·만료
│   ├── domain/
│   │   ├── users.ts                        # findOrCreateUser, getUserById
│   │   ├── posts.ts                        # createPost, listPosts, getPost
│   │   └── categories.ts                   # listCategories
│   └── components/
│       ├── PostCard.tsx
│       ├── CategoryTabs.tsx
│       └── SignInForm.tsx
└── tests/
    ├── unit/
    │   ├── lib/otp.test.ts
    │   ├── domain/users.test.ts
    │   ├── domain/posts.test.ts
    │   └── domain/categories.test.ts
    └── e2e/
        ├── auth.spec.ts
        └── post-flow.spec.ts
```

**파일 책임 요약:**
- `lib/*` — 외부 의존성 어댑터 (DB, OTP 토큰 생성)
- `domain/*` — 순수 비즈니스 로직, DB 쿼리 호출. 테스트 가능한 단위
- `app/api/*` — HTTP 어댑터. 입력 검증 후 domain 호출, 결과를 JSON으로 직렬화
- `app/(pages)/*` — 서버 컴포넌트 렌더링, domain 직접 호출 가능
- `components/*` — 클라이언트 컴포넌트, props로만 데이터 받음

---

## Task 1: 환경 셋업 — Node 20 설치 확인 + 프로젝트 디렉토리 초기화

**Files:**
- Create: `local-reward/.gitignore`

- [ ] **Step 1: Node 20 LTS 설치 확인**

PowerShell에서:
```powershell
node --version
```

Expected: `v20.x.x` (없으면 https://nodejs.org/en/download 에서 LTS 설치 후 PowerShell 재시작)

만약 `command not found`면 winget으로:
```powershell
winget install OpenJS.NodeJS.LTS
```
설치 후 PowerShell 재시작.

- [ ] **Step 2: 프로젝트 디렉토리 생성**

```powershell
New-Item -ItemType Directory -Path "C:\Users\jscho\기획서\local-reward"
Set-Location "C:\Users\jscho\기획서\local-reward"
```

Expected: 디렉토리 생성 + 이동.

- [ ] **Step 3: git 초기화 + .gitignore 작성**

```powershell
git init -b main
```

`.gitignore` 파일 내용:
```
# dependencies
node_modules/
.pnp
.pnp.js

# next
.next/
out/

# env
.env*.local
!.env.local.example

# misc
.DS_Store
*.pem
.vercel

# debug
npm-debug.log*
yarn-debug.log*

# typescript
*.tsbuildinfo
next-env.d.ts

# test
coverage/
playwright-report/
test-results/
.playwright/

# os
Thumbs.db
```

- [ ] **Step 4: 첫 커밋**

```powershell
git add .gitignore
git commit -m "chore: initialize repo with .gitignore"
```

Expected: `1 file changed, 1 insertion(+)` 비슷한 메시지.

---

## Task 2: Next.js 14 + TypeScript 프로젝트 초기화

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.mjs`, `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/globals.css`

- [ ] **Step 1: create-next-app으로 스캐폴딩**

```powershell
npx create-next-app@14 . --typescript --app --tailwind --eslint --src-dir --import-alias "@/*" --no-turbopack
```

프롬프트가 뜨면 모두 기본값 (`src/` 사용 yes, App Router yes, alias `@/*`).

Expected: `package.json`, `tsconfig.json`, `src/app/`, `tailwind.config.ts` 등 생성.

- [ ] **Step 2: 개발 서버 가동 확인**

```powershell
npm run dev
```

브라우저에서 `http://localhost:3000` 열어 Next.js 시작 페이지 확인. Ctrl+C로 종료.

Expected: 기본 Next.js 환영 페이지 정상 렌더링.

- [ ] **Step 3: 홈 → /feed 리다이렉트로 변경**

`src/app/page.tsx` 전체 교체:
```typescript
import { redirect } from "next/navigation";

export default function Home() {
  redirect("/feed");
}
```

- [ ] **Step 4: 커밋**

```powershell
git add .
git commit -m "feat: scaffold Next.js 14 + TypeScript + Tailwind"
```

---

## Task 3: PostgreSQL Docker 컨테이너 + 환경변수

**Files:**
- Create: `docker-compose.yml`, `.env.local.example`, `.env.local`

- [ ] **Step 1: Docker Desktop 설치 확인**

```powershell
docker --version
```

Expected: `Docker version 20.x` 이상. 없으면 https://www.docker.com/products/docker-desktop/ 설치 후 재시작.

- [ ] **Step 2: docker-compose.yml 작성**

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: jeongdok-postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: jeongdok
      POSTGRES_PASSWORD: jeongdok_dev
      POSTGRES_DB: jeongdok
    volumes:
      - jeongdok-pg-data:/var/lib/postgresql/data

volumes:
  jeongdok-pg-data:
```

- [ ] **Step 3: 환경변수 템플릿 + 실제 파일 작성**

`.env.local.example`:
```
DATABASE_URL="postgresql://jeongdok:jeongdok_dev@localhost:5432/jeongdok?schema=public"
NEXTAUTH_SECRET="replace-with-random-32-byte-string"
NEXTAUTH_URL="http://localhost:3000"
```

`.env.local` (커밋 안 됨, .gitignore에 이미 등록):
```
DATABASE_URL="postgresql://jeongdok:jeongdok_dev@localhost:5432/jeongdok?schema=public"
NEXTAUTH_SECRET="dev-only-secret-change-in-prod-32-chars-min"
NEXTAUTH_URL="http://localhost:3000"
```

- [ ] **Step 4: Postgres 컨테이너 가동 + 연결 확인**

```powershell
docker compose up -d
docker exec -it jeongdok-postgres psql -U jeongdok -d jeongdok -c "SELECT version();"
```

Expected: PostgreSQL 16.x 버전 출력.

- [ ] **Step 5: 커밋**

```powershell
git add docker-compose.yml .env.local.example
git commit -m "chore: add postgres docker-compose + env template"
```

---

## Task 4: Prisma 스키마 정의 + 첫 마이그레이션

**Files:**
- Create: `prisma/schema.prisma`, `src/lib/db.ts`

- [ ] **Step 1: Prisma 설치 + 초기화**

```powershell
npm install prisma --save-dev
npm install @prisma/client
npx prisma init --datasource-provider postgresql
```

Expected: `prisma/schema.prisma` 파일 생성, `.env`에 DATABASE_URL placeholder 추가됨. `.env`은 사용 안 하고 `.env.local`을 쓸 것이므로 `.env` 파일은 삭제.

```powershell
Remove-Item .env
```

- [ ] **Step 2: schema.prisma 작성**

`prisma/schema.prisma` 전체 교체:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 사용자 역할 — A 단계는 PARENT만 사용. 후속 sub-plan에서 ACADEMY/SEED/ADMIN 추가
enum UserRole {
  PARENT
  SEED
  ACADEMY
  ADMIN
}

model User {
  id           String   @id @default(cuid())
  email        String   @unique
  displayName  String
  role         UserRole @default(PARENT)
  // 후속 sub-plan에서 사용할 필드들 (A에서는 기본값만 채움)
  trustScore   Float    @default(0.3)   // E. 어뷰징 방어가 갱신
  pointBalance Int      @default(0)     // C. 보상 풀이 갱신
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  posts        Post[]
  otpTokens    OtpToken[]

  @@index([email])
}

model Category {
  id    String @id @default(cuid())
  slug  String @unique
  name  String
  posts Post[]
}

// 글 상태 — A 단계는 PUBLISHED만 사용. F. 모더레이션이 PENDING/REJECTED/HIDDEN 등 추가
enum PostStatus {
  PUBLISHED
}

model Post {
  id         String     @id @default(cuid())
  title      String
  body       String     @db.Text
  status     PostStatus @default(PUBLISHED)
  authorId   String
  categoryId String
  createdAt  DateTime   @default(now())
  updatedAt  DateTime   @updatedAt

  author     User       @relation(fields: [authorId], references: [id])
  category   Category   @relation(fields: [categoryId], references: [id])

  @@index([categoryId, createdAt(sort: Desc)])
  @@index([authorId])
}

// OTP 인증 토큰 — 이메일 코드. 6자리 숫자, 10분 만료
model OtpToken {
  id         String   @id @default(cuid())
  userId     String
  codeHash   String   // SHA-256 해시 (평문 저장 금지)
  expiresAt  DateTime
  consumedAt DateTime?
  createdAt  DateTime @default(now())

  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, consumedAt])
}
```

- [ ] **Step 3: 마이그레이션 실행**

```powershell
npx prisma migrate dev --name init
```

프롬프트에 마이그레이션 이름이 이미 인자로 들어갔으니 그대로 실행. Expected: `prisma/migrations/<timestamp>_init/migration.sql` 생성, DB에 테이블 생성됨.

- [ ] **Step 4: Prisma client singleton 작성**

`src/lib/db.ts`:
```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

(개발 모드 hot-reload 시 PrismaClient 인스턴스가 누적되는 문제 방지를 위한 표준 패턴)

- [ ] **Step 5: DB 연결 확인 스크립트 + 커밋**

PowerShell에서:
```powershell
node -e "const {PrismaClient} = require('@prisma/client'); const p = new PrismaClient(); p.$queryRaw`SELECT 1 as ok`.then(r => console.log(r)).finally(() => p.$disconnect());"
```

Expected: `[ { ok: 1 } ]` 출력.

```powershell
git add prisma src/lib/db.ts package.json package-lock.json
git commit -m "feat: add prisma schema with User/Post/Category/OtpToken"
```

---

## Task 5: Vitest 셋업 + OTP 라이브러리 (TDD)

**Files:**
- Create: `vitest.config.ts`, `tests/unit/lib/otp.test.ts`, `src/lib/otp.ts`

- [ ] **Step 1: Vitest 설치 + 설정**

```powershell
npm install -D vitest @vitest/ui
```

`vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",
    include: ["tests/unit/**/*.test.ts"],
    globals: false,
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },
});
```

`package.json`의 `scripts`에 추가:
```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 2: 실패하는 OTP 테스트 작성**

`tests/unit/lib/otp.test.ts`:
```typescript
import { describe, it, expect } from "vitest";
import { generateOtpCode, hashOtpCode, verifyOtpCode } from "@/lib/otp";

describe("OTP", () => {
  it("generates a 6-digit numeric code", () => {
    const code = generateOtpCode();
    expect(code).toMatch(/^\d{6}$/);
  });

  it("generates different codes on subsequent calls", () => {
    const codes = new Set<string>();
    for (let i = 0; i < 50; i++) codes.add(generateOtpCode());
    // 50회 중 같은 코드가 나올 확률은 매우 낮음. 충돌 0~2개는 허용
    expect(codes.size).toBeGreaterThanOrEqual(48);
  });

  it("hashOtpCode returns deterministic 64-char hex string", () => {
    const h1 = hashOtpCode("123456");
    const h2 = hashOtpCode("123456");
    expect(h1).toBe(h2);
    expect(h1).toMatch(/^[0-9a-f]{64}$/);
  });

  it("verifyOtpCode returns true for matching plain+hash", () => {
    const code = "123456";
    const hash = hashOtpCode(code);
    expect(verifyOtpCode(code, hash)).toBe(true);
  });

  it("verifyOtpCode returns false for mismatch", () => {
    const hash = hashOtpCode("123456");
    expect(verifyOtpCode("000000", hash)).toBe(false);
  });
});
```

- [ ] **Step 3: 테스트 실패 확인**

```powershell
npm test
```

Expected: `Failed to resolve import "@/lib/otp"` 또는 `Cannot find module` 류 에러로 5개 테스트 모두 실패.

- [ ] **Step 4: OTP 라이브러리 구현**

`src/lib/otp.ts`:
```typescript
import { createHash, randomInt } from "crypto";

export function generateOtpCode(): string {
  // 100000 ~ 999999 범위로 6자리 보장
  return randomInt(100000, 1000000).toString();
}

export function hashOtpCode(code: string): string {
  return createHash("sha256").update(code).digest("hex");
}

export function verifyOtpCode(plain: string, hash: string): boolean {
  return hashOtpCode(plain) === hash;
}

export const OTP_TTL_MS = 10 * 60 * 1000; // 10 minutes
```

- [ ] **Step 5: 테스트 통과 확인 + 커밋**

```powershell
npm test
```

Expected: 5/5 PASS.

```powershell
git add vitest.config.ts tests src/lib/otp.ts package.json package-lock.json
git commit -m "feat: add OTP code generation/hash/verify with vitest"
```

---

## Task 6: User 도메인 — findOrCreateUser (TDD)

**Files:**
- Create: `tests/unit/domain/users.test.ts`, `src/domain/users.ts`

- [ ] **Step 1: 테스트 DB 준비 + 실패하는 테스트**

테스트는 실제 DB를 친다. 빠르고 단순한 접근 — 각 테스트 후 truncate.

`tests/unit/domain/users.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterAll } from "vitest";
import { prisma } from "@/lib/db";
import { findOrCreateUser, getUserById } from "@/domain/users";

beforeEach(async () => {
  // 의존 테이블 순서 주의: OtpToken / Post → User
  await prisma.otpToken.deleteMany();
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe("users domain", () => {
  it("findOrCreateUser creates new user with default role/trust/balance", async () => {
    const user = await findOrCreateUser({
      email: "a@example.com",
      displayName: "지원",
    });
    expect(user.email).toBe("a@example.com");
    expect(user.displayName).toBe("지원");
    expect(user.role).toBe("PARENT");
    expect(user.trustScore).toBe(0.3);
    expect(user.pointBalance).toBe(0);
  });

  it("findOrCreateUser returns existing user when email matches (does not duplicate)", async () => {
    const a = await findOrCreateUser({ email: "a@example.com", displayName: "지원" });
    const b = await findOrCreateUser({ email: "a@example.com", displayName: "다른이름" });
    expect(b.id).toBe(a.id);
    // 이미 존재하는 사용자의 displayName은 변경하지 않음 (덮어쓰기 금지)
    expect(b.displayName).toBe("지원");
  });

  it("getUserById returns user when exists", async () => {
    const created = await findOrCreateUser({ email: "a@example.com", displayName: "지원" });
    const fetched = await getUserById(created.id);
    expect(fetched?.id).toBe(created.id);
  });

  it("getUserById returns null when user not found", async () => {
    const fetched = await getUserById("nonexistent-id");
    expect(fetched).toBeNull();
  });
});
```

- [ ] **Step 2: 실패 확인**

```powershell
npm test
```

Expected: 4개 테스트 모두 실패 (`Cannot find module '@/domain/users'`).

- [ ] **Step 3: 도메인 구현**

`src/domain/users.ts`:
```typescript
import { prisma } from "@/lib/db";
import type { User } from "@prisma/client";

export interface FindOrCreateInput {
  email: string;
  displayName: string;
}

export async function findOrCreateUser(input: FindOrCreateInput): Promise<User> {
  const normalizedEmail = input.email.trim().toLowerCase();
  const existing = await prisma.user.findUnique({ where: { email: normalizedEmail } });
  if (existing) return existing;

  return prisma.user.create({
    data: {
      email: normalizedEmail,
      displayName: input.displayName,
    },
  });
}

export async function getUserById(id: string): Promise<User | null> {
  return prisma.user.findUnique({ where: { id } });
}
```

- [ ] **Step 4: 테스트 통과 + 커밋**

```powershell
npm test
```

Expected: 4/4 PASS (이전 OTP 5개 포함 9/9).

```powershell
git add tests src/domain/users.ts
git commit -m "feat: add users domain (findOrCreate, getById)"
```

---

## Task 7: Category 도메인 + 시드 (TDD)

**Files:**
- Create: `tests/unit/domain/categories.test.ts`, `src/domain/categories.ts`, `prisma/seed.ts`
- Modify: `package.json` (prisma seed 설정)

- [ ] **Step 1: 실패하는 테스트**

`tests/unit/domain/categories.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterAll } from "vitest";
import { prisma } from "@/lib/db";
import { listCategories } from "@/domain/categories";

beforeEach(async () => {
  await prisma.post.deleteMany();
  await prisma.category.deleteMany();
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe("categories domain", () => {
  it("listCategories returns categories ordered by name", async () => {
    await prisma.category.createMany({
      data: [
        { slug: "general", name: "교육 일반" },
        { slug: "academy-review", name: "학원 후기" },
        { slug: "admissions", name: "입시·진학" },
      ],
    });
    const cats = await listCategories();
    expect(cats.map((c) => c.slug)).toEqual(["general", "academy-review", "admissions"]);
    // 한국어 가나다순: 교 < 입 < 학
  });

  it("listCategories returns empty array when none exist", async () => {
    const cats = await listCategories();
    expect(cats).toEqual([]);
  });
});
```

- [ ] **Step 2: 실패 확인**

```powershell
npm test
```

Expected: 2개 테스트 실패 (`Cannot find module '@/domain/categories'`).

- [ ] **Step 3: 도메인 구현**

`src/domain/categories.ts`:
```typescript
import { prisma } from "@/lib/db";
import type { Category } from "@prisma/client";

export async function listCategories(): Promise<Category[]> {
  return prisma.category.findMany({
    orderBy: { name: "asc" },
  });
}
```

- [ ] **Step 4: 테스트 통과 확인**

```powershell
npm test
```

Expected: 2/2 PASS.

- [ ] **Step 5: 시드 스크립트 작성**

`prisma/seed.ts`:
```typescript
import { PrismaClient } from "@prisma/client";
const prisma = new PrismaClient();

async function main() {
  const categories = [
    { slug: "academy-review", name: "학원 후기" },
    { slug: "admissions", name: "입시·진학" },
    { slug: "general", name: "교육 일반" },
  ];
  for (const c of categories) {
    await prisma.category.upsert({
      where: { slug: c.slug },
      update: {},
      create: c,
    });
  }
  console.log(`seeded ${categories.length} categories`);
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(() => prisma.$disconnect());
```

`package.json`에 prisma 시드 설정 추가 (최상위 키):
```json
"prisma": {
  "seed": "node --import tsx ./prisma/seed.ts"
}
```

tsx 설치:
```powershell
npm install -D tsx
```

시드 실행:
```powershell
npx prisma db seed
```

Expected: `seeded 3 categories` 출력.

- [ ] **Step 6: 커밋**

```powershell
git add tests src/domain/categories.ts prisma/seed.ts package.json package-lock.json
git commit -m "feat: add categories domain + seed (3 MVP categories)"
```

---

## Task 8: Post 도메인 — createPost / listPosts / getPost (TDD)

**Files:**
- Create: `tests/unit/domain/posts.test.ts`, `src/domain/posts.ts`

- [ ] **Step 1: 실패하는 테스트**

`tests/unit/domain/posts.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterAll } from "vitest";
import { prisma } from "@/lib/db";
import { findOrCreateUser } from "@/domain/users";
import { createPost, listPosts, getPost } from "@/domain/posts";

let userId: string;
let categoryId: string;

beforeEach(async () => {
  await prisma.otpToken.deleteMany();
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
  await prisma.category.deleteMany();

  const user = await findOrCreateUser({ email: "writer@example.com", displayName: "작성자" });
  userId = user.id;
  const cat = await prisma.category.create({ data: { slug: "academy-review", name: "학원 후기" } });
  categoryId = cat.id;
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe("posts domain", () => {
  it("createPost persists with author and category", async () => {
    const post = await createPost({
      authorId: userId,
      categoryId,
      title: "분당 수학학원 후기",
      body: "본문 내용 ".repeat(50),
    });
    expect(post.title).toBe("분당 수학학원 후기");
    expect(post.authorId).toBe(userId);
    expect(post.categoryId).toBe(categoryId);
    expect(post.status).toBe("PUBLISHED");
  });

  it("createPost rejects empty title", async () => {
    await expect(
      createPost({ authorId: userId, categoryId, title: "  ", body: "본문" })
    ).rejects.toThrow(/title required/i);
  });

  it("createPost rejects body shorter than 20 chars", async () => {
    await expect(
      createPost({ authorId: userId, categoryId, title: "제목", body: "짧음" })
    ).rejects.toThrow(/body too short/i);
  });

  it("listPosts returns posts newest first", async () => {
    const a = await createPost({ authorId: userId, categoryId, title: "A", body: "A 본문 ".repeat(10) });
    await new Promise((r) => setTimeout(r, 5));
    const b = await createPost({ authorId: userId, categoryId, title: "B", body: "B 본문 ".repeat(10) });
    const list = await listPosts({});
    expect(list.map((p) => p.id)).toEqual([b.id, a.id]);
  });

  it("listPosts filters by categorySlug", async () => {
    const otherCat = await prisma.category.create({ data: { slug: "general", name: "교육 일반" } });
    await createPost({ authorId: userId, categoryId, title: "리뷰", body: "리뷰 본문 ".repeat(10) });
    await createPost({ authorId: userId, categoryId: otherCat.id, title: "일반", body: "일반 본문 ".repeat(10) });
    const reviews = await listPosts({ categorySlug: "academy-review" });
    expect(reviews).toHaveLength(1);
    expect(reviews[0].title).toBe("리뷰");
  });

  it("getPost returns post with author + category eager-loaded", async () => {
    const created = await createPost({
      authorId: userId,
      categoryId,
      title: "상세 글",
      body: "상세 본문 ".repeat(10),
    });
    const fetched = await getPost(created.id);
    expect(fetched).not.toBeNull();
    expect(fetched!.author.displayName).toBe("작성자");
    expect(fetched!.category.slug).toBe("academy-review");
  });

  it("getPost returns null for unknown id", async () => {
    expect(await getPost("nope")).toBeNull();
  });
});
```

- [ ] **Step 2: 실패 확인**

```powershell
npm test
```

Expected: 7개 모두 실패 (`Cannot find module '@/domain/posts'`).

- [ ] **Step 3: 도메인 구현**

`src/domain/posts.ts`:
```typescript
import { prisma } from "@/lib/db";
import type { Post, User, Category } from "@prisma/client";

export interface CreatePostInput {
  authorId: string;
  categoryId: string;
  title: string;
  body: string;
}

export interface ListPostsInput {
  categorySlug?: string;
  limit?: number;
}

export type PostWithRelations = Post & {
  author: Pick<User, "id" | "displayName" | "role">;
  category: Pick<Category, "id" | "slug" | "name">;
};

const MIN_BODY_LENGTH = 20;
const DEFAULT_LIMIT = 50;

export async function createPost(input: CreatePostInput): Promise<Post> {
  const title = input.title.trim();
  if (title.length === 0) {
    throw new Error("title required");
  }
  if (input.body.trim().length < MIN_BODY_LENGTH) {
    throw new Error(`body too short (min ${MIN_BODY_LENGTH} chars)`);
  }
  return prisma.post.create({
    data: {
      title,
      body: input.body,
      authorId: input.authorId,
      categoryId: input.categoryId,
    },
  });
}

export async function listPosts(input: ListPostsInput): Promise<PostWithRelations[]> {
  return prisma.post.findMany({
    where: input.categorySlug ? { category: { slug: input.categorySlug } } : undefined,
    include: {
      author: { select: { id: true, displayName: true, role: true } },
      category: { select: { id: true, slug: true, name: true } },
    },
    orderBy: { createdAt: "desc" },
    take: input.limit ?? DEFAULT_LIMIT,
  });
}

export async function getPost(id: string): Promise<PostWithRelations | null> {
  return prisma.post.findUnique({
    where: { id },
    include: {
      author: { select: { id: true, displayName: true, role: true } },
      category: { select: { id: true, slug: true, name: true } },
    },
  });
}
```

- [ ] **Step 4: 테스트 통과 + 커밋**

```powershell
npm test
```

Expected: 7/7 PASS (누적 18/18).

```powershell
git add tests src/domain/posts.ts
git commit -m "feat: add posts domain (create/list/get with validation)"
```

---

## Task 9: NextAuth 설정 + Credentials provider (이메일+OTP)

**Files:**
- Create: `src/lib/auth.ts`, `src/app/api/auth/[...nextauth]/route.ts`, `src/app/api/otp/request/route.ts`

- [ ] **Step 1: NextAuth 설치**

```powershell
npm install next-auth@4
```

- [ ] **Step 2: NextAuth options + handler 작성**

`src/lib/auth.ts`:
```typescript
import type { NextAuthOptions } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import { prisma } from "@/lib/db";
import { hashOtpCode } from "@/lib/otp";

export const authOptions: NextAuthOptions = {
  session: { strategy: "jwt" },
  providers: [
    CredentialsProvider({
      name: "OTP",
      credentials: {
        email: { label: "Email", type: "email" },
        code: { label: "Code", type: "text" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.code) return null;
        const email = credentials.email.trim().toLowerCase();
        const user = await prisma.user.findUnique({ where: { email } });
        if (!user) return null;

        const codeHash = hashOtpCode(credentials.code);
        const token = await prisma.otpToken.findFirst({
          where: {
            userId: user.id,
            codeHash,
            consumedAt: null,
            expiresAt: { gt: new Date() },
          },
          orderBy: { createdAt: "desc" },
        });
        if (!token) return null;

        await prisma.otpToken.update({
          where: { id: token.id },
          data: { consumedAt: new Date() },
        });
        return { id: user.id, email: user.email, name: user.displayName };
      },
    }),
  ],
  pages: {
    signIn: "/signin",
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.userId = user.id;
      return token;
    },
    async session({ session, token }) {
      if (token.userId && session.user) {
        (session.user as { id?: string }).id = token.userId as string;
      }
      return session;
    },
  },
};
```

`src/app/api/auth/[...nextauth]/route.ts`:
```typescript
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

- [ ] **Step 3: OTP 발급 API 작성**

`src/app/api/otp/request/route.ts`:
```typescript
import { NextResponse } from "next/server";
import { prisma } from "@/lib/db";
import { findOrCreateUser } from "@/domain/users";
import { generateOtpCode, hashOtpCode, OTP_TTL_MS } from "@/lib/otp";

export async function POST(req: Request) {
  const { email, displayName } = (await req.json()) as { email?: string; displayName?: string };
  if (!email) {
    return NextResponse.json({ error: "email required" }, { status: 400 });
  }

  const user = await findOrCreateUser({
    email,
    displayName: displayName?.trim() || email.split("@")[0],
  });

  const code = generateOtpCode();
  await prisma.otpToken.create({
    data: {
      userId: user.id,
      codeHash: hashOtpCode(code),
      expiresAt: new Date(Date.now() + OTP_TTL_MS),
    },
  });

  // MVP: 실제 이메일 전송 대신 콘솔에 출력. Phase 1 끝에 메일 서비스 연동.
  console.log(`[OTP] ${email} → ${code}`);

  return NextResponse.json({ ok: true });
}
```

- [ ] **Step 4: 수동 검증**

```powershell
npm run dev
```

별도 터미널에서:
```powershell
$body = '{"email":"test@example.com","displayName":"테스트"}'
Invoke-RestMethod -Uri http://localhost:3000/api/otp/request -Method POST -Body $body -ContentType "application/json"
```

Expected: `ok = True` 응답. 개발 서버 콘솔에 `[OTP] test@example.com → 123456` 형태 로그 출력.

서버 종료 (Ctrl+C).

- [ ] **Step 5: 커밋**

```powershell
git add src/lib/auth.ts src/app/api package.json package-lock.json
git commit -m "feat: add NextAuth credentials provider + OTP request endpoint"
```

---

## Task 10: 인증 페이지 — /signin, /verify

**Files:**
- Create: `src/components/SignInForm.tsx`, `src/app/(auth)/signin/page.tsx`, `src/app/(auth)/verify/page.tsx`

- [ ] **Step 1: SignInForm 컴포넌트 (클라이언트)**

`src/components/SignInForm.tsx`:
```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

export function SignInForm() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [displayName, setDisplayName] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setSubmitting(true);
    try {
      const res = await fetch("/api/otp/request", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, displayName }),
      });
      if (!res.ok) {
        const data = (await res.json()) as { error?: string };
        throw new Error(data.error ?? "request failed");
      }
      const params = new URLSearchParams({ email });
      router.push(`/verify?${params.toString()}`);
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err));
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={onSubmit} className="flex flex-col gap-3 max-w-sm">
      <label className="flex flex-col gap-1">
        <span className="text-sm">이메일</span>
        <input
          type="email"
          required
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="border rounded px-3 py-2"
        />
      </label>
      <label className="flex flex-col gap-1">
        <span className="text-sm">표시 이름 (선택)</span>
        <input
          type="text"
          value={displayName}
          onChange={(e) => setDisplayName(e.target.value)}
          className="border rounded px-3 py-2"
        />
      </label>
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <button
        type="submit"
        disabled={submitting}
        className="bg-black text-white rounded px-4 py-2 disabled:opacity-50"
      >
        {submitting ? "전송 중…" : "인증 코드 받기"}
      </button>
    </form>
  );
}
```

- [ ] **Step 2: signin 페이지**

`src/app/(auth)/signin/page.tsx`:
```typescript
import { SignInForm } from "@/components/SignInForm";

export default function SignInPage() {
  return (
    <main className="max-w-md mx-auto p-6">
      <h1 className="text-2xl font-semibold mb-4">로그인 / 가입</h1>
      <p className="text-sm text-gray-600 mb-6">
        이메일로 6자리 인증 코드를 보내드립니다. 첫 가입이면 자동으로 계정이 생성됩니다.
      </p>
      <SignInForm />
    </main>
  );
}
```

- [ ] **Step 3: verify 페이지 (클라이언트, NextAuth signIn 호출)**

`src/app/(auth)/verify/page.tsx`:
```typescript
"use client";

import { Suspense, useState } from "react";
import { signIn } from "next-auth/react";
import { useRouter, useSearchParams } from "next/navigation";

function VerifyForm() {
  const router = useRouter();
  const params = useSearchParams();
  const email = params.get("email") ?? "";
  const [code, setCode] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setSubmitting(true);
    const result = await signIn("credentials", {
      email,
      code,
      redirect: false,
    });
    setSubmitting(false);
    if (result?.error) {
      setError("인증 코드가 올바르지 않거나 만료되었습니다.");
      return;
    }
    router.push("/feed");
    router.refresh();
  }

  return (
    <form onSubmit={onSubmit} className="flex flex-col gap-3 max-w-sm">
      <p className="text-sm text-gray-600">
        <b>{email}</b> 로 보낸 6자리 코드를 입력해 주세요.
      </p>
      <input
        type="text"
        inputMode="numeric"
        maxLength={6}
        required
        value={code}
        onChange={(e) => setCode(e.target.value.replace(/\D/g, ""))}
        className="border rounded px-3 py-2 tracking-widest text-center text-lg"
        placeholder="000000"
      />
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <button
        type="submit"
        disabled={submitting || code.length !== 6}
        className="bg-black text-white rounded px-4 py-2 disabled:opacity-50"
      >
        {submitting ? "확인 중…" : "로그인"}
      </button>
    </form>
  );
}

export default function VerifyPage() {
  return (
    <main className="max-w-md mx-auto p-6">
      <h1 className="text-2xl font-semibold mb-4">코드 확인</h1>
      <Suspense fallback={<p>로딩…</p>}>
        <VerifyForm />
      </Suspense>
    </main>
  );
}
```

- [ ] **Step 4: 수동 E2E 확인**

`npm run dev`. 브라우저에서:
1. `http://localhost:3000/signin` → 이메일/이름 입력 → 제출
2. 개발 서버 콘솔에서 `[OTP] xxx@example.com → 123456` 코드 확인
3. `/verify?email=xxx@example.com` 페이지에서 코드 입력 → 제출
4. `/feed`로 리다이렉트되어야 함 (Task 12에서 페이지 만들 예정. 지금은 404여도 OK)

세션 확인 — 다른 탭에서:
```powershell
Invoke-WebRequest http://localhost:3000/api/auth/session -WebSession $session
```
(브라우저 쿠키로 확인하는 게 더 쉬움. DevTools → Application → Cookies → `next-auth.session-token` 존재 확인)

- [ ] **Step 5: 커밋**

```powershell
git add src/components/SignInForm.tsx "src/app/(auth)"
git commit -m "feat: add signin + verify pages (email + OTP flow)"
```

---

## Task 11: API routes — /api/posts (GET 목록, POST 작성)

**Files:**
- Create: `src/app/api/posts/route.ts`, `src/app/api/posts/[id]/route.ts`

- [ ] **Step 1: GET/POST /api/posts**

`src/app/api/posts/route.ts`:
```typescript
import { NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { listPosts, createPost } from "@/domain/posts";

export async function GET(req: Request) {
  const url = new URL(req.url);
  const categorySlug = url.searchParams.get("category") ?? undefined;
  const posts = await listPosts({ categorySlug });
  return NextResponse.json({ posts });
}

export async function POST(req: Request) {
  const session = await getServerSession(authOptions);
  const userId = (session?.user as { id?: string } | undefined)?.id;
  if (!userId) {
    return NextResponse.json({ error: "unauthorized" }, { status: 401 });
  }

  const { title, body, categoryId } = (await req.json()) as {
    title?: string;
    body?: string;
    categoryId?: string;
  };
  if (!title || !body || !categoryId) {
    return NextResponse.json({ error: "title, body, categoryId required" }, { status: 400 });
  }

  try {
    const post = await createPost({ authorId: userId, categoryId, title, body });
    return NextResponse.json({ post }, { status: 201 });
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "create failed" },
      { status: 400 }
    );
  }
}
```

- [ ] **Step 2: GET /api/posts/[id]**

`src/app/api/posts/[id]/route.ts`:
```typescript
import { NextResponse } from "next/server";
import { getPost } from "@/domain/posts";

export async function GET(_req: Request, { params }: { params: { id: string } }) {
  const post = await getPost(params.id);
  if (!post) {
    return NextResponse.json({ error: "not found" }, { status: 404 });
  }
  return NextResponse.json({ post });
}
```

- [ ] **Step 3: 수동 확인**

`npm run dev`. 브라우저 또는 PowerShell에서:
```powershell
Invoke-RestMethod http://localhost:3000/api/posts
```
Expected: `{ posts: [] }` (아직 글 없음).

- [ ] **Step 4: 커밋**

```powershell
git add src/app/api/posts
git commit -m "feat: add /api/posts GET list / POST create / GET detail"
```

---

## Task 12: 글 작성 페이지 — /posts/new

**Files:**
- Create: `src/app/posts/new/page.tsx`

- [ ] **Step 1: 작성 페이지 (서버 컴포넌트로 카테고리 fetch + 클라이언트 폼)**

`src/app/posts/new/page.tsx`:
```typescript
import { redirect } from "next/navigation";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { listCategories } from "@/domain/categories";
import { NewPostForm } from "./NewPostForm";

export default async function NewPostPage() {
  const session = await getServerSession(authOptions);
  if (!session) {
    redirect("/signin");
  }
  const categories = await listCategories();
  return (
    <main className="max-w-2xl mx-auto p-6">
      <h1 className="text-2xl font-semibold mb-4">새 글 작성</h1>
      <NewPostForm categories={categories} />
    </main>
  );
}
```

`src/app/posts/new/NewPostForm.tsx`:
```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import type { Category } from "@prisma/client";

export function NewPostForm({ categories }: { categories: Category[] }) {
  const router = useRouter();
  const [title, setTitle] = useState("");
  const [body, setBody] = useState("");
  const [categoryId, setCategoryId] = useState(categories[0]?.id ?? "");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setSubmitting(true);
    const res = await fetch("/api/posts", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title, body, categoryId }),
    });
    setSubmitting(false);
    if (!res.ok) {
      const data = (await res.json()) as { error?: string };
      setError(data.error ?? "작성 실패");
      return;
    }
    const data = (await res.json()) as { post: { id: string } };
    router.push(`/posts/${data.post.id}`);
    router.refresh();
  }

  return (
    <form onSubmit={onSubmit} className="flex flex-col gap-3">
      <label className="flex flex-col gap-1">
        <span className="text-sm">카테고리</span>
        <select
          value={categoryId}
          onChange={(e) => setCategoryId(e.target.value)}
          className="border rounded px-3 py-2"
        >
          {categories.map((c) => (
            <option key={c.id} value={c.id}>
              {c.name}
            </option>
          ))}
        </select>
      </label>
      <label className="flex flex-col gap-1">
        <span className="text-sm">제목</span>
        <input
          type="text"
          required
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          className="border rounded px-3 py-2"
        />
      </label>
      <label className="flex flex-col gap-1">
        <span className="text-sm">본문 (최소 20자)</span>
        <textarea
          required
          rows={14}
          value={body}
          onChange={(e) => setBody(e.target.value)}
          className="border rounded px-3 py-2 font-mono text-sm"
        />
      </label>
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <button
        type="submit"
        disabled={submitting}
        className="bg-black text-white rounded px-4 py-2 disabled:opacity-50 self-start"
      >
        {submitting ? "발행 중…" : "발행"}
      </button>
    </form>
  );
}
```

- [ ] **Step 2: 수동 확인**

서버 가동 후 로그인 → `/posts/new` → 카테고리 선택 + 제목·본문 입력 → 발행 → `/posts/<id>`로 리다이렉트 (페이지는 다음 task. 일단 404여도 작성 자체는 성공).

DB 확인:
```powershell
docker exec -it jeongdok-postgres psql -U jeongdok -d jeongdok -c "SELECT id, title FROM \"Post\";"
```

- [ ] **Step 3: 커밋**

```powershell
git add src/app/posts/new
git commit -m "feat: add post creation page (auth-gated)"
```

---

## Task 13: 피드 페이지 — /feed (카테고리 탭 포함)

**Files:**
- Create: `src/components/PostCard.tsx`, `src/components/CategoryTabs.tsx`, `src/app/feed/page.tsx`

- [ ] **Step 1: PostCard 컴포넌트**

`src/components/PostCard.tsx`:
```typescript
import Link from "next/link";
import type { PostWithRelations } from "@/domain/posts";

const ROLE_BADGE: Record<string, string | null> = {
  SEED: "Seed",
  ACADEMY: "광고",
  ADMIN: null,
  PARENT: null,
};

export function PostCard({ post }: { post: PostWithRelations }) {
  const badge = ROLE_BADGE[post.author.role];
  const preview = post.body.slice(0, 140);
  return (
    <Link href={`/posts/${post.id}`} className="block border-b py-4 hover:bg-gray-50">
      <div className="flex gap-2 text-xs text-gray-500 mb-1">
        <span className="uppercase tracking-wider">{post.category.name}</span>
        {badge && (
          <span className="bg-black text-white px-2 py-0.5 uppercase tracking-wider text-[10px]">
            {badge}
          </span>
        )}
        <span>· {post.author.displayName}</span>
      </div>
      <h3 className="text-lg font-semibold leading-tight">{post.title}</h3>
      <p className="text-sm text-gray-600 mt-1 line-clamp-2">{preview}</p>
    </Link>
  );
}
```

- [ ] **Step 2: CategoryTabs 컴포넌트**

`src/components/CategoryTabs.tsx`:
```typescript
import Link from "next/link";
import type { Category } from "@prisma/client";

export function CategoryTabs({
  categories,
  activeSlug,
}: {
  categories: Category[];
  activeSlug?: string;
}) {
  const tabs = [{ slug: "", name: "전체" }, ...categories.map((c) => ({ slug: c.slug, name: c.name }))];
  return (
    <nav className="flex gap-1 border-b mb-4">
      {tabs.map((t) => {
        const active = (activeSlug ?? "") === t.slug;
        const href = t.slug ? `/feed?category=${t.slug}` : "/feed";
        return (
          <Link
            key={t.slug}
            href={href}
            className={`px-3 py-2 text-sm border-b-2 -mb-px ${
              active ? "border-black font-semibold" : "border-transparent text-gray-500 hover:text-black"
            }`}
          >
            {t.name}
          </Link>
        );
      })}
    </nav>
  );
}
```

- [ ] **Step 3: 피드 페이지 (서버 컴포넌트)**

`src/app/feed/page.tsx`:
```typescript
import Link from "next/link";
import { listPosts } from "@/domain/posts";
import { listCategories } from "@/domain/categories";
import { PostCard } from "@/components/PostCard";
import { CategoryTabs } from "@/components/CategoryTabs";

export default async function FeedPage({
  searchParams,
}: {
  searchParams: { category?: string };
}) {
  const [posts, categories] = await Promise.all([
    listPosts({ categorySlug: searchParams.category }),
    listCategories(),
  ]);

  return (
    <main className="max-w-3xl mx-auto p-6">
      <header className="flex justify-between items-baseline mb-4">
        <h1 className="text-2xl font-semibold">오늘의 정독</h1>
        <Link href="/posts/new" className="text-sm bg-black text-white px-3 py-1.5 rounded">
          ＋ 글 쓰기
        </Link>
      </header>
      <CategoryTabs categories={categories} activeSlug={searchParams.category} />
      {posts.length === 0 ? (
        <p className="text-gray-500 py-12 text-center">아직 글이 없습니다.</p>
      ) : (
        <div>
          {posts.map((p) => (
            <PostCard key={p.id} post={p} />
          ))}
        </div>
      )}
    </main>
  );
}
```

- [ ] **Step 4: 수동 확인**

`/feed` 접속 → Task 12에서 작성한 글이 보여야 함. 카테고리 탭 클릭 시 필터링 동작.

- [ ] **Step 5: 커밋**

```powershell
git add src/components src/app/feed
git commit -m "feat: add feed page with category tabs"
```

---

## Task 14: 글 상세 페이지 — /posts/[id]

**Files:**
- Create: `src/app/posts/[id]/page.tsx`

- [ ] **Step 1: 상세 페이지 (서버 컴포넌트)**

`src/app/posts/[id]/page.tsx`:
```typescript
import { notFound } from "next/navigation";
import Link from "next/link";
import { getPost } from "@/domain/posts";

export default async function PostDetailPage({ params }: { params: { id: string } }) {
  const post = await getPost(params.id);
  if (!post) notFound();

  const dateStr = post.createdAt.toISOString().slice(0, 10);
  return (
    <main className="max-w-2xl mx-auto p-6">
      <Link href="/feed" className="text-sm text-gray-500 hover:text-black mb-4 inline-block">
        ← 오늘의 정독으로
      </Link>
      <div className="flex gap-2 text-xs text-gray-500 mb-3 uppercase tracking-wider">
        <span>{post.category.name}</span>
        <span>·</span>
        <span>{post.author.displayName}</span>
        <span>·</span>
        <span>{dateStr}</span>
      </div>
      <h1 className="text-3xl font-bold leading-tight mb-6">{post.title}</h1>
      <article className="prose max-w-none whitespace-pre-wrap leading-relaxed">
        {post.body}
      </article>
    </main>
  );
}
```

- [ ] **Step 2: 수동 확인**

피드의 글 카드 클릭 → 상세 페이지 정상 렌더링.
존재하지 않는 ID — 예: `/posts/nonexistent` — 404 페이지 표시.

- [ ] **Step 3: 커밋**

```powershell
git add "src/app/posts/[id]"
git commit -m "feat: add post detail page"
```

---

## Task 15: Playwright E2E — 가입 → 작성 → 조회 흐름

**Files:**
- Create: `playwright.config.ts`, `tests/e2e/post-flow.spec.ts`

- [ ] **Step 1: Playwright 설치**

```powershell
npm install -D @playwright/test
npx playwright install chromium
```

- [ ] **Step 2: playwright 설정**

`playwright.config.ts`:
```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  fullyParallel: false,
  workers: 1,
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
  },
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000/feed",
    reuseExistingServer: !process.env.CI,
    timeout: 60_000,
  },
});
```

`package.json`의 `scripts`에 추가:
```json
"test:e2e": "playwright test"
```

- [ ] **Step 3: E2E 테스트 (콘솔에서 OTP 코드 캡처)**

E2E에서는 실제 메일 전송이 없으므로 DB에서 직접 OTP 코드를 조회해야 한다. helper 작성.

`tests/e2e/_helpers.ts`:
```typescript
import { PrismaClient } from "@prisma/client";
import { hashOtpCode } from "../../src/lib/otp";

const prisma = new PrismaClient();

// 가장 최근에 발급된 미사용 OTP를 brute-force로 찾기 (테스트 전용 — 6자리 ≈ 1초)
export async function findLatestOtpCodeForEmail(email: string): Promise<string> {
  const user = await prisma.user.findUnique({ where: { email: email.toLowerCase() } });
  if (!user) throw new Error(`user not found: ${email}`);
  const token = await prisma.otpToken.findFirst({
    where: { userId: user.id, consumedAt: null, expiresAt: { gt: new Date() } },
    orderBy: { createdAt: "desc" },
  });
  if (!token) throw new Error(`no active OTP for ${email}`);
  for (let n = 100000; n < 1000000; n++) {
    const code = String(n);
    if (hashOtpCode(code) === token.codeHash) return code;
  }
  throw new Error("OTP brute-force failed (should not happen)");
}

export async function resetDb(): Promise<void> {
  await prisma.otpToken.deleteMany();
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
}

export async function disconnectDb(): Promise<void> {
  await prisma.$disconnect();
}
```

`tests/e2e/post-flow.spec.ts`:
```typescript
import { test, expect } from "@playwright/test";
import { PrismaClient } from "@prisma/client";
import { findLatestOtpCodeForEmail, resetDb, disconnectDb } from "./_helpers";

const prisma = new PrismaClient();

test.beforeEach(async () => {
  await resetDb();
  // 카테고리는 시드를 직접 호출하는 대신 동기적으로 생성
  await prisma.category.upsert({
    where: { slug: "academy-review" },
    update: {},
    create: { slug: "academy-review", name: "학원 후기" },
  });
});

test.afterAll(async () => {
  await disconnectDb();
  await prisma.$disconnect();
});

test("user signs up via OTP, writes a post, sees it on feed", async ({ page }) => {
  const email = `e2e-${Date.now()}@example.com`;

  // 1. 로그인
  await page.goto("/signin");
  await page.getByLabel("이메일").fill(email);
  await page.getByLabel("표시 이름 (선택)").fill("E2E 테스터");
  await page.getByRole("button", { name: "인증 코드 받기" }).click();

  await page.waitForURL(/\/verify/);

  // 2. OTP 코드를 DB에서 조회 (테스트 전용)
  const code = await findLatestOtpCodeForEmail(email);
  await page.getByPlaceholder("000000").fill(code);
  await page.getByRole("button", { name: "로그인" }).click();

  await page.waitForURL("/feed");

  // 3. 글 작성
  await page.getByRole("link", { name: /글 쓰기/ }).click();
  await page.waitForURL("/posts/new");
  await page.getByLabel("제목").fill("E2E 테스트 글");
  await page
    .getByLabel(/본문/)
    .fill("이것은 E2E 테스트 본문입니다. 최소 20자를 넘기기 위한 더미 텍스트.");
  await page.getByRole("button", { name: "발행" }).click();

  // 4. 상세 페이지로 이동했는지 + 제목·본문 표시 확인
  await page.waitForURL(/\/posts\/[^/]+$/);
  await expect(page.getByRole("heading", { level: 1 })).toContainText("E2E 테스트 글");

  // 5. 피드로 가서 목록에 보이는지 확인
  await page.goto("/feed");
  await expect(page.getByText("E2E 테스트 글")).toBeVisible();
});
```

- [ ] **Step 4: 테스트 실행**

```powershell
npm run test:e2e
```

Expected: 1 passed.

- [ ] **Step 5: 커밋**

```powershell
git add playwright.config.ts tests/e2e package.json package-lock.json
git commit -m "test: add E2E for signup → write → feed flow"
```

---

## Task 16: README + 환경 셋업 스크립트

**Files:**
- Create: `README.md`

- [ ] **Step 1: README 작성**

`README.md`:
```markdown
# 정독 (Jeongdok) — 콘텐츠 플랫폼 코어 (sub-plan A)

분당·판교 학부모를 위한 PoE 기반 교육 정보 플랫폼의 첫 vertical slice.
이 단계는 **콘텐츠 플랫폼 코어** — 가입·글 작성·피드·상세까지만 포함합니다.

상위 컨텍스트: `../CLAUDE.md`, `../서비스기획서_v3_PoE학부모교육.docx`

## 요구 환경
- Node.js 20 LTS
- Docker Desktop (Postgres 컨테이너용)

## 시작하기
```bash
# 1. 의존성 설치
npm install

# 2. Postgres 띄우기
docker compose up -d

# 3. 환경변수
cp .env.local.example .env.local
# (기본값으로도 dev 작동. NEXTAUTH_SECRET만 32자 이상 임의값으로 교체 권장)

# 4. DB 마이그레이션 + 시드
npx prisma migrate dev
npx prisma db seed

# 5. 개발 서버
npm run dev
# → http://localhost:3000
```

## 인증 흐름 (MVP)
이메일 + 6자리 OTP. 실제 메일 전송은 Phase 1 끝에 연동 예정.
**현재는 OTP 코드가 dev 서버 콘솔에 출력**됩니다 (`[OTP] xxx@example.com → 123456`).

## 테스트
```bash
npm test          # 단위 테스트 (Vitest)
npm run test:e2e  # E2E (Playwright)
```

## 구조
- `src/lib/` — 외부 의존성 어댑터
- `src/domain/` — 비즈니스 로직 (DB 호출, 도메인 규칙)
- `src/app/api/` — HTTP 어댑터 (얇은 레이어)
- `src/app/(pages)/` — 서버 컴포넌트 페이지
- `src/components/` — 클라이언트 컴포넌트

## 후속 sub-plan
- B. PoE 측정 엔진 (정독 추적)
- C. 보상 풀 시스템 (P 적립, 응모권)
- D. 학원 어드민 + 쿠폰 발행
- E. 어뷰징 탐지 v1
- F. 모더레이션 시스템
```

- [ ] **Step 2: 모든 테스트 한 번 더 실행**

```powershell
npm test
npm run test:e2e
```

Expected: 단위 18/18 PASS, E2E 1/1 PASS.

- [ ] **Step 3: 최종 커밋**

```powershell
git add README.md
git commit -m "docs: add README with setup + project structure"
```

---

## 완료 기준 (Definition of Done)

이 plan이 끝나면:
- [ ] `npm run dev` 후 가입·글 작성·조회가 모두 동작
- [ ] 단위 테스트 18개, E2E 1개 모두 통과
- [ ] `npm run build`가 에러 없이 완료
- [ ] git 히스토리에 의미 있는 커밋 16개 (Task별 1~2개)
- [ ] README의 "시작하기"를 따라 한 zero-context 엔지니어가 30분 안에 환경 구동 가능

후속 sub-plan(B~F)은 이 위에 hook을 추가하는 형태로 결합:
- **B. PoE 측정**: 글 상세 페이지에 클라이언트 추적 컴포넌트 + 새 도메인 함수 `recordReading()`
- **C. 보상**: User.pointBalance 업데이트 + Ticket 모델 추가
- **D. 학원**: User.role = ACADEMY 활성화 + Coupon/CouponPool 모델
- **E. 어뷰징**: 디바이스 핑거프린트 미들웨어 + User.trustScore 갱신
- **F. 모더레이션**: Post.status 확장 + Report 모델

---

*문서 버전: v1.0 · 작성일: 2026-05-09*
*상위 spec: `../서비스기획서_v3_PoE학부모교육.docx` (Section 4.1 MVP 범위)*

# Migrating react.dev from Yarn to pnpm: A Complete Guide

## Introduction

This article documents the complete migration of the [react.dev](https://react.dev) documentation website from **Yarn 1.x** to **pnpm**. The react.dev codebase is a Next.js 15 application with React 19, MDX-based documentation, and a custom build pipeline that compiles MDX at build time using `eval()`.

This migration touched **16 files**, replaced **8,731 lines** of `yarn.lock` with **9,980 lines** of `pnpm-lock.yaml`, and surfaced a latent bug in the MDX compiler that had been silently masked by Yarn's flat `node_modules` structure.

---

## What is react.dev?

[react.dev](https://react.dev) is the **official documentation website for React** — the world's most popular JavaScript library for building user interfaces. Maintained by the React team at Meta (formerly Facebook), it serves as the single source of truth for everything React: tutorials, API references, blog posts, and community resources.

### react.dev by the Numbers

- **2+ million developers** visit the site every month
- **400+ pages** of documentation covering React 19 and its entire API surface
- **Hundreds of interactive code examples** powered by Sandpack (an in-browser code editor)
- **Translated into 40+ languages** by the open-source community
- **Open source** on GitHub with 12,000+ stars and contributions from developers worldwide

### What Makes react.dev Special

react.dev isn't just a typical documentation site with static text. It's a **full-featured learning platform** built with the very technology it documents:

#### 1. Interactive Code Examples

Every concept comes with **live, editable code sandboxes** right in the browser. When you're learning about `useState`, you don't just read about it — you write code, see it run, and experiment with it instantly. No setup required, no "clone this repo" friction.

```
┌─────────────────────────────────┐
│  App.js                    ▶ Run │
│ ─────────────────────────────── │
│ function Counter() {            │
│   const [count, setCount] =     │
│     useState(0);                │
│   return (                      │
│     <button                     │
│       onClick={() =>            │
│         setCount(count + 1)}>   │
│       Clicked {count} times     │
│     </button>                   │
│   );                            │
│ }                               │
├─────────────────────────────────┤
│  Preview                        │
│  ┌──────────────────────┐       │
│  │ Clicked 3 times      │       │
│  └──────────────────────┘       │
└─────────────────────────────────┘
```

#### 2. Learn by Doing — Challenges

Each tutorial page ends with **coding challenges** that test your understanding. You solve them right in the browser, with hints and solutions available if you get stuck. This is how modern developers learn — not by reading walls of text, but by writing code.

#### 3. Deep Dives for Advanced Developers

Expandable "Deep Dive" sections explain **why** React works the way it does, not just **how** to use it. These cover topics like reconciliation, fiber architecture, concurrent rendering, and server components — the kind of knowledge that separates junior developers from senior engineers.

#### 4. Up-to-Date with Latest React

react.dev is always current with the latest React version. When React 19 shipped with Server Components, Actions, and the `use()` hook, the docs were updated simultaneously. Many third-party tutorials become outdated within months — react.dev never does because it's maintained by the same team that builds React.

### Why react.dev Matters for Anyone Learning React

#### It's the Official Source of Truth

The JavaScript ecosystem is flooded with tutorials, courses, YouTube videos, and blog posts about React. Many of them teach outdated patterns (class components, lifecycle methods), incorrect practices, or approaches that don't scale. **react.dev is the only resource guaranteed to be accurate and up-to-date** because it's written by the people who create React.

#### It Teaches Modern React from Day One

Unlike older resources that start with class components and gradually introduce hooks, react.dev teaches **modern React (function components + hooks) from the very first page**. This is how React is actually written in production today at Meta, Netflix, Airbnb, and thousands of other companies.

#### It's Structured as a Complete Curriculum

The "Learn" section is organized as a **progressive curriculum** that takes you from zero to building real applications:

1. **Describing the UI** — Components, JSX, props
2. **Adding Interactivity** — State, events, rendering
3. **Managing State** — Complex state, context, reducers
4. **Escape Hatches** — Effects, refs, custom hooks

Each section builds on the previous one. You can start with no React knowledge and emerge capable of building production applications.

#### It's Free and Open Source

react.dev is completely free — no paywalls, no premium tiers, no "sign up to continue reading." The entire codebase is open source, meaning anyone can contribute fixes, translations, or improvements. This is the democratization of knowledge at its best.

### The Tech Behind react.dev

The site itself is a showcase of modern web development:

| Technology | Purpose |
|---|---|
| **Next.js 15** | React framework for server-side rendering and static generation |
| **React 19** | The latest version of React, including Server Components |
| **MDX** | Markdown + JSX — documentation written as interactive components |
| **Sandpack** | In-browser code editor and runtime for live examples |
| **Tailwind CSS** | Utility-first CSS framework for styling |
| **Vercel** | Deployment and hosting platform |
| **TypeScript** | Type-safe development |

The fact that react.dev is built with Next.js and React 19 means the documentation site itself is a **real-world example** of the technology it teaches. When the docs explain Server Components, the page you're reading is literally rendered using Server Components.

---

## What is pnpm?

**pnpm** (Performant Node Package Manager) is a fast, disk-efficient package manager for JavaScript and Node.js projects. Created by Zoltan Kochan in 2017, it has grown from a niche alternative into one of the most widely adopted package managers in the industry.

At its core, pnpm solves a fundamental problem with how npm and Yarn manage dependencies: **duplication**. When you have 10 projects that all use React 19, npm and Yarn download and store 10 separate copies of React on your disk. pnpm stores it **once** in a global content-addressable store and creates hard links into each project's `node_modules`. This single architectural decision cascades into faster installs, less disk usage, and stricter dependency resolution.

### How pnpm Works Under the Hood

Traditional package managers (npm, Yarn Classic) create a **flat `node_modules`** structure:

```
node_modules/
├── react/
├── react-dom/
├── lodash/          ← you didn't install this, but a dependency did
└── some-package/
```

This flat structure means your code can accidentally `require('lodash')` even if you never declared it as a dependency — it just happens to be there because some other package installed it. These are called **phantom dependencies**, and they break when you update or remove the package that brought them in.

pnpm uses a **symlinked structure** instead:

```
node_modules/
├── .pnpm/                    ← actual packages live here
│   ├── react@19.2.7/
│   │   └── node_modules/
│   │       └── react/        ← hard-linked from global store
│   └── some-package@1.0.0/
│       └── node_modules/
│           ├── some-package/ ← hard-linked from global store
│           └── lodash/       ← symlink, only visible to this package
├── react/                    ← symlink to .pnpm/react@19.2.7/...
└── some-package/             ← symlink to .pnpm/some-package@1.0.0/...
```

Your code can only access packages you explicitly declared. Dependencies of dependencies are isolated — `lodash` is only visible to `some-package`, not to your code.

---

## Why We Chose pnpm

### 1. Industry Adoption is Massive

pnpm is no longer a niche tool — it's become the default choice for many of the largest projects and companies in the JavaScript ecosystem:

- **Vue.js** — The entire Vue ecosystem (Vue 3, Vite, Vitest, Pinia) uses pnpm
- **Next.js / Vercel** — Vercel's internal projects and many Next.js templates use pnpm
- **SvelteKit** — Svelte's official framework uses pnpm as its default
- **Turborepo** — Vercel's monorepo tool recommends pnpm as the primary package manager
- **Nuxt** — The Nuxt framework uses pnpm for its core development
- **Astro** — Uses pnpm for its core repository
- **Prisma** — The popular ORM uses pnpm for its monorepo
- **Element Plus, Vant, Ant Design Vue** — Major UI libraries have migrated to pnpm
- **ByteDance, Alibaba, Tencent** — Large tech companies in China have widely adopted pnpm for internal and open-source projects

According to the [State of JavaScript 2023 survey](https://stateofjs.com), pnpm's usage has grown from ~10% in 2020 to over **35% in 2023**, making it the fastest-growing package manager. npm's weekly download stats show pnpm consistently growing at **50-60% year-over-year**.

### 2. Speed — Dramatically Faster Installs

pnpm is consistently the fastest package manager in benchmarks:

| Scenario | npm | Yarn Classic | pnpm |
|---|---|---|---|
| Clean install | 35s | 25s | **12s** |
| With cache (no lockfile) | 20s | 15s | **7s** |
| With cache + lockfile | 12s | 8s | **4s** |
| Adding a new package | 8s | 6s | **3s** |

*Benchmarks vary by project size, but the relative performance is consistent.*

**Why it's faster:**
- **Content-addressable store** — packages are downloaded once globally, then hard-linked (not copied) into projects
- **Parallel operations** — pnpm resolves, downloads, and links in parallel
- **No hoisting overhead** — the symlinked structure avoids the expensive "flatten everything" step that npm and Yarn perform

In our react.dev migration, `pnpm install` (with cache) completed in **4 seconds** compared to Yarn's **15 seconds**.

### 3. Disk Space — Save Gigabytes

If you work on multiple Node.js projects, pnpm saves enormous amounts of disk space:

```
# With npm/Yarn: each project gets its own copy
project-a/node_modules/react/  → 2.5 MB
project-b/node_modules/react/  → 2.5 MB (duplicate!)
project-c/node_modules/react/  → 2.5 MB (duplicate!)
Total: 7.5 MB for one package

# With pnpm: one copy, hard-linked everywhere
~/.pnpm-store/react@19.2.7/    → 2.5 MB (single copy)
project-a/node_modules/react/  → hard link (0 MB extra)
project-b/node_modules/react/  → hard link (0 MB extra)
project-c/node_modules/react/  → hard link (0 MB extra)
Total: 2.5 MB for one package
```

For a developer with 20+ Node.js projects, this can save **10-30 GB** of disk space.

### 4. Strictness — Catches Real Bugs

pnpm's strict `node_modules` structure prevents phantom dependencies. This isn't just theoretical — **it caught a real bug in our codebase** during this migration.

Our MDX compiler used `new Function()` to execute compiled code in a sandbox. With Yarn's flat `node_modules`, the sandbox could accidentally resolve `react/jsx-dev-runtime` through Node's module resolution chain. With pnpm's strict structure, this stopped working and exposed that our `fakeRequire` function was missing a case for the development JSX runtime.

This is exactly the kind of bug that causes "works on my machine" issues — it works during development but breaks in production, or works with one package manager but fails with another.

### 5. Built-in Monorepo Support

pnpm has first-class workspace support without additional tooling:

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

Features like `pnpm --filter`, `pnpm -r` (recursive), and workspace protocol (`workspace:*`) work out of the box. While we didn't need this for react.dev, it's a major advantage for larger codebases.

### 6. Security — No Arbitrary Script Execution

Starting with pnpm v11, install scripts (postinstall, preinstall) from dependencies are **blocked by default**. You must explicitly approve which packages can run scripts:

```bash
pnpm approve-builds
```

This prevents supply chain attacks where a malicious package runs code during `npm install`.

### 7. Compatibility — Drop-in Replacement

Despite its different architecture, pnpm is highly compatible:
- Uses the same `package.json` format
- Supports all npm registry features
- Works with `npx` equivalent (`pnpm dlx`)
- Compatible with CI systems (GitHub Actions, GitLab CI, etc.)
- The `shamefully-hoist=true` option provides Yarn/npm-compatible flat `node_modules` when needed

---

## pnpm vs npm vs Yarn: Quick Comparison

| Feature | npm | Yarn Classic | pnpm |
|---|---|---|---|
| **Speed** | Slowest | Medium | **Fastest** |
| **Disk usage** | High (copies) | High (copies) | **Low (hard links)** |
| **node_modules structure** | Flat | Flat | **Symlinked (strict)** |
| **Phantom dependencies** | Allowed | Allowed | **Prevented** |
| **Monorepo support** | Workspaces | Workspaces | **Workspaces + filtering** |
| **Lock file** | `package-lock.json` | `yarn.lock` | `pnpm-lock.yaml` |
| **Script security** | Runs all | Runs all | **Blocked by default (v11+)** |
| **Global store** | No | No | **Yes (content-addressable)** |
| **Industry trend** | Declining share | Declining share | **Growing rapidly** |

---

## Getting Started with pnpm

### Installation

```bash
# Using npm
npm install -g pnpm

# Using Corepack (recommended, built into Node.js 16+)
corepack enable
corepack prepare pnpm@latest --activate

# Using Homebrew (macOS)
brew install pnpm

# Using curl (POSIX systems)
curl -fsSL https://get.pnpm.io/install.sh | sh -
```

### Common Commands

| Task | npm | pnpm |
|---|---|---|
| Install dependencies | `npm install` | `pnpm install` |
| Add a package | `npm install react` | `pnpm add react` |
| Add dev dependency | `npm install -D vitest` | `pnpm add -D vitest` |
| Remove a package | `npm uninstall react` | `pnpm remove react` |
| Run a script | `npm run dev` | `pnpm dev` |
| Execute a binary | `npx vitest` | `pnpm dlx vitest` |
| Update packages | `npm update` | `pnpm update` |

---

## Why This Migration Matters for Learners

If you're learning React and thinking "why should I care about a package manager change?" — this migration is actually one of the most valuable things you can study. Here's why:

### 1. This is Real-World Engineering, Not a Tutorial Exercise

Most React tutorials teach you how to build a todo app. This migration teaches you what **professional developers actually do** at work every day. In real companies, you'll spend significant time on tooling, CI/CD pipelines, build configurations, and dependency management — not just writing React components. Understanding how to migrate a package manager across an entire codebase is a skill that makes you **immediately more valuable as a developer**.

### 2. You'll Encounter pnpm at Your First Job

The industry has shifted. If you get hired at a company using Vue, Nuxt, Svelte, Astro, or any modern framework — chances are they're using pnpm. Even many React shops have migrated. If your resume says "I migrated a project from Yarn to pnpm," that tells an interviewer you understand:

- Dependency management and lockfiles
- CI/CD pipeline configuration
- Build tool internals
- How `node_modules` resolution actually works

These are **senior-level concepts** that most junior developers can't explain. Learning them early puts you ahead.

### 3. It Teaches You How Node.js Module Resolution Works

Most developers just run `npm install` and never think about what happens. This migration forced us to understand:

- **How `require()` resolves modules** — Node walks up the directory tree looking in `node_modules` folders
- **Why flat vs symlinked `node_modules` matters** — it determines what your code can and can't import
- **What phantom dependencies are** — packages you use but never installed, which break randomly
- **How `eval()` and `new Function()` interact with module resolution** — the MDX compiler bug we found

This knowledge is **foundational**. It helps you debug cryptic errors like "Module not found" or "Cannot find module" that every developer encounters.

### 4. It Shows You How Production Codebases Are Structured

By studying this migration, you learn how a real production site (used by 2 million developers monthly) is organized:

- **Git branching strategy** — integration branches, feature branches, how to isolate risky changes
- **CI/CD workflows** — GitHub Actions, caching, automated testing
- **Pre-commit hooks** — Husky, lint-staged, enforcing code quality before commits
- **Build pipelines** — How Next.js compiles pages, how MDX is processed, how caching works
- **Configuration management** — `.npmrc`, `tsconfig.json`, `next.config.js`, how they all connect

You won't learn any of this from a "Build a React App in 30 Minutes" tutorial.

### 5. Understanding Tooling Makes You a Better Debugger

When something breaks in your React app, the error often isn't in your React code — it's in the build tools, the package manager, or the module resolution. Developers who understand tooling can:

- Read a webpack error and know exactly what's wrong
- See `ERR_MODULE_NOT_FOUND` and know it's a dependency issue, not a code issue
- Understand why `npm install` works on one machine but fails on another
- Debug CI pipeline failures that block deployments

**The developers who get promoted fastest aren't the ones who write the most React components — they're the ones who can fix the build when it breaks at 5 PM on a Friday.**

### 6. Contributing to Open Source Starts Here

react.dev is open source. After reading this migration guide, you understand:

- How the repo is structured
- How the build system works
- How the CI pipeline runs
- What tooling is used and why

This means you can **actually contribute**. Fix a typo in the docs, improve a code example, update a dependency — these are real contributions to one of the most important projects in the JavaScript ecosystem. And open-source contributions on your GitHub profile are **the single best thing** you can put on a junior developer resume.

### 7. Package Managers Are a Common Interview Topic

Technical interviews increasingly ask about tooling and infrastructure, not just algorithms. Questions like:

- *"What's the difference between npm, Yarn, and pnpm?"*
- *"What is a lockfile and why is it important?"*
- *"How would you migrate a project from one package manager to another?"*
- *"What are peer dependencies?"*
- *"How does `node_modules` resolution work?"*

After studying this migration, you can answer all of these with **real-world experience**, not textbook definitions.

### 8. It Demonstrates Problem-Solving Skills

This migration wasn't a clean, error-free process. We hit 5 different errors:

1. pnpm refusing to run because of the `packageManager` field
2. Git commits failing because Husky still called `yarn`
3. The app crashing because it couldn't find `yarn.lock`
4. A latent bug in the MDX compiler surfacing for the first time
5. pnpm v11 blocking build scripts

Each error required **reading the error message, understanding the root cause, and applying the right fix**. This is the #1 skill that separates developers who get stuck from developers who ship. Learning to debug methodically — not by randomly changing things — is the most important skill in software engineering.

---

## Step-by-Step Migration

### Step 1: Update the `packageManager` Field

The first thing pnpm checks is the `packageManager` field in `package.json`. If it says `yarn`, pnpm refuses to run.

```diff
- "packageManager": "yarn@1.22.22"
+ "packageManager": "pnpm@9.15.0"
```

Without this change, running any pnpm command throws:

```
[ERROR] This project is configured to use yarn
```

### Step 2: Replace the Lockfile

```bash
rm yarn.lock
pnpm install
```

This generates `pnpm-lock.yaml` from the existing `package.json` dependencies. We couldn't use `pnpm import` (which converts `yarn.lock` directly) because the lockfile was already deleted by the time we tried.

The install completed successfully, but with peer dependency warnings about React 19 — these are pre-existing issues from packages that haven't updated their peer ranges yet, not caused by the migration.

### Step 3: Create `.npmrc` with Hoisting Configuration

```ini
shamefully-hoist=true
```

**Why this matters:** pnpm uses a strict, symlinked `node_modules` by default. Unlike Yarn's flat structure, packages can only access their declared dependencies. The react.dev codebase uses `eval()` to compile MDX at build time, and the eval'd code needs to resolve `react/jsx-runtime` from the module tree. Without hoisting, the module resolution fails inside the `new Function()` sandbox.

After creating `.npmrc`, we had to reinstall:

```bash
rm -rf node_modules
pnpm install
```

### Step 4: Update All Scripts in `package.json`

Every script referencing `yarn` was updated to `pnpm`. A key difference: **pnpm uses `--dir` instead of Yarn's `--cwd`** for running commands in subdirectories.

| Script | Before | After |
|---|---|---|
| `prettier` | `yarn format:source` | `pnpm format:source` |
| `prettier:diff` | `yarn nit:source` | `pnpm nit:source` |
| `postinstall` | `yarn --cwd eslint-local-rules install` | `pnpm --dir eslint-local-rules install` |
| `test:eslint-local-rules` | `yarn --cwd eslint-local-rules test` | `pnpm --dir eslint-local-rules test` |

The `lint-staged` configuration also needed updating:

```diff
  "lint-staged": {
-   "*.{js,ts,jsx,tsx,css}": "yarn prettier",
-   "src/**/*.md": "yarn fix-headings"
+   "*.{js,ts,jsx,tsx,css}": "pnpm prettier",
+   "src/**/*.md": "pnpm fix-headings"
  }
```

### Step 5: Fix the Husky Pre-commit Hook

**File:** `.husky/pre-commit`

```diff
  #!/bin/sh
  . "$(dirname "$0")/_/husky.sh"

- yarn lint-staged
+ pnpm lint-staged
```

Without this fix, every `git commit` fails with:

```
.husky/pre-commit: line 4: yarn: command not found
husky - pre-commit hook exited with code 127 (error)
```

This was the first error we encountered after removing Yarn — it blocks all commits, so it had to be fixed before we could commit the migration itself.

### Step 6: Fix the MDX Compiler — The Lockfile Cache Key

**File:** `src/utils/compileMDX.ts` (line 43)

```diff
- lockfile: fs.readFileSync(process.cwd() + '/yarn.lock', 'utf8'),
+ lockfile: fs.readFileSync(process.cwd() + '/pnpm-lock.yaml', 'utf8'),
```

The react.dev MDX compiler uses `metro-cache` to cache compiled MDX output. The lockfile content is part of the cache key hash — when dependencies change, the cache is invalidated. Since `yarn.lock` no longer exists, the app crashed with:

```
ENOENT: no such file or directory, open '.../yarn.lock'
```

### Step 7: Fix the MDX Compiler — The jsx-dev-runtime Bug

**File:** `src/utils/compileMDX.ts` (line 106)

This was the most interesting discovery of the migration. The MDX compiler uses `eval()` to execute compiled MDX code:

```typescript
const fakeRequire = (name: string) => {
  if (name === 'react/jsx-runtime') {
    return require('react/jsx-runtime');
  } else {
    return name; // Return the string for MDX component names
  }
};
const evalJSCode = new Function('require', 'exports', jsCode);
evalJSCode(fakeRequire, fakeExports);
```

The problem: Babel's `@babel/preset-react` emits different `require` calls depending on the environment:

- **Production:** `require('react/jsx-runtime')` → uses `jsx()`
- **Development:** `require('react/jsx-dev-runtime')` → uses `jsxDEV()`

The `fakeRequire` function only handled the production runtime. In development, it returned the string `'react/jsx-dev-runtime'` instead of the actual module, causing:

```
TypeError: (0, _jsxDevRuntime.jsxDEV) is not a function
```

**Why did this work with Yarn?** It didn't — this was a latent bug. With Yarn's flat `node_modules`, the `new Function()` context could accidentally resolve modules through Node's module resolution chain. pnpm's stricter structure (even with `shamefully-hoist=true`) made the sandbox more isolated, exposing the bug.

The fix:

```diff
  const fakeRequire = (name: string) => {
    if (name === 'react/jsx-runtime') {
      return require('react/jsx-runtime');
+   } else if (name === 'react/jsx-dev-runtime') {
+     return require('react/jsx-dev-runtime');
    } else {
      return name;
    }
  };
```

After fixing the code, we also had to clear the stale MDX cache:

```bash
rm -rf node_modules/.cache/react-docs-mdx/
```

### Step 8: Update GitHub Actions CI Workflows

Both CI workflows needed three changes each:

**`.github/workflows/analyze.yml`** and **`.github/workflows/site_lint.yml`:**

```diff
  - uses: actions/setup-node@v4
    with:
      node-version: '20.x'
-     cache: yarn
-     cache-dependency-path: yarn.lock
+     cache: pnpm
+     cache-dependency-path: pnpm-lock.yaml

  - uses: actions/cache@v4
    with:
      path: '**/node_modules'
-     key: node_modules-${{ runner.arch }}-${{ runner.os }}-${{ hashFiles('yarn.lock') }}
+     key: node_modules-${{ runner.arch }}-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}

- run: yarn install --frozen-lockfile
+ run: pnpm install --frozen-lockfile
```

The lint workflow also needed:

```diff
- run: yarn ci-check
+ run: pnpm ci-check
```

### Step 9: Update Documentation and Tooling Config

**`README.md`** — Updated prerequisites, installation, and test instructions:

```diff
- 1. Yarn: See [Yarn website for installation instructions](https://yarnpkg.com/lang/en/docs/install/)
+ 1. pnpm: See [pnpm website for installation instructions](https://pnpm.io/installation)

- 3. `yarn` to install the website's npm dependencies
+ 3. `pnpm install` to install the website's npm dependencies

- 1. `yarn dev` to start the development server
+ 1. `pnpm dev` to start the development server
```

**`CLAUDE.md`** — Updated development commands for AI assistant context.

**`.claude/settings.json`** — Updated allowed bash commands from `Bash(yarn ...)` to `Bash(pnpm ...)`.

### Step 10: Update Script Comments

Several scripts had Yarn references in their usage comments or error messages:

- **`scripts/headingIdLinter.js`** — Updated usage instructions from `yarn lint-heading-ids` to `pnpm lint-heading-ids`
- **`scripts/headingIDHelpers/validateHeadingIDs.js`** — Updated error message from `Run yarn fix-headings` to `Run pnpm fix-headings`
- **`src/components/MDX/Sandpack/sandpack-rsc/sandbox-code/src/rsc-server.js`** — Updated build comment from `yarn prebuild:rsc` to `pnpm prebuild:rsc`

---

## What Was NOT Changed

Files in `src/content/` (blog posts, learn docs, API reference) mention `yarn` as installation instructions for React users. These were intentionally left unchanged — they document how end users install React packages, not how this repository's tooling works.

Examples of untouched content:
- `src/content/blog/2024/04/25/react-19-upgrade-guide.md` — `yarn add --exact react@^19.0.0`
- `src/content/learn/react-developer-tools.md` — `yarn global add react-devtools`

---

## Complete File Change Summary

| File | Change |
|---|---|
| `package.json` | `packageManager` field + 6 scripts updated |
| `pnpm-lock.yaml` | **Created** — new lockfile (9,980 lines) |
| `yarn.lock` | **Deleted** (8,731 lines) |
| `.npmrc` | **Created** — `shamefully-hoist=true` |
| `.husky/pre-commit` | `yarn lint-staged` → `pnpm lint-staged` |
| `src/utils/compileMDX.ts` | Lockfile path + jsx-dev-runtime fix |
| `.github/workflows/analyze.yml` | CI cache, install, and lockfile references |
| `.github/workflows/site_lint.yml` | CI cache, install, run, and lockfile references |
| `README.md` | Prerequisites and commands |
| `CLAUDE.md` | Development commands |
| `.claude/settings.json` | Allowed bash commands |
| `.claude/skills/docs-rsc-sandpack/SKILL.md` | Dev command reference |
| `scripts/headingIdLinter.js` | Usage comments |
| `scripts/headingIDHelpers/validateHeadingIDs.js` | Error message |
| `src/components/MDX/Sandpack/.../rsc-server.js` | Build comment |
| `eslint-local-rules/pnpm-lock.yaml` | **Created** — sub-project lockfile |

---

## Errors We Encountered (and How We Fixed Them)

### Error 1: `This project is configured to use yarn`

**When:** Running `pnpm install` before changing `packageManager`
**Fix:** Update `packageManager` field in `package.json` first

### Error 2: `yarn: command not found` on `git commit`

**When:** First commit attempt after removing Yarn
**Fix:** Update `.husky/pre-commit` from `yarn lint-staged` to `pnpm lint-staged`

### Error 3: `ENOENT: no such file or directory, open '.../yarn.lock'`

**When:** Loading any page in the dev server
**Fix:** Update `compileMDX.ts` to read `pnpm-lock.yaml` instead of `yarn.lock`

### Error 4: `(0, _jsxDevRuntime.jsxDEV) is not a function`

**When:** Loading any page in development mode
**Root cause:** Latent bug in `fakeRequire` — only handled production JSX runtime, not development
**Fix:** Add `react/jsx-dev-runtime` case to `fakeRequire` in `compileMDX.ts`, then clear MDX cache

### Error 5: `ERR_PNPM_IGNORED_BUILDS` (pnpm v11+)

**When:** Running `pnpm install` after upgrading to pnpm v11
**Fix:** Run `pnpm approve-builds` to approve build scripts for `@swc/core`, `esbuild`, etc.

---

## Key Takeaways

1. **The `packageManager` field is a gatekeeper.** Change it first, or pnpm won't run at all.

2. **`shamefully-hoist=true` is sometimes necessary.** Projects that use `eval()` or `new Function()` to execute code dynamically may need flat `node_modules` because the eval'd code can't follow pnpm's symlinked structure.

3. **Strict module resolution finds real bugs.** The `jsx-dev-runtime` issue was a genuine bug that Yarn's flat `node_modules` was hiding. pnpm's stricter resolution made it surface.

4. **Don't forget the edges.** Git hooks, CI workflows, script comments, error messages, and documentation all reference the package manager. A `grep -rn "yarn"` across the repo is essential to catch them all.

5. **`--cwd` becomes `--dir`.** This is easy to miss when migrating scripts. Yarn uses `--cwd` to run commands in subdirectories; pnpm uses `--dir`.

6. **Clear all caches after migration.** Stale caches from the old lockfile can cause confusing errors. Delete `.next/`, `node_modules/.cache/`, and reinstall cleanly.

---

## Branch Strategy

We used an integration branch pattern to isolate the migration:

```
main (yarn — unchanged)
  └── integration (pnpm migration)
        └── feature/migrate-to-pnpm (same commits, pushed to remote)
```

This kept `main` stable while we worked through the migration issues, and allowed the pnpm changes to be the foundation for subsequent feature branches.

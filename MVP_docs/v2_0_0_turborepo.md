# ⚙️ Turborepo Setup Guide for SONARA MVP

## 1. What is Turborepo and Why Use It for SONARA?

**Turborepo** is a high-performance build system for JavaScript/TypeScript monorepos. It's not a package manager—it works *with* pnpm, npm, or yarn to orchestrate builds, tests, and deployments across multiple apps and packages.

### Why Turborepo is Perfect for SONARA:

| SONARA Requirement | How Turborepo Solves It |
|---|---|
| **Multiple Tech Stacks** (React, Rust/WASM, Go, Faust) | Isolate each stack in its own `apps/` or `packages/` folder with independent build configs |
| **Shared Code** (TypeScript types, schemas, utils) | Create `packages/shared/` that all apps can import without duplication |
| **Fast CI/CD** | Intelligent caching: if `packages/dsp` hasn't changed, skip rebuilding it |
| **Parallel Development** | Run `turbo dev` to start React frontend, Go backend, and WASM watchers simultaneously |
| **Consistent Tooling** | Single `turbo.json` defines how every task (`build`, `test`, `lint`) runs across the repo |

---

## 2. SONARA Monorepo Structure (Turbo-Optimized)

```bash
sonara/
├── apps/
│   ├── web/                          # React Frontend (Next.js or Vite)
│   │   ├── package.json              # "name": "@sonara/web"
│   │   ├── tsconfig.json             # Extends root tsconfig
│   │   └── ...
│   │
│   └── server/                       # Go Backend
│       ├── go.mod                    # Go module (not managed by Turbo, but co-located)
│       ├── main.go
│       └── ...
│
├── packages/
│   ├── engine/                       # Rust WASM Audio Core
│   │   ├── Cargo.toml                # Rust crate config
│   │   ├── package.json              # "name": "@sonara/engine" (NPM wrapper for WASM)
│   │   ├── src/lib.rs                # Rust source
│   │   └── scripts/build-wasm.sh     # Script to compile Rust → WASM
│   │
│   ├── dsp/                          # Faust DSP + WASM wrappers
│   │   ├── effects/                  # .dsp source files
│   │   ├── package.json              # "name": "@sonara/dsp"
│   │   └── scripts/compile-faust.sh  # Faust → C++ → WASM pipeline
│   │
│   └── shared/                       # Shared TypeScript code
│       ├── package.json              # "name": "@sonara/shared"
│       ├── src/
│       │   ├── schemas/              # JSON Schema for .sonara project files
│       │   ├── types/                # TypeScript interfaces (Project, Track, Clip)
│       │   └── utils/                # Helper functions (timecode formatting, etc.)
│       └── tsconfig.json
│
├── docker/                           # Dockerfiles for each app
├── .github/workflows/                # CI/CD pipelines
├── package.json                      # Root package.json (workspace config)
├── pnpm-workspace.yaml               # pnpm workspace definition
├── turbo.json                        # Turborepo pipeline config ⭐
├── tsconfig.json                     # Root TypeScript config (base)
└── README.md
```

---

## 3. Key Configuration Files

### A. Root `package.json` (Workspace Definition)

```json
{
  "name": "sonara-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "clean": "turbo run clean && rm -rf node_modules",
    "format": "prettier --write \"**/*.{ts,tsx,md}\""
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0"
  },
  "packageManager": "pnpm@9.0.0",
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### B. `pnpm-workspace.yaml` (Define Workspace Packages)

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### C. `turbo.json` (The Brain of the Operation) ⭐

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["tsconfig.json", ".env"],
  "globalEnv": ["NODE_ENV", "SONARA_ENV"],
  "tasks": {
    // --- Build Tasks ---
    "build": {
      "dependsOn": ["^build"],
      "outputs": [
        "apps/web/dist/**",
        "apps/server/bin/**",
        "packages/engine/pkg/**",
        "packages/dsp/wasm/**",
        "packages/shared/dist/**"
      ],
      "cache": true
    },

    // --- Development Tasks ---
    "dev": {
      "dependsOn": ["^build"],
      "persistent": true,
      "cache": false
    },

    // --- Testing ---
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true
    },

    // --- Linting ---
    "lint": {
      "cache": true
    },

    // --- Cleaning ---
    "clean": {
      "cache": false
    },

    // --- Rust/WASM Specific ---
    "build:wasm": {
      "dependsOn": ["^build"],
      "outputs": ["packages/engine/pkg/**", "packages/dsp/wasm/**"],
      "cache": true
    },

    // --- Go Backend Specific ---
    "build:go": {
      "dependsOn": ["^build"],
      "outputs": ["apps/server/bin/**"],
      "cache": true
    }
  }
}
```

#### 🔑 Key `turbo.json` Concepts:
- **`dependsOn`:** Defines task dependencies. `"^build"` means "build all dependencies in the dependency graph first."
- **`outputs`:** Tells Turbo what files are produced. This enables caching—if outputs haven't changed, skip the task.
- **`persistent`:** For long-running tasks like `dev` servers.
- **`cache`:** Whether to cache the task results in `node_modules/.cache/turbo`.

### D. Root `tsconfig.json` (Base TypeScript Config)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "composite": true,
    "baseUrl": ".",
    "paths": {
      "@sonara/shared/*": ["packages/shared/src/*"],
      "@sonara/engine": ["packages/engine/src"],
      "@sonara/dsp": ["packages/dsp/src"]
    }
  },
  "exclude": ["node_modules", "**/pkg", "**/wasm", "**/dist"]
}
```

---

## 4. Package-Specific Configurations

### A. `packages/engine/package.json` (Rust WASM Wrapper)

```json
{
  "name": "@sonara/engine",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./pkg/sonara_engine.js",
  "types": "./pkg/sonara_engine.d.ts",
  "scripts": {
    "build": "./scripts/build-wasm.sh",
    "clean": "rm -rf pkg",
    "dev": "npm run build -- --watch"
  },
  "devDependencies": {
    "@wasm-tool/wasm-pack-plugin": "^1.7.0",
    "wasm-pack": "^0.12.0"
  }
}
```

### B. `packages/dsp/package.json` (Faust WASM)

```json
{
  "name": "@sonara/dsp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "exports": {
    "./eq3": "./wasm/eq3.js",
    "./compressor": "./wasm/compressor.js",
    "./reverb": "./wasm/reverb.js"
  },
  "scripts": {
    "build": "./scripts/compile-faust.sh",
    "clean": "rm -rf wasm"
  }
}
```

### C. `apps/web/package.json` (React Frontend)

```json
{
  "name": "@sonara/web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,tsx",
    "clean": "rm -rf dist node_modules/.vite"
  },
  "dependencies": {
    "@sonara/shared": "workspace:*",
    "@sonara/engine": "workspace:*",
    "@sonara/dsp": "workspace:*",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "zustand": "^4.5.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "vite": "^5.0.0",
    "vite-plugin-wasm": "^3.3.0"
  }
}
```

---

## 5. How to Initialize Turborepo for SONARA

### Step 1: Create the Monorepo Scaffold

```bash
# Create root directory
mkdir sonara && cd sonara

# Initialize with pnpm (recommended for monorepos)
pnpm init

# Install Turborepo
pnpm add -D turbo

# Create workspace config
echo "packages:
  - 'apps/*'
  - 'packages/*'" > pnpm-workspace.yaml
```

### Step 2: Generate `turbo.json`

```bash
# Create turbo.json with the config from Section 3C above
# Or use the interactive generator:
npx turbo@latest init
```

### Step 3: Create the Packages

```bash
# Create shared TypeScript package
mkdir -p packages/shared/src/{schemas,types,utils}
cd packages/shared
pnpm init
# Edit package.json with config from Section 4
cd ../..

# Create engine (Rust WASM) package
mkdir -p packages/engine/{src,scripts,pkg}
cd packages/engine
pnpm init
# Edit package.json with Rust/WASM config
cd ../..

# Create dsp (Faust) package
mkdir -p packages/dsp/{effects,wasm,scripts}
cd packages/dsp
pnpm init
# Edit package.json with Faust config
cd ../..

# Create web (React) app
pnpm create vite apps/web --template react-ts
cd apps/web
pnpm add @sonara/shared @sonara/engine @sonara/dsp zustand
cd ../..

# Create server (Go) app
mkdir -p apps/server/{cmd,internal,migrations}
cd apps/server
go mod init sonara-server
cd ../..
```

### Step 4: Configure Path Aliases

In `apps/web/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/shared" },
    { "path": "../../packages/engine" },
    { "path": "../../packages/dsp" }
  ]
}
```

---

## 6. Essential Turborepo Commands for SONARA

```bash
# 🔥 Start all development servers in parallel
pnpm dev
# → Starts: React Vite server, Go backend, WASM file watchers

# 🏗️ Build everything (with caching)
pnpm build

# 🧪 Run tests across all packages
pnpm test

# 🧹 Clean all build artifacts
pnpm clean

# 🎯 Run a specific task in a specific package
pnpm --filter @sonara/web build
pnpm --filter @sonara/engine build:wasm

# 📊 Visualize the task dependency graph
npx turbo run build --graph=dependency-graph.png

# 🔄 Force rebuild ignoring cache
pnpm build -- --force
```

---

## 7. Advanced: Handling Non-JS Packages (Rust, Go, Faust)

Turborepo is JS/TS-native, but SONARA uses Rust, Go, and Faust. Here's how to integrate them:

### A. Rust/WASM (`packages/engine`)
- Use `wasm-pack` to compile Rust → WASM + JS bindings.
- Wrap the build command in a `package.json` script so Turbo can track it.
- Declare `pkg/` as an output in `turbo.json` for caching.

### B. Faust DSP (`packages/dsp`)
- Faust compiles to C++, then to WASM via `emscripten`.
- Create a shell script (`compile-faust.sh`) that loops through `.dsp` files.
- Treat the compiled `.js`/`.wasm` files in `wasm/` as build outputs.

### C. Go Backend (`apps/server`)
- Go is not managed by pnpm/Turbo, but co-locate it in the monorepo.
- Add a `build:go` task in `turbo.json` that runs `go build -o bin/server`.
- Use `dependsOn: ["^build"]` to ensure shared types are built first.

### D. Cross-Package Type Sharing
- Define TypeScript interfaces in `packages/shared/src/types/audio.ts`.
- Import them in both `apps/web` and `apps/server` (via code generation or manual sync).
- Use `json-schema-to-typescript` to generate TS types from your `.sonara` JSON schema.

---

## 8. CI/CD Integration (GitHub Actions Example)

`.github/workflows/ci.yml`:

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      
      - name: Install Rust & Faust
        run: |
          curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
          curl -sSL https://faust.grame.fr/install.php | bash
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Cache Turbo builds
        uses: actions/cache@v4
        with:
          path: node_modules/.cache/turbo
          key: turbo-${{ runner.os }}-${{ hashFiles('**/package.json') }}
      
      - name: Build all packages
        run: pnpm build
      
      - name: Run tests
        run: pnpm test
      
      - name: Build Docker images
        run: |
          docker build -f docker/web.Dockerfile -t sonara-web .
          docker build -f docker/server.Dockerfile -t sonara-server .
```

---

## 9. Common Pitfalls & Solutions

| Pitfall | Solution |
|---|---|
| **"Cannot find module @sonara/shared"** | Ensure `pnpm install` was run at root. Check `pnpm-workspace.yaml` includes the package. |
| **WASM files not loading in browser** | Vite needs `vite-plugin-wasm` and `vite-plugin-top-level-await`. Configure `vite.config.ts`. |
| **Turbo cache growing too large** | Add `--cache-dir=/tmp/turbo` or configure cache limits in `turbo.json`. |
| **Go backend can't import shared types** | Use `json-schema-to-typescript` to generate Go structs from the same JSON schema used for TS. |
| **Faust compilation fails on CI** | Pre-install the Faust CLI in your Dockerfile or GitHub Actions runner. |

---

## 10. Final Checklist: Is Your Turborepo Set Up Correctly?

✅ Root `package.json` has `turbo` as a devDependency  
✅ `pnpm-workspace.yaml` lists all `apps/*` and `packages/*`  
✅ `turbo.json` defines `build`, `dev`, `test` tasks with correct `outputs`  
✅ Each package has its own `package.json` with `workspace:*` dependencies  
✅ TypeScript `tsconfig.json` uses `composite: true` and `references` for type safety  
✅ Rust/Faust build scripts are wrapped in `package.json` so Turbo can cache them  
✅ `pnpm dev` starts all services with one command  
✅ CI pipeline uses Turbo's caching to speed up builds  

---

**Turbo Tip:** Run `npx turbo login` to enable remote caching. This lets your team share build caches across machines, dramatically speeding up CI and local development.

With this setup, SONARA's complex, multi-language architecture becomes manageable, fast, and scalable—exactly what a self-hosted, open-source DAW needs. 🎛️🚀

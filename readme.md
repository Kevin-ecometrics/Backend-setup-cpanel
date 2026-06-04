# Bun → cPanel Deployment Guide

Deploy any Express + Bun project to cPanel/Passenger (Node.js runtime).

---

## 1. Project Structure

```
my-app/
├── dist/                    # Build output (1 file)
│   └── index.js
├── src/
│   ├── index.ts             # Entry point
│   ├── app.ts               # Express app setup
│   ├── config/
│   │   └── index.ts         # Env vars loader
│   ├── db/
│   │   ├── connection.ts    # Pool / client
│   │   └── schema.sql       # DDL
│   ├── middleware/
│   │   ├── auth.ts          # JWT verification
│   │   ├── adminOnly.ts     # Role guard
│   │   ├── errorHandler.ts  # Global error handler
│   │   └── validate.ts      # Input validation
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── users.ts
│   │   ├── products.ts
│   │   └── index.ts         # Route aggregator
│   ├── controllers/
│   │   ├── authController.ts
│   │   ├── userController.ts
│   │   └── productController.ts
│   ├── services/
│   │   ├── authService.ts
│   │   ├── emailService.ts
│   │   └── paymentService.ts
│   ├── types/
│   │   └── index.ts         # Shared types / interfaces
│   └── utils/
│       ├── helpers.ts
│       └── logger.ts
├── package.json
├── tsconfig.json
└── .env.example
```

---

## 2. package.json

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "dev":   "bun run src/index.ts",
    "build": "bun build src/index.ts --format=cjs --outdir=dist",
    "start": "node dist/index.js"
  }
}
```

| Field | Why |
|---|---|
| `"main": "dist/index.js"` | Passenger detects the entry point automatically |
| `--format=cjs` | cPanel runs Node.js, not Bun → CommonJS required |
| `"start": "node dist/index.js"` | Passenger executes `npm start` |
| No `"type": "module"` | CJS doesn't need it; omitting it avoids ESM errors |

---

## 3. src/config/index.ts

```ts
export const config = {
  port: parseInt(process.env.PORT ?? '3000'),
  nodeEnv: process.env.NODE_ENV ?? 'development',

  db: {
    host: process.env.DB_HOST ?? 'localhost',
    port: parseInt(process.env.DB_PORT ?? '3306'),
    user: process.env.DB_USER ?? 'root',
    password: process.env.DB_PASSWORD ?? '',
    name: process.env.DB_NAME ?? 'myapp',
  },

  jwt: {
    secret: process.env.JWT_SECRET ?? '',
    refreshSecret: process.env.JWT_REFRESH_SECRET ?? '',
  },
}
```

---

## 4. src/db/connection.ts

```ts
import mysql from 'mysql2/promise'
import { config } from '../config'

const pool = mysql.createPool({
  host: config.db.host,
  user: config.db.user,
  password: config.db.password,
  database: config.db.name,
  waitForConnections: true,
  connectionLimit: 10,
})

export default pool
```

For MongoDB:
```ts
import { MongoClient } from 'mongodb'
import { config } from '../config'

const client = new MongoClient(config.db.url)
export const db = client.db(config.db.name)
```

---

## 5. src/middleware/auth.ts

```ts
import type { Request, Response, NextFunction } from 'express'
import jwt from 'jsonwebtoken'
import { config } from '../config'

export interface AuthUser {
  id: number
  email: string
  role: string
}

declare global {
  namespace Express {
    interface Request { user?: AuthUser }
  }
}

export function auth(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing token' })
  }

  try {
    const token = header.slice(7)
    req.user = jwt.verify(token, config.jwt.secret) as AuthUser
    next()
  } catch {
    return res.status(401).json({ error: 'Invalid token' })
  }
}
```

---

## 6. src/middleware/errorHandler.ts

```ts
import type { Request, Response, NextFunction } from 'express'

export function errorHandler(err: any, _req: Request, res: Response, _next: NextFunction) {
  console.error('[ERROR]', err)

  const status = err.status || err.statusCode || 500
  const message = err.message || 'Internal server error'

  res.status(status).json({ error: message })
}
```

---

## 7. src/routes/index.ts

```ts
import { Router } from 'express'
import authRoutes from './auth'
import userRoutes from './users'
import productRoutes from './products'
import { auth } from '../middleware/auth'

const router = Router()

router.use('/auth', authRoutes)
router.use('/users', auth, userRoutes)
router.use('/products', auth, productRoutes)

export default router
```

---

## 8. src/routes/auth.ts

```ts
import { Router } from 'express'
import { login, register } from '../controllers/authController'

const router = Router()

router.post('/login', login)
router.post('/register', register)

export default router
```

---

## 9. src/controllers/authController.ts

```ts
import type { Request, Response } from 'express'
import { authenticateUser } from '../services/authService'

export async function login(req: Request, res: Response) {
  try {
    const { email, password } = req.body
    const result = await authenticateUser(email, password)
    res.json(result)
  } catch (err: any) {
    const message = err.message
    if (message === 'Invalid credentials') {
      return res.status(401).json({ error: message })
    }
    res.status(500).json({ error: 'Internal server error' })
  }
}

export async function register(req: Request, res: Response) {
  // ...
}
```

---

## 10. src/services/authService.ts

```ts
import bcrypt from 'bcryptjs'
import jwt from 'jsonwebtoken'
import pool from '../db/connection'
import { config } from '../config'

export async function authenticateUser(email: string, password: string) {
  const [rows] = await pool.query('SELECT * FROM users WHERE email = ?', [email]) as any[]
  const user = rows[0]
  if (!user) throw new Error('Invalid credentials')

  const valid = await bcrypt.compare(password, user.password_hash)
  if (!valid) throw new Error('Invalid credentials')

  const token = jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    config.jwt.secret,
    { expiresIn: '24h' }
  )

  return { token, user: { id: user.id, email: user.email, name: user.name, role: user.role } }
}
```

---

## 11. src/types/index.ts

```ts
export interface User {
  id: number
  email: string
  name: string | null
  role: 'admin' | 'operator'
  created_at: string
}

export interface Product {
  id: number
  name: string
  price: number
  stock: number
  barcode: string | null
  created_at: string
}
```

---

## 12. src/utils/logger.ts

```ts
import fs from 'fs'
import path from 'path'

const logDir = process.env.LOG_DIR || 'logs'

try {
  fs.mkdirSync(logDir, { recursive: true })
} catch {
  // Directory creation failed — logs go to stdout only
}

export function info(message: string, ...args: any[]) {
  console.log(`[INFO] ${message}`, ...args)
}

export function error(message: string, ...args: any[]) {
  console.error(`[ERROR] ${message}`, ...args)
}

// Winston alternative (uncomment if winston is installed):
// import winston from 'winston'
// const transports: winston.transport[] = [new winston.transports.Console()]
// try {
//   fs.mkdirSync(logDir, { recursive: true })
//   transports.push(new winston.transports.File({
//     filename: path.join(logDir, 'error.log'),
//     level: 'error',
//   }))
// } catch { /* console only */ }
// export default winston.createLogger({ transports })
```

**Why the try-catch:** Passenger may have restricted filesystem permissions — the fallback ensures the app starts regardless.

---

## 13. src/index.ts (Entry Point)

```ts
import 'dotenv/config'
import { app } from './app'
import { config } from './config'
import logger from './utils/logger'
import pool from './db/connection'

const startErrors: string[] = []

// Diagnostic endpoint — always first
app.get('/api/startup-status', (_req, res) => {
  res.json({
    alive: true,
    node: process.version,
    env: config.nodeEnv,
    cwd: process.cwd(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    errors: startErrors,
  })
})

async function start() {
  try {
    // Verify DB connection (optional)
    await pool.query('SELECT 1')
    logger.info('Database connected')
  } catch (e: any) {
    logger.error('Database connection failed:', e.message)
  }

  app.listen(config.port, () => {
    logger.info(`App running on port ${config.port}`)
  })
}

start().catch(e => startErrors.push(e.message))
// NEVER use process.exit(1) — Passenger kills the process on non-zero exit
```

---

## 14. src/app.ts (Express Setup)

```ts
import express from 'express'
import cors from 'cors'
import routes from './routes'
import { errorHandler } from './middleware/errorHandler'
import path from 'path'
import fs from 'fs'

export const app = express()

app.use(cors())
app.use(express.json())
app.use(express.urlencoded({ extended: true }))

// API routes
app.use('/api', routes)

// Optional: serve static frontend on same domain
// const staticDir = process.env.STATIC_DIR || path.resolve('../frontend/dist')
// if (fs.existsSync(staticDir)) {
//   app.use(express.static(staticDir, { extensions: ['html'] }))
// }

// Optional: trailing slash removal for SPA routing
// app.use((req, res, next) => {
//   if (req.path.length > 1 && req.path.endsWith('/')) {
//     return res.redirect(301, req.path.slice(0, -1) + req.url.slice(req.path.length))
//   }
//   next()
// })

// Error handler (must be last)
app.use(errorHandler)
```

### Middleware Order

```
 1. CORS
 2. Body parser (JSON + URL-encoded)
 3. Diagnostic endpoint (/api/startup-status)
 4. API routes
 5. Trailing-slash redirect (301)   ← if serving frontend
 6. Static files                     ← if serving frontend
 7. 404 handler
 8. Global error handler
```

---

## 15. Golden Rules

| ❌ Don't | ✅ Do |
|---|---|
| `process.exit(1)` on startup fail | Collect errors in `startErrors[]` |
| `throw` at top level | `start().catch(e => startErrors.push(e))` |
| `"type": "module"` in package.json | Omit it (pure CJS) |
| Dynamic `import()` | Not supported well in CJS bundles |

---

## 16. .env.example

```env
# Server
PORT=3000
NODE_ENV=production

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_NAME=myapp

# JWT
JWT_SECRET=change-me
JWT_REFRESH_SECRET=change-me-too

# External APIs (example)
# API_KEY=
# API_SECRET=
# REDIRECT_URI=http://localhost:3000/api/callback

# Logs
LOG_DIR=logs
LOG_LEVEL=info
```

---

## 17. Deploy Checklist

### Local
- [ ] `bun run build` → produces `dist/index.js`
- [ ] Test locally: `node dist/index.js`
- [ ] Verify `/api/startup-status` responds

### cPanel
- [ ] Upload `dist/index.js` to the app directory
- [ ] Upload `.env` (never commit it to the repo)
- [ ] `npm install --production` (installs runtime dependencies)
- [ ] Configure Passenger:
  - Type: Node.js
  - Startup file: `dist/index.js`
  - (Auto-detected via `"main"` in package.json)
- [ ] Restart Passenger app
- [ ] Visit `https://yourdomain.com/api/startup-status` to verify

### Troubleshooting

| Error | Likely cause |
|---|---|
| Passenger error ID (e.g. `714fb24b`) | `process.exit(1)` — process dies on start |
| `Cannot find module` | ESM vs CJS mismatch → check `--format=cjs` and no `"type": "module"` |
| `MODULE_NOT_FOUND` in production | `npm install --production` not run or missing deps |
| Blank page / 404 | Static frontend path misconfigured (`STATIC_DIR`) |
| Logger crashes startup | Missing `fs.mkdirSync` or try-catch for File transport |
| `var` in built output | Normal — Bun/esbuild converts `const`/`let` to `var` |

---

## 18. Quick Reference

```bash
# Development
bun run src/index.ts            # with --watch for hot reload

# Build for cPanel
bun run build                   # → dist/index.js (1 CJS file)

# Test build locally
node dist/index.js

# On cPanel (SSH)
cd ~/applications/my-app
npm install --production
node dist/index.js              # Manual test before enabling Passenger
```

---

## Summary

```
Bun (dev)           bun build        dist/index.js (CJS)     upload      cPanel + Passenger
const/let,          ──────────▶      var, bundled,            ──────────▶  Node.js runtime
many files                          single file                             no deps needed
```

Every time you change source code:
1. `bun run build`
2. Upload `dist/index.js`
3. Passenger detects changes automatically (or manual restart)

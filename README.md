# AI Animator (Manim Studio)

Type a topic, review a storyboard, and get back a rendered explainer video. An LLM writes the storyboard, you approve it, a second LLM pass turns it into [Manim](https://www.manim.community/) Python code, and a worker renders the video. If the render fails, the error goes back to the LLM to fix the code and the render runs again.

## Contents

- [How it works](#how-it-works)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Running it on your own machine](#running-it-on-your-own-machine)
- [Running with no internet (Ollama)](#running-with-no-internet-ollama)
- [Environment variables](#environment-variables)
- [API reference](#api-reference)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)

## How it works

1. You sign up or sign in. The API sets a JWT in an HTTP-only cookie and also returns it in the response body.
2. You enter a topic. `POST /generate/chat` creates a `Thread` row in Postgres, starts a LangGraph run with a fresh thread id, and returns the storyboard.
3. The graph pauses at `askForApproval` using LangGraph's `interrupt()`. Its state is saved in MongoDB, so the process can handle other requests while it waits.
4. If you reject the storyboard, `POST /generate/approve_storyboard` resumes the graph with `approved: false`. It loops back to `storyboardWriter` and returns a new storyboard.
5. If you approve, the graph resumes, `coder` writes Manim code, and `offload` puts a render job on a BullMQ queue backed by Redis. The graph then pauses again at `waitForRenderResult`.
6. The BullMQ worker picks up the job, writes `code.py`, runs `manim -ql`, and moves the output to `backend/temp/<userId>/video.mp4`.
7. The worker resumes the paused graph with `Command({ resume })`. On success the graph ends. On failure it goes to `solveError`, where the LLM rewrites the code, and then back to `offload`. After three failed retries it gives up.
8. While all this happens, the approve request stays open. The API polls for `video.mp4` and sends the file back in the same response.

## Architecture

```mermaid
flowchart TB
    subgraph FE["Frontend: React + Vite"]
        Auth["Auth.tsx"]
        Studio["Studio.tsx"]
        Api["api.ts (fetch + JWT)"]
    end

    subgraph BE["Backend: Express, port 3000"]
        Users["/users: signup, signin"]
        Guard["authenticate_user (JWT)"]
        Gen["/generate: chat, approve_storyboard"]
        Poll["await-video: poll + sendFile"]
    end

    subgraph LG["LangGraph state machine"]
        SW["storyboardWriter (LLM)"]
        AA["askForApproval (interrupt)"]
        CO["coder (LLM)"]
        OF["offload"]
        WR["waitForRenderResult (interrupt)"]
        SE["solveError (LLM)"]
        END_(["END"])
        SW --> AA --> CO --> OF --> WR
        WR -- "error, retries left" --> SE --> OF
        WR -- "success or give up" --> END_
    end

    subgraph RW["Render worker: same Node process"]
        Q["BullMQ worker: processManim"]
        M["Manim CLI in .venv"]
        V[("temp/userId/video.mp4")]
        Q --> M --> V
    end

    PG[("Postgres: users, threads")]
    MG[("MongoDB: graph checkpoints")]
    RD[("Redis: render-queue")]
    LLM["Gemini via LangChain"]

    Studio --> Api
    Auth --> Api
    Api <--> Users
    Api <--> Guard --> Gen
    Users --> PG
    Gen --> PG
    Gen <--> LG
    LG --> LLM
    LG <--> MG
    OF -- "enqueue" --> RD --> Q
    Q -- "Command(resume)" --> WR
    V --> Poll --> Api
```

### The graph

| Node | Type | What it does |
|---|---|---|
| `storyboardWriter` | LLM | Turns the topic into a scene-by-scene storyboard |
| `askForApproval` | interrupt | Pauses until the client approves or rejects |
| `coder` | LLM | Turns the approved storyboard into a Manim scene |
| `offload` | plain | Adds the render job to the queue and returns immediately |
| `waitForRenderResult` | interrupt | Pauses until the worker resumes it with a result |
| `solveError` | LLM | Gets the error and the failing code, returns fixed code |

Graph state (`backend/src/graph/state.ts`) holds `userId`, `topic`, `storyBoard`, `approved`, `code`, `error`, `hasError`, `videoPath`, `retryCount` and `status`.

### Data stores

| Store | Used for | Written by |
|---|---|---|
| Postgres | `User`, `Thread`, `Message` tables | Prisma, from the auth routes and `/generate/chat` |
| MongoDB | LangGraph checkpoints, so paused runs survive | `MongoDBSaver` |
| Redis | The `render-queue` BullMQ queue | `offload`, read by the worker |
| Local disk | `backend/temp/<userId>/code.py` and `video.mp4` | The render processor |

## Tech stack

- Frontend: React 18, Vite 5, TypeScript, React Router, GSAP
- Backend: Node.js, Express 4, TypeScript (ESM), Prisma 6
- Orchestration: LangGraph, LangChain (`@langchain/google-genai`, `@langchain/ollama`)
- Queue: BullMQ and ioredis
- Rendering: Manim Community Edition in a Python virtualenv

## Project structure

```
ai-animator/
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   └── src/
│       ├── index.ts                  Express app, mounts /users and /generate
│       ├── config/
│       │   ├── db.ts                 Prisma client singleton
│       │   ├── redis.config.ts       ioredis connection (localhost:6379)
│       │   ├── checkpointer.ts       MongoDB checkpointer for LangGraph
│       │   └── model.ts              LLM client (Gemini by default)
│       ├── middlewares/
│       │   └── authentication.middleware.ts
│       ├── routes/
│       │   ├── authentication.routes.ts
│       │   ├── chat.routes.ts        /generate/chat, /generate/approve_storyboard
│       │   └── await-video.ts        polls for video.mp4
│       ├── graph/
│       │   ├── graph.ts              nodes, edges, retry logic
│       │   ├── nodes.ts
│       │   ├── state.ts
│       │   └── prompts.ts            system prompts for the three LLM nodes
│       ├── queues/render.queue.ts
│       ├── workers/render.worker.ts  BullMQ worker, started on import
│       └── processors/
│           ├── render.processor.ts   saves code, runs Manim, resumes graph
│           └── render_code.processor.ts   older copy, unused
└── frontend/
    └── src/
        ├── App.tsx                   routes and top bar
        ├── Auth.tsx                  sign in and sign up
        ├── Studio.tsx                topic, storyboard, generating, video
        └── api.ts                    fetch wrapper and token handling
```

## Running it on your own machine

Everything except the LLM runs locally. By default the LLM is Gemini, which needs an internet connection and an API key. To remove that dependency, see the next section.

### 1. Install the prerequisites

| Tool | Version | Used for |
|---|---|---|
| Node.js | 20 or newer | backend and frontend |
| PostgreSQL | 14 or newer | users and threads |
| MongoDB | 6 or newer | LangGraph checkpoints |
| Redis | 6 or newer | render queue |
| Python | 3.10 or newer | Manim |
| FFmpeg | any recent | video encoding |

Manim also needs Cairo and Pango. On Ubuntu or Debian:

```bash
sudo apt install -y ffmpeg libcairo2-dev libpango1.0-dev pkg-config \
  python3-dev python3-venv build-essential
```

On macOS:

```bash
brew install ffmpeg cairo pango pkg-config
```

LaTeX is not needed. The prompts forbid `Tex` and `MathTex`, so generated scenes use plain `Text`.

### 2. Start Postgres, MongoDB and Redis

With system services:

```bash
sudo systemctl start postgresql mongod redis
```

Or with Docker:

```bash
docker run -d --name animator-pg    -p 5432:5432 -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=animator postgres:16
docker run -d --name animator-mongo -p 27017:27017 mongo:7
docker run -d --name animator-redis -p 6379:6379 redis:7
```

Redis has to be on `localhost:6379`. That address is hardcoded in `backend/src/config/redis.config.ts`.

### 3. Configure the backend

Create `backend/.env`:

```env
PORT=3000
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/animator"
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DB="manim_generator"
JWT_SECRET="replace-with-a-long-random-string"
GOOGLE_API_KEY="your-gemini-api-key"
```

`GOOGLE_API_KEY` is not needed if you switch to Ollama.

### 4. Install dependencies and create the tables

```bash
cd backend
npm install
npx prisma generate
npx prisma migrate deploy
```

If the migrations fail on a fresh database, `npm run db:push` creates the tables straight from `schema.prisma`.

### 5. Build the Manim environment ahead of time

The worker creates `backend/.venv` and installs Manim the first time a job runs, which needs internet and makes the first render very slow. Do it once now instead:

```bash
cd backend
python3 -m venv .venv
.venv/bin/pip install manim numpy
.venv/bin/manim --version
```

Once `.venv` exists, the worker skips the install step and never touches the network for rendering.

### 6. Start the backend

```bash
cd backend
npm run dev
```

The server listens on `http://localhost:3000`. The render worker starts in the same process, so there is nothing else to launch.

Don't use `npm start`. Its path points at `/home/sam/projects/ai-animator/backend/dist/index.js`. To run a built copy, use `npm run build` and then `node dist/index.js` from the `backend` folder.

### 7. Start the frontend

```bash
cd frontend
cp .env.example .env
npm install
npm run dev
```

`.env` should contain:

```env
VITE_API_BASE_URL=http://localhost:3000
```

Open the address Vite prints, normally `http://localhost:5173`. Sign up, enter a topic, approve the storyboard, and wait for the video.

## Running with no internet (Ollama)

`@langchain/ollama` is already a dependency, so only the model config changes.

1. Install [Ollama](https://ollama.com) while you still have a connection and pull a model that writes decent code:

   ```bash
   ollama pull qwen2.5-coder:7b
   ```

2. Replace `backend/src/config/model.ts`:

   ```ts
   import { ChatOllama } from "@langchain/ollama";

   export const localLlm = new ChatOllama({
     model: "qwen2.5-coder:7b",
     baseUrl: "http://localhost:11434",
   });
   ```

3. Make sure `backend/.venv` is already built (step 5 above) and run `npm install` in both folders before you disconnect.

4. Start Postgres, MongoDB, Redis, Ollama, the backend and the frontend. Nothing in the pipeline calls an outside host after that.

Small local models make more Manim mistakes than Gemini does, so you will see more passes through the `solveError` retry loop. A 14B model or larger gives noticeably better first-try code if your hardware can run it.

## Environment variables

### Backend (`backend/.env`)

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `DATABASE_URL` | yes | none | Postgres connection string for Prisma |
| `MONGODB_URI` | yes | none | MongoDB connection string. The server throws on startup without it |
| `MONGODB_DB` | no | `manim_generator` | Database name for checkpoints |
| `JWT_SECRET` | yes | none | Signs and verifies auth tokens |
| `GOOGLE_API_KEY` | with Gemini | none | Gemini API key |
| `PORT` | no | `3000` | API port |
| `RENDER_TIMEOUT_MS` | no | `300000` | How long a request waits for a render before returning an error |
| `BACKEND_URL` | no | `http://localhost:3000` | Read by the render processor, currently unused |

### Frontend (`frontend/.env`)

| Variable | Required | Purpose |
|---|---|---|
| `VITE_API_BASE_URL` | yes | Base URL of the backend |

## API reference

All `/generate` routes need a valid token, sent either as the `token` cookie or as `Authorization: Bearer <token>`.

### `POST /users/signup`

Body: `{ "email", "password", "username?", "name?" }`. Returns `200` with `{ message, token }` and sets the cookie. Returns `400` if a field is missing or the email is already registered.

### `POST /users/signin`

Body: `{ "email", "password" }`. Returns `200` with `{ message, token }` and sets the cookie. Returns `401` for an unknown email or a wrong password.

### `POST /generate/chat`

Body: `{ "topic": "How binary search works" }`. `message` is accepted as an alias for `topic`.

Creates a thread and runs the graph up to the approval pause. Returns:

```json
{ "storyBoard": "...", "threadId": "uuid" }
```

Keep the `threadId` and send it back on the next call.

### `POST /generate/approve_storyboard`

Body: `{ "approved": true | false, "threadId": "uuid" }`.

- `approved: false` returns `{ "storyBoard": "..." }` with a new storyboard for the same thread.
- `approved: true` holds the request open until the render finishes, then responds with the `video/mp4` file. If the render times out or fails after all retries, it returns `500` with `{ "message": "..." }`.

If `threadId` is omitted, the server falls back to the user's most recent thread.

## Troubleshooting

**`MONGODB_URI is not defined` on startup.** `backend/.env` is missing, or you started the server from a folder other than `backend/`.

**The first render takes minutes or fails.** The virtualenv doesn't exist yet and the worker is installing Manim. Build it manually as described in step 5.

**`Cannot find module` or `ERR_MODULE_NOT_FOUND`.** The backend is ESM (`"type": "module"`) and imports use `.js` extensions even for `.ts` files. Don't remove the extensions.

**Render fails with `FileNotFoundError: 'latex'`.** The model ignored the no-LaTeX rule and used `Tex`, `MathTex` or `DecimalNumber`. The retry loop usually fixes it. If it keeps happening, switch to a stronger model.

**Request returns "The render took too long".** Raise `RENDER_TIMEOUT_MS`. Low-quality renders (`-ql`) of long scenes can still take several minutes on a slow CPU.

**Gemini returns a model-not-found error.** `backend/src/config/model.ts` sets the model name. Change it to one your key can use.

**Frontend gets 401 right after signing in.** The cookie is set with `secure: false` and `sameSite: lax`. Run the frontend and backend on `localhost`, not on two different hostnames or over HTTPS to one and HTTP to the other. The client also sends the token as a Bearer header, which covers most of these cases.

**Redis connection errors.** Check that Redis answers on `localhost:6379` with `redis-cli ping`.

## Known limitations

- One video per user. The output path is `temp/<userId>/video.mp4`, so two simultaneous renders by the same user overwrite each other.
- The render worker runs inside the API process. A long render doesn't block requests, since Manim is a child process, but restarting the API restarts the worker.
- Video delivery depends on one long HTTP request. If the connection drops, the render still finishes, but the client has no endpoint to fetch the result afterwards.
- Redis host and port are hardcoded to `localhost:6379`.
- `render_code.processor.ts` is an older copy of the processor with a hardcoded thread id. Nothing imports it.
- The `Message` and `file` Prisma models exist but no route uses them yet.

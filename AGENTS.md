### ACE-Step Studio — Agent Guidelines 


#### 1. Project Overview & Core Principle

ACE-Step Studio is a **portable, self-contained** AI music generation app. It bundles Python (with CUDA), Node.js, and the ACE-Step 1.5 pipeline into a single launcher (`run.bat`). No installation required.

**The Single Most Important Rule:**
> **Never spawn a new Python/Gradio process to change models.** Use the `/v1/init` API for in-process model switching. Process restarts are for crash recovery only.

#### 2. Core Architecture (Simplified)

One `run.bat` → One Terminal Window → **Three Processes managed by Express**:

1.  **Express (Port 3001)**: The **supervisor**. Serves the static web UI (from `app/dist/`) and hosts all backend API routes.
2.  **Python/Gradio (Port 8001)**: The **ML worker**. Handles all model inference.
3.  **Vite**: **Development only**. Not present in production.

**Why This Works:** Express controls everything. It starts, monitors, and restarts the Python process if it crashes.

#### 3. Model Switching: The Critical Path (Do This)

**DO NOT** restart the Gradio process to switch models. Instead, use the dedicated API.

- **Express Route:** `/api/generate/switch-model`
- **Action:** Express calls `POST http://localhost:8001/v1/init` on the *existing* Python process.
- **Python's Job:** Unload the current LM/DiT → Clear GPU cache (`gc.collect()`, `empty_cache()`) → Load the new models.
- **Frontend:** Polls `/api/generate/model-status` to show "unloading → loading → ready".

#### 4. Gradio Communication (Express ↔ Python)

- **Tool:** Use `@gradio/client` library in Express.
- **Method:** Call `client.predict('/generation_wrapper', { ...namedParams })`.
- **Crucial:** Pass parameters as **named arguments** (a JavaScript object), NOT as an array.
- **Parameter Matching:** The object keys **must** exactly match the parameter names in Python's `generation_wrapper()` function signature.

#### 5. Adding a New Field to the `songs` Table (The #1 Bug Source)

This is a **multi-file, multi-query** change. You **must** update **all** of the following:

- **`migrate.ts`**: Add the `ALTER TABLE` SQL statement.
- **Database Queries (6+ locations)**: All `SELECT *` queries in `songs.ts` and `index.ts`.
- **Backend Model Logic (`acestep.ts`, `generate.ts`)**:
    - Map the field from `job.result`.
    - Include it in **both** `INSERT` statements (success & fallback). Use `activeLoadedModel`, NOT `params.*`.
- **Frontend API Client (`api.ts`)**:
    - Add to `Song` interface.
    - Add to **all** data mappers: `transformSongs`, `getSong`, `getFullSong`, `updateSong`.
- **Frontend State (`App.tsx`)**:
    - Add to **all three** song mapping functions: initial `mapSong`, `refreshSongsList` mapper, and `loadSongs` merge logic.
    - **Use the fail-safe pattern:** `s.snake_case || s.camelCase` for every field read from the API.
- **Frontend Types (`types.ts`)**: Add to `Song` interface.

#### 6. Internationalization (i18n)

- **Supported:** `en`, `ru`, `zh`, `ja`, `ko` (files in `app/i18n/`).
- **Rule:** Every user-visible string **must** use `t('key')`.
- **Pattern:** Every component using `t()` **must** have:
    ```tsx
    import { useI18n } from '../context/I18nContext';
    // Inside component:
    const { t } = useI18n();
    ```
- **Exceptions:** Universal terms (e.g., Facebook, GPU, VRAM) stay hardcoded.

#### 7. Frontend Production Build (Critical Reminder)

1.  After **any** change to frontend code (`app/`), you **must** rebuild:
    ```bash
    cd app && npx vite build
    ```
2.  This outputs to `app/dist/`, which Express serves in production (`run.bat`).
3.  **If you don't rebuild, your changes won't appear in production.**

#### 8. Video Studio Export Pipeline

The video generator (`VideoGeneratorModal.tsx`) follows this exact flow:

1.  **Browser (Preview & Frame Capture)**: Renders frames using the same logic as the preview.
2.  **Chunked Upload**: Sends frames to the server in batches of 50 (as base64 JPEGs).
3.  **Server-Endpoint Sequence**:
    1.  `POST /api/render-video/start` (init job)
    2.  `POST /api/render-video/frames` (send frame chunks)
    3.  `POST /api/render-video/finish` (trigger FFmpeg encoding)
4.  **Server (FFmpeg)**: Runs `ffmpeg.exe` locally (with NVENC if available) to encode the final MP4.
5.  **Return**: Server sends the final MP4 binary to the browser.

#### 9. Gradio Server: Fixing Your Tech Debt

Your current setup uses a standalone Python process. You can now upgrade to `gradio.Server` for better features.

**The Improvement:** Replace your custom Python/Gradio entry point with `gradio.Server`.

**How it simplifies your architecture:**

| Current (Your Setup) | Improved (with `gradio.Server`) |
| :--- | :--- |
| Python process is a "black box" Express manages via stdout. | Python process becomes a proper, extensible FastAPI app. |
| Pipeline readiness detected by parsing stdout. | Readiness is inherent; Express can use a standard health check on a FastAPI route. |
| Model switching via custom `/v1/init` route. | Same, but now built on a standard framework. |
| Express must spawn and manage the Python process. | **Better:** Express can still spawn it, OR `gradio.Server` can be the main app that serves both API & UI, reducing complexity. |
| No built-in queue for concurrent generations. | **`gradio.Server` provides automatic queuing and concurrency management for your `/generation_wrapper` endpoint.** |
| Not directly compatible with Hugging Face Spaces. | **Fully compatible with HF Spaces and ZeroGPU.** |

**Actionable Next Step for Tech Debt:**
Refactor `ACE-Step-1.5/acestep/acestep_v15_pipeline.py` to use `gradio.Server`. Your `generation_wrapper` becomes an `@app.api()` endpoint, gaining queuing, better client compatibility, and easier future hosting on Spaces.

#### 10. Key Rules (The "Don't Forget" List)

1.  **Never spawn new Gradio processes.**
2.  **Rebuild `app/dist/` after every frontend change.**
3.  **Update all 6+ song mappers when adding a DB field.**
4.  **Include `useI18n()` and `import` in every component with `t()`.**
5.  **Test in a real browser and check the console.**
6.  **Default DiT Model:** `marcorez8/acestep-v15-xl-turbo-bf16`
7.  **Default LM:** `acestep-5Hz-lm-0.6B` (backend: `pt`)

#### 11. Environment Variables (Most Important)

| Variable | Default | Description |
| :--- | :--- | :--- |
| `MANAGE_PIPELINE` | `false` | **Must be `true`** for Express to spawn Python. |
| `INIT_LLM` | `true` | Set `false` to run without LM (`run-no-lm.bat`). |
| `LM_MODEL` | `acestep-5Hz-lm-0.6B` | Which LM to load. |
| `LM_BACKEND` | `pt` | `pt` or `vllm`. |
| `PYTORCH_CUDA_ALLOC_CONF` | `expandable_segments:True` | **Always set this** to reduce VRAM fragmentation. |

#### 12. Development Commands

```bash
# Development (3 terminals, Vite HMR on port 3000)
run-dev.bat

# Production (1 terminal, static files on port 3001)
run.bat

# Manual frontend rebuild
cd app && npx vite build
```

#### 13. Known Tech Debt (Prioritized)

1.  **[CRITICAL] Model Switching Refactor**: Implement `gradio.Server` to get built-in queuing and stability.
2.  **[HIGH] App.tsx Mappers**: Refactor into a single, reusable `mapApiSongToAppSong` utility function.
3.  **[MEDIUM] CDN Dependencies**: Bundle Tailwind and React locally to remove runtime internet dependency.
4.  **[LOW] Video Export**: Move frame rendering to an `OffscreenCanvas` Web Worker to avoid UI blocking.

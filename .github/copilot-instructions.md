# Copilot instructions

## Snapshot
- React 19 + Vite 7 SPA under `src/`; Cloudflare Worker entry in `worker/index.ts` with SPA fallback configured in `wrangler.jsonc`.
- UI already migrated: `App.tsx` wires providers + router, `pages/Index.tsx` stitches every homepage section, `components/ui/*` hosts the shadcn kit.
- Tailwind + glassmorphism tokens live in `tailwind.config.ts` and `src/index.css`; media (hero, gallery, track) is in `src/assets/`.

## Frontend architecture
- `src/main.tsx` mounts `<App />` via `createRoot`. `App.tsx` owns the shared providers: `QueryClientProvider` → `TooltipProvider` → `Toaster` + `Sonner` → `BrowserRouter`. Add new routes inside the `<Routes>` block before the `*` catch-all (`pages/NotFound.tsx`).
- `pages/Index.tsx` renders section components in order: `HeroSection`, `AboutSection`, `GallerySection`, `StatsSection`, `Footer`, `AudioPlayer`. Anchor IDs (`about`, `gallery`) are used by CTA buttons for smooth scroll.
- State outside React Query uses local hooks; there’s no global store. Reach for `@tanstack/react-query` if you introduce async/server data so the existing provider keeps working.

## UI system & styling
- `components/ui/*` are shadcn-generated primitives that assume the `cn` helper in `src/lib/utils.ts` and Tailwind tokens defined in `src/index.css`. Keep imports as `@/components/ui/button`, etc.—the `@` alias is set in `vite.config.ts` + `tsconfig*.json`.
- `src/index.css` defines HSL design tokens plus `.glass-card`, `.glass-effect`, `.text-gradient`, and animation helpers used across sections. Tailwind scans `./src/**/*.{ts,tsx}` (see `tailwind.config.ts`); keep classes inside those paths to avoid missing styles.
- Animations frequently rely on inline `<style>` blocks (Hero particles, About/Stats/Gallery scroll transitions). Preserve or co-locate those snippets when refactoring; removing them silently breaks reveal effects.

## Behavioral patterns & hooks
- `hooks/use-mobile.tsx` is the single breakpoint watcher (max-width 768px). Use it for responsive toggles instead of duplicating `matchMedia` logic.
- `hooks/use-toast.ts` enforces a single toast (`TOAST_LIMIT = 1`) with a very long auto-dismiss (1,000,000 ms). Always render both `<Toaster />` and `<Sonner />` inside `TooltipProvider` to keep focus management correct.
- Scroll/visibility effects lean on `IntersectionObserver` (see `AboutSection`, `GallerySection`, `StatsSection`) and mutate DOM classes. Gate those effects inside `useEffect` so they never run during SSR/Workers rendering.
- `StatsSection` uses `echarts-for-react` with `opts={{ renderer: 'svg' }}` to avoid canvas hydration issues; copy that pattern for any new charts.
- `AudioPlayer.tsx` autoplays after a timeout, manipulates DOM classes, and controls `<audio>` via refs. Keep all logic behind `useEffect` guards and remember browsers may block autoplay until user interaction.

## Assets & media
- `src/assets/` holds `hero-mykaela.jpg`, six gallery images, and `track.mp3`. Components import them via the `@` alias; prefer static imports so Vite can optimize and fingerprint.
- The audio player currently streams a remote sample (`soundhelix.com`). Replace the `<source>` URL with `/src/assets/track.mp3` (or a Worker-served URL) when the final track is ready, and update any R2/Image delivery steps accordingly.

## Worker & API surface
- `worker/index.ts` currently returns JSON for `/api/*` and `404`s everything else; SPA assets are served by Cloudflare’s static site integration thanks to `assets.not_found_handling = "single-page-application"`.
- To add endpoints, branch on `url.pathname` inside `fetch` and `return Response.json(...)`. Keep payloads tiny—Workers default to ~10 ms CPU budgets. If you introduce bindings (R2, KV, D1, AI), add them in `wrangler.jsonc` and regenerate types with `npm run cf-typegen`.

## Tooling & workflows
- Scripts: `npm run dev` (Vite dev server on `::`:8080), `npm run lint` (eslint flat config), `npm run build` (`tsc -b` + Vite), `npm run preview`, `npm run deploy` (build + `wrangler deploy`), `npm run cf-typegen` (Worker env types).
- Always run `lint` + `build` locally before `deploy`; Wrangler re-runs the build but catching TS/Tailwind issues earlier saves time.
- `vite.config.ts` also loads the `@cloudflare/vite-plugin` for Worker-friendly builds and conditionally enables `lovable-tagger` during development. Keep that plugin last in the list if you add more tooling.

## Gotchas
- DOM-manipulating sections (Hero, About, Gallery, Stats, AudioPlayer) assume `window` is available. Guard any new effects with `typeof window !== "undefined"` if there’s a chance they’ll run server-side.
- Keep `TooltipProvider` > `Toaster` > `Sonner` order intact; swapping them breaks focus handling and toast dismissal.
- The gallery lightbox reuses the shadcn `Dialog`; when adding new media types, ensure the dialog stays portal-based to avoid z-index clipping with the floating audio player.
- Tailwind classes such as `glass-card` rely on `@layer utilities`; deleting those declarations in `index.css` will break the entire visual system.

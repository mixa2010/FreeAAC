# AGENTS.md

Offline-first AAC app for Android/iOS/web. Expo SDK 55 + Expo Router, React Native 0.83 with react-native-web, React 19, TypeScript strict, Zustand. AGPL-3.0.

## Commands

- `npm install` (CI uses `npm ci`); dev: `npm run web` / `npm run android` / `npm run ios`.
- `npm run lint` — the only automated check, also run by web CI. No test suite and no test/typecheck script; do not invent `npm test`.
- `npx tsc --noEmit` passes but is not run by CI; `npm run lint` is the only automated check.
- `npm run deploy:web` — static export, copies `dist/index.html` to `404.html`, regenerates the workbox service worker.
- `npm run reset-project` is stale (missing `scripts/reset-project.js`); don't run it.
- Native releases via EAS (see `eas.json`, README): `eas build -p android --profile preview --local`, `eas build --platform all --auto-submit`.

## Architecture

- Expo Router routes in `app/`: `/` board list, `/[boardId]` redirects to the manifest root, `/[boardId]/[pageId]` is the board UI; plus `create`, `settings`, `templates`, `privacy`.
- Two storage layers: board metadata (ids, names, page list, edit mode, message window, logs) is in Zustand (`stores/boards.ts`) persisted to MMKV; page content is OBF files on disk managed by `utils/file.ts` and loaded per page. Don't put page grids in the store.
- Platform-split modules: `utils/io.ts` (web OPFS) / `utils/io.native.ts` (expo-file-system), and `utils/tts.ts` (web no-op) / `utils/tts.native.ts` (Kokoro via react-native-executorch). Metro prefers `.native.ts` on native; keep both halves in sync.
- `@willwade/aac-processors` handles OBF/OBZ/Gridset import/export, the page model (`AACPage`/`AACTree`), and OBL analytics. Import from its subpaths: `/browser`, `/analytics`, `/metrics`.
- Speech: device voice (`expo-speech`) or on-device Kokoro; playback and audio session live in `components/AudioController.tsx`.
- UI: use `useTheme()` and the `GAP`/`PADDING`/`FONT_SIZE`/`RADIUS`/`ICON_SIZE`/`MAX_WIDTH` constants from `utils/theme.ts`; app screens use `Button`/`Text` from `components/Styled.tsx`.
- Import alias `@/*` maps to the repo root.
- `metro.config.cjs` enables package exports and forces zustand to its CJS entry (Expo workaround); leave intact.

## Gotchas

- Generated/gitignored, do not edit or commit: `android/`, `ios/`, `.expo/` (including typed-route types in `.expo/types/router.d.ts`), `dist/`, `expo-env.d.ts`. Root `.apk`/`.zip` files are local artifacts.
- `app.json` enables `reactCompiler` and `typedRoutes`; route strings are type-checked against generated types while the dev server runs.
- Web-only APIs (OPFS, `window`) belong only in `utils/io.ts`; shared code must go through the `.native.ts` twins.
- Formatting: Prettier `semi: false` + `experimentalTernaries`; ESLint ignores `dist/`, `android/`, `ios/`, `*.cjs`. VSCode runs fixAll/organizeImports on save.
- CI: push to `main`/`deploy-web` lints and deploys web to GitHub Pages (`deploy-web.yml`); every PR lints, builds the web export, and publishes a Cloudflare Pages preview at `https://pr-<number>.freeaac.pages.dev` with a sticky comment, deleted on close (`preview-web.yml`, needs `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` secrets). Android build/submit is manual `workflow_dispatch` in `deploy-android.yml`.

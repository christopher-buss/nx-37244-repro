# nx#37244 reproduction

Reproduction for [nrwl/nx#37244](https://github.com/nrwl/nx/issues/37244):
the project graph silently drops all TypeScript import edges when `typescript`
is not resolvable from nx's install directory, and caches that graph until
`nx reset`.

## Setup

- `@repro/lib-b` imports `@repro/lib-a` in source only (no `package.json`
  dependency).
- `libs/lib-b/tsconfig.json` already references `lib-a`, as `nx sync` writes it.
- `pnpm-workspace.yaml` sets `enableGlobalVirtualStore: true`, so nx installs
  into the pnpm global store. From there, the workspace's `typescript` does not
  resolve through the `node_modules/.bin/nx` shim. `pnpm nx` adds
  `<root>/node_modules` to `NODE_PATH`, so `typescript` resolves there.

## Steps

Requires pnpm 12.

```bash
pnpm install
export NX_DAEMON=false
```

1. Run nx through the bare shim. nx prints no warning, and `lib-b` has no
   dependencies:

   ```bash
   ./node_modules/.bin/nx graph --file=graph.json
   node -p "JSON.stringify(require('./graph.json').graph.dependencies['@repro/lib-b'])"
   # []
   ```

2. Run nx through pnpm. `typescript` now resolves, but nx reuses the cache from
   step 1 and reports a false stale reference:

   ```bash
   pnpm nx sync:check
   # libs/lib-b/tsconfig.json:
   #   - Stale references: libs/lib-a/tsconfig.json
   ```

3. Reset the cache. The edge comes back:

   ```bash
   pnpm nx reset
   pnpm nx sync:check
   # [@nx/js:typescript-sync]: All files are up to date.
   ```

## Expected

- Step 1 warns that nx skips source-file dependency analysis because it cannot
  resolve `typescript`.
- Step 2 does not reuse a file map built without the import scan.

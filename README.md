# `.ts` route files get JSX scaffolding, which stalls the generator

The route generator picks its scaffold template from `config.target` and
`_fsRouteType`, never from the file extension. So an empty `.ts` route file gets
the React component template, complete with JSX, and the generator then fails to
parse its own output as TypeScript.

## Reproduce

```bash
pnpm install
pnpm generate-routes
```

```
Error: Error transforming route file src/routes/api/foo.ts: SyntaxError: Missing semicolon. (8:19)
```

Line 8 column 19 is the `<` of the `<div>` in the filled-in template.

- `foo.ts` is empty.
- `bar.tsx` got the normal React component boilerplate.
- `src/routeTree.gen.ts` was never written, so it knows about neither file.

Re-running doesn't recover. `foo.ts` is still empty, so the generator scaffolds
it again and fails again. Route generation stays dead until you delete that file
or fill it in by hand.

`pnpm dev` fails the same way, easy to miss that the generated router file stops
generating.

## Cause

TanStack's Router Generator fills empty route files with the specified
React/Solid/Vue template, but always attempts to insert JSX regardless of file
extension. It then attempts to format the template which fails.

## Workaround

Give it a template with no JSX in it. In `tsr.config.json` for the CLI, or under
`tanstackStart({ router: { ... } })` in `vite.config.ts` for the Vite plugin:

```jsonc
{
  "target": "react",
  "customScaffolding": {
    "routeTemplate": "%%tsrImports%%\n\n%%tsrExportStart%%{}%%tsrExportEnd%%\n"
  }
}
```

Both extensions then scaffold to `createFileRoute('/api/foo')({})`, valid in
either. The catch is that `routeTemplate` is one global string, so `.tsx` routes
lose their component stub too. Per-extension templates aren't possible today.

Failing that, never create an empty route file. Any content at all skips
scaffolding.

## Versions

```
@tanstack/react-router      1.170.38    vite        8.3.0
@tanstack/react-start       1.168.57    typescript  6.0.3
@tanstack/router-plugin     1.168.40    node        v24.13.0
@tanstack/router-generator  1.167.38    pnpm        12.3.4
@tanstack/router-cli        1.167.38
@tanstack/start-plugin-core 1.171.47
```

Stock TanStack CLI scaffold, `default` preset plus the `nitro` add-on. See `.cta.json`.

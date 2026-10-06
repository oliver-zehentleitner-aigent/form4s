# AGENTS.md

## Project

form4s — a Scala 3 library for type-safe web form rendering, decoding, and validation, built on optics (zio-blocks-schema). Library has no entry point; `demo` module contains a runnable zio-http + Bootstrap 5 demo app.

## Build

- **Build tool:** Mill 1.1.7 (use `./mill` launcher; `//| mill-version: 1.1.7` in `build.mill`)
- **Scala:** 3.8.4
- **Modules:** `form4s` (library, `PublishModule`, org `io.github.npcrus`), `demo` (depends on `form4s`), `test` (depends on `form4s`)

### Commands

| Task | Command |
|---|---|
| Compile | `./mill form4s.compile` |
| Test | `./mill test` |
| Format | `./mill mill.scalalib.scalafmt.ScalafmtModule/reformatAll __.sources` |
| Format check | `./mill mill.scalalib.scalafmt.ScalafmtModule/checkFormatAll __.sources` |
| Run demo | `./mill demo.run` |

## Dependencies

- `zio-blocks-schema` 0.0.51 — `Schema` derivation + `Lens` optics (optics core)
- `zio` 2.1.26 — effect system (effectful validation)
- `zio-http` 3.11.2 — HTTP form parsing (`zio.http.Form`) + `template2` HTML DSL

## Source Layout

```
form4s/src/          # package form4s (the library)
  package.scala      # package-level overview doc (entry-point map)
  Field.scala        # Field / LeafField / Path / FormFields.field (optics)
  FieldMacro.scala   # field(_.name) macro implementation
  AutoFields.scala   # AutoFields / FieldsFor / FieldsHolder (auto-derived optics)
  AutoFieldsMacro.scala # auto-derivation macro implementation
  Decoder.scala      # FieldDecoder / FormDecoder (+ derived, Either, map)
  Validator.scala    # Validator (pure) + rule/whole/ruleEach/Validation (effectful)
  ValidatorZIO.scala # ValidatorZIO (effectful leaf validator)
  Render.scala       # RenderContext / Drawn / FormRenderer / draw / drawField / renderForm
  Totality.scala     # compile-time "all fields drawn" check (match types + macro)
  Pipeline.scala     # Incomplete / decodeAndValidate
demo/src/            # package demo — DemoApp (server, port 8080), Model, Views (Bootstrap 5)
test/src/            # package form4s — utest suites + Playwright E2E
```

## Architecture

- **Optics (the shared core)** — `Field[S, A]` (name + get), `LeafField[S, A, N]` (a lens tagged with the compile-time literal field name `N`), `Path[S, A]` (composed). Obtained via `field(_.name)` (a macro in `FormFields[S]`) or `AutoFields[S]` (a `given Conversion` using the companion-object trick to auto-derive every field from `Schema[S]`).
- **Decoding** — `FieldDecoder[A]` decodes one raw string (leaf); `FormDecoder[A]` decodes a `zio.http.Form` into `Either[Seq[DecodingError], A]`. Case classes derive via `FormDecoder.derived` (Mirror, not Magnolia); supports Option/Seq/Either, dot-notation + indexed nesting. `isOptional` controls whether a missing field errors with "Обязательное поле".
- **Validation** — `Validator[A]` (pure leaf, presets like `nonEmpty`/`isEmail`/`min`/`max`/`custom`) and `ValidatorZIO[A]` (effectful leaf). Rules `rule`/`whole`/`ruleEach` produce `Rule[S] = S => ZIO[Any, Nothing, Map[String, Seq[String]]]`, assembled into `Validation[S]` (concurrent via `collectAllPar`). Pure validators auto-lift via `Validator.toZIO`, so `rule(field, Validator.nonEmpty)` and `rule(field, someValidatorZIO)` both work. Error keys are dot-notation paths (`address.city`, `docs.0.typ`); `ruleEach` produces indexed keys.
- **Rendering** — element-agnostic: `drawField(field)(render)` and `draw(fields...)(combine)` build a `FormRenderer[S, E]`; `E` is the caller's output type (e.g. `List[Dom]`), `combine` is the caller's fold (e.g. `_ ++ _`). `draw` runs `checkTotality` at compile time (Mirror + macro) so every field must be drawn exactly once. `RenderContext[A]` carries path/oldValue/errors.
- **Pipeline** — `decodeAndValidate[A](form): ZIO[Any, Incomplete[A], A]` runs `FormDecoder.decode` then `Validation.run`; `Incomplete(errors, oldForm)` has `oldForm = Some` on validation failure, `None` on decode failure, for re-rendering via `renderForm`.

## Conventions

- Scala 3 features: given instances, `inline` macros, match types, higher-kinded types, `transparent inline`
- scalafmt 3.10.7 with `runner.dialect = scala3`
- ScalaDoc on all public API entities; usage examples in `{{ }}` code blocks (written to be snippet-compiler checkable); cross-reference with `[[...]]`
- Validator / FormDecoder error messages are in Russian ("Обязательное поле", "Невозможно преобразовать в число", etc.)
- Test framework: utest 0.8.9 (`test` module, sources in `test/src/`)
- E2E tests use Playwright 1.60.0 (headless Chromium) + zio-http server
- Playwright browser install: `./mill test.runMain com.microsoft.playwright.CLI install chromium`

## MCP Servers

- **Metals** (Scala LSP) — remote at `http://127.0.0.1:33215/mcp` (config in `opencode.jsonc`). Useful tools: `compile-full`/`compile-module`, `test`, `format-file`, `get-source`/`get-docs`/`inspect`, `get-usages`, `glob-search`
- **BrowserMCP** — local via `npx @browsermcp/mcp@latest`

## Keep the Why

This project records the reasoning behind its code with the Keep the Why
skill (https://keepthewhy.com) — the `.keep-the-why` file at the project
root is its config. Before doing anything else in a session, whatever the
first request is about, load the skill: in opencode or Claude Code, invoke the
`keep-the-why` skill (skill tool); in any other agent, read the skill's
`SKILL.md` from where it is installed and follow it, including the
`references/*.md` files it points to for the situation at hand.

Keep the Why's config for this project migrated to .keep-the-why on
2026-10-06 — requires skill version 0.10.0 or later to read it.

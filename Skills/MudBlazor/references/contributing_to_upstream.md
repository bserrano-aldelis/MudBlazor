# Contribuir al upstream — workflow y AGENTS.md compendio

> Esta referencia se carga cuando el usuario quiere abrir un Pull Request al upstream `MudBlazor/MudBlazor`, o cuando un cambio en el fork puede subirse arriba. Cubre: workflow de fork tracking, separación de cambios genéricos vs específicos del cliente, AGENTS.md compendio operativo y plantilla de PR.

[← Volver al SKILL.md](../SKILL.md)

---

## 1. Política dual del fork

El fork `bserrano-aldelis/MudBlazor` tiene una política dual:

| Tipo de cambio | Destino | Distribución |
|---|---|---|
| Bugfix genérico (afecta a cualquier consumer) | **PR al upstream** (`MudBlazor/MudBlazor`) | Una vez merged, descontinuar el código local. |
| Mejora genérica con valor para la comunidad | **PR al upstream** | Tras merge, esperar release y volver al NuGet oficial. |
| Branding Aldelis (paletas, terminología "tarifa") | **Solo en el fork**, rama `dev` o feature | NuGet local `\\10.100.1.180\nuget\`. |
| Componente específico de dominio (`MudAldeworksTariffPicker`) | **Solo en el fork** | NuGet local. |
| Cambio en gris (parte genérica + parte cliente) | **Split**: parte genérica → upstream; parte cliente → fork | Cuando upstream merge la parte genérica, refactor del fork para consumirla. |

**Cómo decidir**: si el cambio fuera útil a otra empresa que use MudBlazor, va arriba. Si es solo para Aldeworks, se queda en el fork.

---

## 2. Sincronizar el fork con upstream (rutina)

```bash
cd C:/dev/MudBlazor

# Trae los últimos cambios del upstream
git fetch upstream

# En la rama dev local
git checkout dev
git merge --ff-only upstream/dev

# Si --ff-only falla, hay commits divergentes en origin/dev → para y consulta
# Si tiene éxito:
git push origin dev
```

**Regla**: nunca hagas `git rebase` o `git merge` no-ff sin avisar. Si el fork divergió, lo más probable es que toque cherry-pick selectivo o nueva rama feature.

---

## 3. Workflow para un PR al upstream

### 3.1. Antes de tocar código

1. Busca en GitHub si el bug ya está reportado o el feature ya está en discusión:
   - Issues: <https://github.com/MudBlazor/MudBlazor/issues>
   - Discussions: <https://github.com/MudBlazor/MudBlazor/discussions>
   - PRs abiertos: <https://github.com/MudBlazor/MudBlazor/pulls>
2. Si no existe, abre un **Issue** o **Discussion** describiendo el problema/propuesta antes de codificar (especialmente para features).
3. Una vez hay luz verde (o tras un día sin respuesta y el cambio es trivial), crea la rama.

### 3.2. Rama y commits

```bash
# Asegúrate de estar al día
git fetch upstream
git checkout dev
git merge --ff-only upstream/dev

# Rama feature/fix
git checkout -b fix/menu-keyboard-trap-12345     # o feat/datagrid-export-csv

# Trabaja → tests → docs → format
# ...

# Commits con mensaje claro (referenciando el issue si aplica)
git add <files>
git commit -m "Fix: keyboard trap when MudMenu is opened with arrow keys (#12345)"
```

**Mensaje del commit (convenciones del upstream)**:

- Inglés.
- Imperativo: `Fix:`, `Add:`, `Refactor:`, `Docs:`, `Test:`, `Chore:`.
- Referencia al issue (`#12345`) si lo hay.
- Cuerpo opcional explicando el "por qué" si no es obvio.

### 3.3. Validación local — checklist (de AGENTS.md)

```bash
# 1. Build limpio del target relevante
MSYS_NO_PATHCONV=1 dotnet build src/MudBlazor.UnitTests/MudBlazor.UnitTests.csproj \
    --no-restore /p:SkipBunCompile=true --nologo

# 2. Tests filtrados (corre múltiples filtros si tu cambio toca varios componentes)
MSYS_NO_PATHCONV=1 dotnet test \
    --project src/MudBlazor.UnitTests/MudBlazor.UnitTests.csproj \
    --no-build --no-restore \
    -- --filter "FullyQualifiedName~MenuTests" \
    --output Normal --no-ansi --hangdump --hangdump-timeout 30s

# 3. Si tocaste docs/examples, valida los tests autogenerados
MSYS_NO_PATHCONV=1 dotnet test \
    --project src/MudBlazor.UnitTests.Docs/MudBlazor.UnitTests.Docs.csproj \
    /p:GenerateDocsTests=true

# 4. Format (UNA vez al final, antes del commit)
cd src
MSYS_NO_PATHCONV=1 dotnet format whitespace --no-restore \
    --include MudBlazor/Components/Menu/MudMenu.razor.cs MudBlazor.UnitTests/Components/MenuTests.cs
```

### 3.4. Push y PR

```bash
git push origin fix/menu-keyboard-trap-12345
```

Abre PR en GitHub:

- Base: `MudBlazor:dev`
- Compare: `bserrano-aldelis:fix/menu-keyboard-trap-12345`
- Título: `Fix: keyboard trap when MudMenu is opened with arrow keys (#12345)`

---

## 4. Plantilla de PR (recomendada)

```markdown
## Description

<Una o dos frases describiendo el cambio.>

Fixes #12345 <!-- si aplica -->

## Type of change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation only

## Behaviour

### Before

<Capturas / GIF / pasos manuales.>

### After

<Capturas / GIF / pasos manuales.>

## Tests

- [ ] Added unit test in `src/MudBlazor.UnitTests/Components/<X>Tests.cs`
- [ ] Added/updated docs page in `src/MudBlazor.Docs/Pages/Components/<X>/`
- [ ] Existing tests pass: `dotnet test --filter ...`

## Checklist

- [ ] Code follows AGENTS.md component authoring rules (`ParameterState<T>`, `CssBuilder`, XML docs, `[Category]`).
- [ ] No hardcoded colors (uses CSS variables / palette tokens).
- [ ] Accessibility: ARIA, keyboard navigation, RTL `[CascadingParameter]` if relevant.
- [ ] No new dependencies.
- [ ] No breaking changes (or `[Obsolete]` with migration path).
- [ ] `dotnet format whitespace` ran on changed files.

## Notes for reviewers

<Cualquier decisión de diseño que el reviewer deba conocer. Casos límite considerados / no considerados.>
```

---

## 5. AGENTS.md compendio operativo

Las reglas que más impactan a un PR upstream, priorizadas:

### 5.1. Scope

- **Cambios enfocados**: target específico, diffs pequeños. **Nada de** rewrites repo-wide salvo petición explícita en el issue.
- **Mejoras follow-up**: anótalas como TODO o discussion separada, **no** las metas en este PR.
- **Sin nuevas dependencias** sin aprobación previa.

### 5.2. Quality gates

- **Warnings as errors**: el build trata warnings como error. **Arregla**, no suprimas.
- **Sin `dotnet clean`** salvo que el incremental esté roto.
- **Tests aislados, paralelos**.
- **Docs en sync** con el cambio si toca API pública o comportamiento.

### 5.3. Component authoring

| Regla | Fuente |
|---|---|
| Parámetros = auto-properties (sin lógica en setters) | AGENTS §"Parameters and state" |
| Reactividad via `ParameterState<T>` + change handlers | AGENTS §"Parameter registration pattern" |
| `CssBuilder` para clases dinámicas | AGENTS §"Styling and naming" |
| CSS variables / design tokens, NUNCA colores hardcoded | AGENTS §"Styling and naming" |
| Nombres positivos (`Gutters` no `DisableGutters`) | AGENTS §"Styling and naming" |
| XML docs en todos los `[Parameter]` públicos | AGENTS §"Public API documentation" |
| `[Category(CategoryTypes.X.Y)]` en parámetros públicos | AGENTS §"Public API documentation" |
| `[CascadingParameter] public bool RightToLeft` cuando layout depende | AGENTS §"Accessibility and behavior" |
| ARIA + keyboard navigation + accessible names | AGENTS §"Accessibility and behavior" |
| Componentes con lógica → bUnit tests + docs page | AGENTS §"Accessibility and behavior" |

### 5.4. Tests

- Naming: sin `Test`/`Async` suffix, sin `Test_` infix, sin `_` final.
- Sin XML doc en métodos de test (autodocumentados).
- bUnit: nunca caches `Find()`, async siempre, `InvokeAsync` para parameter changes.
- Helpers no triviales sí llevan XML doc.
- Tests que tocan estado estático → restaurar en `[TearDown]`, usar `[NonParallelizable]` solo si no hay otra.
- `TimeProvider` / `FakeTimeProvider` mejor que `Task.Delay`.

### 5.5. Docs

- Una página por componente: `src/MudBlazor.Docs/Pages/Components/<Name>/<Name>Page.razor`.
- Ejemplos en `Examples/`. Naming: `<Name>SimpleExample`, `<Name>VariantExample`, `<Name>TwoWayBindingExample`.
- Progresión: simple → variantes comunes → composición → binding → casos límite → avanzado.
- Etiquetas y datos significativos. **No** `Item 1`, `Lorem ipsum` salvo cuando es irrelevante.
- `Code="@nameof(...)"` para referenciar ejemplos (refactor-safe).
- Mostrar código de ejemplos canónicos. Colapsar `ShowCode="false"` los > 15 líneas.
- **Los ejemplos se testean automáticamente**: deben renderizar sin excepciones con defaults.

### 5.6. Formatting

```bash
cd src
dotnet format whitespace --no-restore --include <files>
```

**Una vez al final**, no en cada iteración. Pasa rutas relativas a `src/`.

---

## 6. Breaking changes — cómo manejar

**Evítalos**. Prefiere:

1. **API aditiva**: nuevo parámetro / método con default que preserva comportamiento.
2. **Defaults seguros**: si añades un parámetro nuevo, su default debe replicar el comportamiento anterior.
3. **`[Obsolete]`**: cuando un parámetro se va a renombrar, marca el viejo con `[Obsolete("Use NewName instead. Will be removed in 10.0.")]` y mantén ambos durante un release entero.

Si el breaking es inevitable:

- **Anúncialo en el PR description** explícitamente: "BREAKING: ..."
- Documenta la migración: "Antes: `<Mud X DisableGutters />`. Ahora: `<Mud X Gutters="false" />`."
- Actualiza docs y tests.
- El upstream lo agendará para la próxima major.

---

## 7. Cambios "en gris" — cómo separarlos

A veces un cambio empieza siendo "para Aldeworks" pero contiene una mejora genérica. Patrón recomendado:

1. Identifica la **parte genérica** (que beneficia a todos): bugfix de un componente, soporte de un nuevo `Variant`, mejora de accesibilidad...
2. Identifica la **parte específica** (solo para Aldeworks): paleta custom, una regla de negocio del Tarificador.
3. **PR upstream** con solo la parte genérica.
4. **Cuando upstream haga merge** y libere release, actualiza el fork: borra el código equivalente del fork y consume la versión upstream.
5. La parte específica se queda en el fork.

Ejemplo real: si descubres que `MudAutocomplete` tiene un bug accesible (no se anuncia el resultado al screen reader) Y además quieres añadir un `MudAldeworksProductPicker` que envuelve `MudAutocomplete`:

- PR upstream: `Fix: MudAutocomplete announces results to screen reader`. Sin tocar nada de Aldeworks.
- Fork: `MudAldeworksProductPicker` que consume el `MudAutocomplete` corregido (o, mientras tanto, añade un workaround con `aria-live` propio).

---

## 8. Cambios "del cliente" — workflow de rama

```bash
cd C:/dev/MudBlazor
git fetch upstream
git checkout dev
git merge --ff-only upstream/dev
git push origin dev

# Rama feature del cliente
git checkout -b client/aldelis-palette-presets

# Trabaja
# ...

# Commit + push
git add <files>
git commit -m "Add Aldelis palette presets to MudTheme."
git push origin client/aldelis-palette-presets
```

**No** abras PR a upstream desde estas ramas. Si quieres mergearlas a `dev` (para que el resto del fork lo consuma) sí abre PR interno (`bserrano-aldelis:client/...` → `bserrano-aldelis:dev`).

**Regla del cliente**: las ramas con prefijo `client/` o `aldelis/` no se proponen upstream. Las que **sí** se proponen llevan `feat/`, `fix/`, `docs/`, `test/`, `chore/`.

---

## 9. Distribución del fork como NuGet local

Si Aldeworks consume `MudBlazor` como NuGet (lo más común), el fork se publica al feed local `\\10.100.1.180\nuget\`. Workflow (lo gestiona la skill `nuget-publish`):

```bash
cd C:/dev/MudBlazor
# Bump de versión solo si los cambios no están en upstream:
# Convención: X.Y.Z-aldelis.N (preview con sufijo del cliente)
# Ejemplo: 9.4.2-aldelis.1
```

Mira también `version-bump` skill para el incremento.

Una vez publicado, en `Aldeworks` se actualiza `Directory.Packages.props`:

```xml
<PackageVersion Include="MudBlazor" Version="9.4.2-aldelis.1" />
```

Y se hace pull del nuevo paquete.

---

## 10. Cuándo mergear cambios upstream al fork

Cada release del upstream que se quiera consumir en Aldeworks:

```bash
git fetch upstream
git checkout dev
git merge --ff-only upstream/dev
git push origin dev
```

**Si hay ramas `client/*` con cambios propios**, considera rebasearlas sobre el nuevo `dev`:

```bash
git checkout client/aldelis-palette-presets
git rebase dev
# resuelve conflictos si los hay
git push --force-with-lease origin client/aldelis-palette-presets
```

`--force-with-lease` (no `--force`) protege contra sobrescribir cambios de otros que llegaron después.

---

## 11. Checklist final pre-PR upstream

- [ ] Issue/Discussion abierto con luz verde (para features).
- [ ] Rama desde `dev` actualizada con upstream.
- [ ] Cambio enfocado, diff pequeño.
- [ ] AGENTS.md component authoring rules respetadas.
- [ ] `ParameterState<T>` para parámetros mutables.
- [ ] `CssBuilder` y CSS variables, sin colores hardcoded.
- [ ] XML docs + `[Category]` en parámetros públicos.
- [ ] ARIA + keyboard nav + RTL.
- [ ] Tests en `MudBlazor.UnitTests` que cubren el caso (filtrados pasan).
- [ ] Docs en `MudBlazor.Docs/Pages/Components/<X>/` actualizadas + ejemplos.
- [ ] `dotnet format whitespace` pasado.
- [ ] Sin nuevas dependencias.
- [ ] Sin breaking salvo que sea inevitable y documentado en PR description.
- [ ] PR description completa (descripción, before/after, tests, checklist).

---

[← Volver al SKILL.md](../SKILL.md)

# Tests con bUnit + NUnit — patrones canónicos

> Esta referencia se carga cuando el usuario pregunta sobre tests de componentes Mud, bUnit, simulación de eventos, viewer test components, fakes, o el loop `dotnet test`. Cubre el patrón usado en `src/MudBlazor.UnitTests/` y las reglas del AGENTS.md.

[← Volver al SKILL.md](../SKILL.md)

Stack:
- **NUnit** (test runner) ejecutado via **Microsoft.Testing.Platform** (definido en `global.json`).
- **bUnit** (Razor component testing).
- **AwesomeAssertions** (assertions fluidas, fork de FluentAssertions).
- **AngleSharp** (DOM querying en los componentes renderizados).

Test project: `src/MudBlazor.UnitTests/`. Tests sobre componentes individuales en `src/MudBlazor.UnitTests/Components/<Name>Tests.cs`.

---

## 1. Loop de validación canónico

```bash
# (Bash / MSYS — el MSYS_NO_PATHCONV evita conversiones de path en /p:)
MSYS_NO_PATHCONV=1 dotnet test \
    --project src/MudBlazor.UnitTests/MudBlazor.UnitTests.csproj \
    --no-restore \
    /p:SkipBunCompile=true \
    -- --filter "FullyQualifiedName~ButtonsTests" \
    --output Normal --no-ansi --hangdump --hangdump-timeout 30s
```

**Reglas (AGENTS.md)**:

- Filtra por `FullyQualifiedName~<Class>` para correr solo lo relevante. La suite entera solo al final.
- `/p:SkipBunCompile=true` omite la compilación de assets frontend (Bun). Solo si tu cambio es C# / Razor / tests. **NO** si tocas `TScripts`, `Styles`, `package.json` o `bun.lock`.
- Microsoft.Testing.Platform: pasa runner-options tras `--`. Usa `--hangdump` + `--hangdump-timeout` para detectar tests colgados.
- Para varios filtros sobre el mismo build: `dotnet build --no-restore /p:SkipBunCompile=true` una vez, después `dotnet test --no-build --no-restore -- --filter ...` cada vez.

---

## 2. Fixture base — `BunitTest`

Todas las clases de test extienden `BunitTest` (archivo `src/MudBlazor.UnitTests/Bunit/BunitTest.cs`). Provee `Context`, `JSInterop` mocks, y servicios MudBlazor pre-registrados.

```csharp
using AngleSharp.Dom;
using AngleSharp.Html.Dom;
using AwesomeAssertions;
using Bunit;
using Microsoft.AspNetCore.Components.Web;
using NUnit.Framework;

namespace MudBlazor.UnitTests.Components
{
    [TestFixture]
    public class MyComponentTests : BunitTest
    {
        [Test]
        public void ComponentRendersWithDefaults()
        {
            var comp = Context.RenderComponent<MudMyComponent>();

            comp.Instance.Variant.Should().Be(Variant.Text);
            comp.Markup.Should().Contain("mud-my-component");
        }
    }
}
```

**Naming reglas (AGENTS.md)**:

- **NO** sufijos `Test` o `Async` en el método.
- **NO** `Test_` en medio.
- **NO** terminar en `_`.
- Tests autodocumentados: el nombre debe leerse como una afirmación. `MudButtonShouldRenderAnAnchorIfLinkIsSetAndIsNotDisabled` es canónico.
- Helpers no triviales o reutilizados sí llevan XML doc; tests no.
- Si el test cubre un issue conocido, referencia el número en el nombre o en el comentario adyacente.

---

## 3. Render con parámetros

```csharp
[Test]
public void MudButtonShouldRenderAnAnchorIfLinkIsSetAndIsNotDisabled()
{
    var comp = Context.RenderComponent<MudButton>(parameters => parameters
        .Add(p => p.Href, "https://www.google.com")
        .Add(p => p.Target, "_blank"));

    comp.Instance.HtmlTag.Should().Be("a");
    comp.Markup.Should().Contain("rel=\"noopener\"");

    comp.Markup
        .Replace(" ", string.Empty)
        .Should()
        .StartWith("<a")
        .And.NotContain("__internal_stopPropagation_onclick");
}
```

**`parameters.Add(p => p.X, value)`** es typed: refactor-safe ante renombrados.

**Render fragments / `ChildContent`**:

```csharp
var comp = Context.RenderComponent<MudCard>(parameters => parameters
    .AddChildContent("<p>Hello</p>"));
```

O con builder:

```csharp
var comp = Context.RenderComponent<MudCard>(parameters => parameters
    .AddChildContent(builder =>
    {
        builder.OpenComponent<MudCardContent>(0);
        builder.CloseComponent();
    }));
```

---

## 4. Buscar elementos en el DOM

```csharp
// Single (lanza si no existe)
var button = comp.Find("button");

// Single typed (lanza si no es ese tipo)
var anchor = comp.Find<IHtmlAnchorElement>("a");

// All
var items = comp.FindAll(".mud-list-item");
```

**Regla crítica AGENTS.md**: **NUNCA caches `Find()` o `FindAll()`**. El componente puede re-renderizar tras cualquier interacción → el referencia DOM apunta a un nodo viejo y los aserts dan falsos positivos / negativos.

```csharp
// ❌ MAL
var button = comp.Find("button");
await button.ClickAsync(new MouseEventArgs());
button.GetAttribute("aria-pressed").Should().Be("true");   // referencia stale

// ✅ Bien
await comp.Find("button").ClickAsync(new MouseEventArgs());
comp.Find("button").GetAttribute("aria-pressed").Should().Be("true");
```

---

## 5. Interacciones — siempre async, dentro de `InvokeAsync`

```csharp
// Click
await comp.Find("button").ClickAsync(new MouseEventArgs());

// Input (escribir en un campo)
await comp.Find("input").InputAsync(new ChangeEventArgs { Value = "Hola" });

// Change
await comp.Find("select").ChangeAsync(new ChangeEventArgs { Value = "option-2" });

// Blur (focus out)
await comp.Find("input").BlurAsync(new FocusEventArgs());

// Key down
await comp.Find("input").KeyDownAsync(new KeyboardEventArgs { Key = "Enter" });
```

**Regla AGENTS.md**: prefiere las versiones `*Async` sobre `Click()`, `Input()`, etc. (sync). Las async aseguran flush completo del render antes del siguiente assert.

**Para invocar un método del componente**:

```csharp
await comp.InvokeAsync(() => comp.Instance.OpenAsync());
```

Sin `InvokeAsync`, llamar directamente puede fallar por estar fuera del dispatcher de Blazor.

---

## 6. Cambiar parámetros tras el render inicial

```csharp
var comp = Context.RenderComponent<MudButton>(parameters => parameters
    .Add(p => p.Disabled, false));

// Re-render con nuevo parámetro
comp.SetParametersAndRender(parameters => parameters
    .Add(p => p.Disabled, true));

comp.Markup.Should().Contain("disabled");
```

Equivalente a que el consumer cambie el bind. Útil para verificar reactividad.

---

## 7. Two-way binding y EventCallback

Verificar que un `XChanged` se dispara:

```csharp
[Test]
public async Task ExpansionPanelShouldRaiseExpandedChanged()
{
    var raised = 0;
    var newValue = false;

    var comp = Context.RenderComponent<MudExpansionPanel>(parameters => parameters
        .Add(p => p.Expanded, false)
        .Add(p => p.ExpandedChanged, EventCallback.Factory.Create<bool>(this, v =>
        {
            raised++;
            newValue = v;
        })));

    await comp.Find(".mud-expansion-panel-header").ClickAsync(new MouseEventArgs());

    raised.Should().Be(1);
    newValue.Should().BeTrue();
    comp.Instance.Expanded.Should().BeTrue();
}
```

`EventCallback.Factory.Create<T>(this, lambda)` construye el callback test-side.

---

## 8. Servicios — overrides y mocks

`Context.Services` permite añadir/sobreescribir servicios antes del render:

```csharp
var fakeSnackbar = new FakeSnackbarService();
Context.Services.AddSingleton<ISnackbar>(fakeSnackbar);

var comp = Context.RenderComponent<MyForm>();
await comp.Find("button[type=submit]").ClickAsync(new MouseEventArgs());

fakeSnackbar.Messages.Should().ContainSingle(m => m.Severity == Severity.Success);
```

`BunitTest` ya registra los servicios MudBlazor por defecto. Solo sobreescribe los que tu test necesite.

---

## 9. JS Interop — mocks

bUnit incluye `Context.JSInterop` con los modos:

- **`JSRuntimeMode.Loose`** (default en `BunitTest`): cualquier llamada JS retorna default sin error.
- **`JSRuntimeMode.Strict`**: las llamadas no setupeadas lanzan.

```csharp
[Test]
public async Task FocusBringsTextfieldIntoView()
{
    Context.JSInterop.SetupVoid("mudInput.focus", _ => true);

    var comp = Context.RenderComponent<MudTextField<string>>();
    await comp.Instance.FocusAsync();

    Context.JSInterop.VerifyInvoke("mudInput.focus");
}
```

---

## 10. Tiempo — `TimeProvider` / `FakeTimeProvider`

**Regla AGENTS.md**: prefiere `TimeProvider` / `FakeTimeProvider` sobre `Task.Delay` (que ralentiza tests). Para componentes que dependen del tiempo (debounce, animations):

```csharp
[Test]
public async Task DebouncedInputFiresAfterInterval()
{
    var time = new FakeTimeProvider();
    Context.Services.AddSingleton<TimeProvider>(time);

    var fired = 0;
    var comp = Context.RenderComponent<MudTextField<string>>(parameters => parameters
        .Add(p => p.DebounceInterval, 300)
        .Add(p => p.ValueChanged, EventCallback.Factory.Create<string>(this, _ => fired++)));

    await comp.Find("input").InputAsync(new ChangeEventArgs { Value = "abc" });
    fired.Should().Be(0);

    time.Advance(TimeSpan.FromMilliseconds(350));
    await Task.Yield();   // ceder al dispatcher

    fired.Should().Be(1);
}
```

---

## 11. Aislamiento y paralelismo

**Tests paralelos por defecto** en NUnit. Reglas:

- **No** modifiques estado estático sin restaurarlo en `[TearDown]`.
- Si la modificación de estado estático es inevitable y no quieres ensuciar otros tests: marca el test con `[NonParallelizable]`. Úsalo lo menos posible (cada uno serializa la suite).

```csharp
[TestFixture]
public class StaticAffectingTests : BunitTest
{
    private MudGlobal.Defaults? _originalDefaults;

    [SetUp]
    public void SetUp()
    {
        _originalDefaults = MudGlobal.Defaults;
        MudGlobal.Defaults = new() { /* override */ };
    }

    [TearDown]
    public void TearDown()
    {
        MudGlobal.Defaults = _originalDefaults!;
    }

    [Test, NonParallelizable]
    public void GlobalDefaultsShouldApplyToButton()
    {
        // ...
    }
}
```

---

## 12. Viewer test components

Para escenarios complejos donde armar el árbol Razor inline en el test es engorroso, MudBlazor usa "viewer test components": un archivo `.razor` con la composición a testar, ubicado en `src/MudBlazor.UnitTests.Viewer/TestComponents/<Name>/`. Después el test lo renderiza:

```razor
@* src/MudBlazor.UnitTests.Viewer/TestComponents/Menu/MenuTest1.razor *@
<MudMenu Icon="@Icons.Material.Filled.MoreVert">
    <MudMenuItem OnClick="@(() => _clicked = true)">Edit</MudMenuItem>
    <MudMenuItem>Delete</MudMenuItem>
</MudMenu>

@code {
    private bool _clicked;
}
```

```csharp
[Test]
public async Task MenuClickShouldFire()
{
    var comp = Context.RenderComponent<MenuTest1>();

    await comp.Find(".mud-menu-button").ClickAsync(new MouseEventArgs());
    await comp.FindAll(".mud-menu-item")[0].ClickAsync(new MouseEventArgs());

    // Inspeccionar el state del viewer
    var instance = comp.FindComponent<MenuTest1>().Instance;
    // ...
}
```

**Reglas (AGENTS.md)**:

- Naming: empieza con el prefijo del componente, casing correcto, termina en `Test` opcionalmente con índice (`MenuTest1`).
- **Máx 40 caracteres** en el nombre del archivo. Prefiere nombres concisos.
- Añade un viewer test component **solo si** el escenario es muy complicado de expresar inline.

---

## 13. Snapshot vs comportamiento

**Regla AGENTS.md**: testar comportamiento, no snapshots de HTML completo. Aserta lo que importa: estado, callbacks, atributos críticos, accesibilidad. NO compares strings de markup enteros — cualquier cambio cosmético rompe el test sin razón.

```csharp
// ❌ MAL — frágil
comp.Markup.Should().Be("<button class=\"mud-button mud-button-text mud-ripple ...\">Click me</button>");

// ✅ Bien — testa el comportamiento concreto
var button = comp.Find("button");
button.GetAttribute("type").Should().Be("button");
button.ClassList.Should().Contain("mud-button");
button.TextContent.Trim().Should().Be("Click me");
```

---

## 14. Tests de docs ejemplos (`MudBlazor.UnitTests.Docs`)

Los ejemplos en `src/MudBlazor.Docs/Pages/Components/<X>/Examples/` se ejecutan automáticamente como tests. Solo se generan cuando se solicitan explícitamente:

```bash
dotnet test --project src/MudBlazor.UnitTests.Docs/MudBlazor.UnitTests.Docs.csproj /p:GenerateDocsTests=true
```

Los archivos generados (`Generated/*.generated.cs`) **no se editan a mano**. Si un ejemplo lanza, arregla el ejemplo, no el test generado.

**Implicación**: cualquier ejemplo que añadas a docs **debe renderizar sin excepciones** con datos por defecto.

---

## 15. Checklist al añadir/modificar un test

- [ ] Naming sin sufijos `Test` ni `Async`, sin `_` final.
- [ ] Nombre autodocumentado (lee como afirmación).
- [ ] Sin XML doc en el método del test.
- [ ] Un solo concepto por test (si tu test hace 5 cosas, son 5 tests).
- [ ] No cachees `Find()` / `FindAll()`.
- [ ] Async siempre (`ClickAsync`, `InputAsync`, etc.).
- [ ] Si tocas estado estático, restáuralo en `[TearDown]`.
- [ ] Si añades dependencia (`TimeProvider`, mock service), explica en comentario solo si no es obvio.
- [ ] Verifica con `dotnet test --filter "FullyQualifiedName~<TuTestClass>"` antes del commit.

---

## 16. Diagnóstico — test que cuelga o falla intermitente

1. Añade `--hangdump --hangdump-timeout 30s` al runner. Si cuelga, genera dump.
2. Verifica que no haya `Task.Delay` o esperas absolutas — sustitúyelas por `FakeTimeProvider`.
3. Verifica que toda interacción esté en `InvokeAsync` o sea `*Async`.
4. Si re-querea el DOM tras cada interacción, las staleness desaparecen.
5. Si solo falla en CI: probablemente race con render — pon `await comp.WaitForState(() => ...)` o `await comp.WaitForAssertion(() => ...)`.

```csharp
await comp.WaitForAssertion(() =>
{
    comp.Find("[role='alert']").TextContent.Should().Be("Done");
}, timeout: TimeSpan.FromSeconds(3));
```

---

[← Volver al SKILL.md](../SKILL.md)

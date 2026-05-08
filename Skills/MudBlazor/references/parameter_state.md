# ParameterState framework — guía completa

> Esta referencia se carga cuando creas o extiendes un componente en el fork. El `ParameterState<T>` es el framework que MudBlazor 9.x usa para gestionar parámetros con reactividad y two-way binding sin lógica en setters. Es **obligatorio** para parámetros mutables en todos los componentes nuevos del fork.

[← Volver al SKILL.md](../SKILL.md)

Fuente: `C:\dev\MudBlazor\src\MudBlazor\State\` (carpeta entera). Tipos clave: `ParameterState<T>`, `RegisterParameterBuilder<T>`, `ParameterRegistrationBuilderScope`, `IParameterChangedHandler`.

---

## 1. Por qué existe

Blazor por defecto entrega los parámetros como `[Parameter] public T Value { get; set; }`. Si quieres reaccionar a cambios, opciones canónicas son:

1. **Lógica en el setter** (`set { _value = value; OnChange(); }`) — el componente se vuelve impredecible: el setter dispara mid-render, los renders pueden re-entrar.
2. **`OnParametersSet` / `SetParametersAsync`** — funciona pero te obliga a comparar todos los parámetros manualmente para saber cuál cambió. Crece linealmente con la cantidad de parámetros y es propenso a olvidos.
3. **`ParameterState<T>`** — solución idiomática de MudBlazor: registra cada parámetro mutable, declara su getter/eventCallback/changeHandler, y el framework se encarga de detectar cambios eficientemente.

Reglas oficiales del [`AGENTS.md`](../../AGENTS.md):

> Component parameters must be auto-properties only. Do not put logic in getters or setters. Do not overwrite component parameters directly. Use the backing `ParameterState<T>` and update through `.Value` or `SetValueAsync`. Do not set other component parameters via `@ref` (`BL0005`). Use declarative binding instead.

---

## 2. Patrón canónico (single parameter)

```csharp
public partial class MudExpansionPanel : MudComponentBase
{
    private readonly ParameterState<bool> _expandedState;

    [Parameter]
    [Category(CategoryTypes.ExpansionPanel.Behavior)]
    public bool Expanded { get; set; }

    [Parameter]
    public EventCallback<bool> ExpandedChanged { get; set; }

    public MudExpansionPanel()
    {
        using var registerScope = CreateRegisterScope();

        _expandedState = registerScope.RegisterParameter<bool>(nameof(Expanded))
            .WithParameter(() => Expanded)
            .WithEventCallback(() => ExpandedChanged)
            .WithChangeHandler(OnExpandedChangedAsync);
    }

    private Task OnExpandedChangedAsync(ParameterChangedEventArgs<bool> args)
    {
        // Reactividad AQUÍ — no en el setter del parámetro.
        // args.LastValue / args.Value disponibles.
        return Task.CompletedTask;
    }

    public Task ToggleAsync()
    {
        // El cambio se propaga al consumer via ExpandedChanged automáticamente.
        return _expandedState.SetValueAsync(!_expandedState.Value);
    }
}
```

**Anatomía**:

- `[Parameter] public bool Expanded { get; set; }` — auto-property pura, sin lógica.
- `[Parameter] public EventCallback<bool> ExpandedChanged { get; set; }` — pareja del two-way binding (`@bind-Expanded`).
- `private readonly ParameterState<bool> _expandedState` — backing field; lo declaramos `readonly` porque se asigna una sola vez (en el ctor).
- `CreateRegisterScope()` — crea un scope de registro al cual se añaden todos los parámetros del componente. Va dentro de un `using` para asegurarse de que se cierra correctamente al final del ctor.
- `.WithParameter(() => Expanded)` — getter del parámetro de la clase. Se usa una lambda para evitar capturar el valor en tiempo de construcción (queremos el valor *actual* en cada lectura).
- `.WithEventCallback(() => ExpandedChanged)` — getter del callback bidireccional.
- `.WithChangeHandler(OnExpandedChangedAsync)` — handler que se invoca cuando el framework detecta cambio en el parámetro.

**Cómo se consume en el código del componente**:

```csharp
// Lectura: usa _expandedState.Value (NO Expanded directamente)
if (_expandedState.Value) { ... }

// Escritura: usa SetValueAsync (NO Expanded = X)
await _expandedState.SetValueAsync(true);
```

---

## 3. Anotación `[ParameterState]` (recomendada)

Las propiedades cuyo valor está gestionado por `ParameterState<T>` se marcan con `[Parameter, ParameterState]` para que el lector entienda al instante el contrato:

```csharp
[Parameter, ParameterState]
public bool Expanded { get; set; }
```

El analyzer del repo no fuerza esta anotación todavía, pero es la convención del AGENTS.md y mejora la legibilidad.

---

## 4. Multi-parameter scope

Cuando un componente tiene varios parámetros gestionados, **un solo scope** los agrupa:

```csharp
public MudCheckBox()
{
    using var registerScope = CreateRegisterScope();

    _valueState = registerScope.RegisterParameter<bool?>(nameof(Value))
        .WithParameter(() => Value)
        .WithEventCallback(() => ValueChanged)
        .WithChangeHandler(OnValueChangedAsync);

    _checkedState = registerScope.RegisterParameter<bool?>(nameof(Checked))
        .WithParameter(() => Checked)
        .WithEventCallback(() => CheckedChanged)
        .WithChangeHandler(OnCheckedChangedAsync);

    _disabledState = registerScope.RegisterParameter<bool>(nameof(Disabled))
        .WithParameter(() => Disabled);                  // sin handler: solo lectura
}
```

**Reglas**:

- Un solo `CreateRegisterScope()` por ctor (NO uno por parámetro).
- Si el parámetro **no** necesita reaccionar a cambios (es solo lectura por el componente), omite `.WithChangeHandler(...)`.
- Si el parámetro **no** es two-way bindable, omite `.WithEventCallback(...)`.

---

## 5. Comparación custom

Por defecto el framework usa `EqualityComparer<T>.Default` para detectar cambios. Si tu tipo necesita comparación custom (ej. dos objetos "iguales en contenido" pero distintas referencias), pásala con `.WithComparer(...)`:

```csharp
_filterState = registerScope.RegisterParameter<FilterDefinition<T>>(nameof(Filter))
    .WithParameter(() => Filter)
    .WithEventCallback(() => FilterChanged)
    .WithChangeHandler(OnFilterChangedAsync)
    .WithComparer(() => new FilterDefinitionEqualityComparer<T>());
```

`WithComparer` también acepta versión "swappable": cambia la comparison lógica en runtime sin re-registrar el parámetro.

---

## 6. Change handlers — async vs sync

Tres firmas válidas para el handler:

```csharp
// Sync, sin args
_state.WithChangeHandler(OnChanged);
private void OnChanged() { ... }

// Sync, con args
_state.WithChangeHandler(OnChanged);
private void OnChanged(ParameterChangedEventArgs<T> args) { ... }

// Async, con args (el más común)
_state.WithChangeHandler(OnChangedAsync);
private Task OnChangedAsync(ParameterChangedEventArgs<T> args) { ... }
```

`ParameterChangedEventArgs<T>` expone:

- `LastValue` — valor anterior.
- `Value` — valor nuevo.
- `ParameterName` — nombre del parámetro (`nameof(Value)`).

**No bloquees** en handlers async (no hagas `Task.Wait()` o `.Result`). Si necesitas serializar varias llamadas, usa un `SemaphoreSlim` o cola interna.

---

## 7. Lifecycle e inicialización

`ParameterState<T>.Value` está **disponible desde el primer `OnParametersSet`**. Antes de eso (constructor, `OnInitialized`), el valor es el default de `T`.

Para componentes que hacen trabajo costoso solo la primera vez:

```csharp
public bool IsInitialized { get; private set; }

protected override void OnParametersSet()
{
    base.OnParametersSet();

    if (!IsInitialized)
    {
        // First-time init que depende de parámetros (ej. construir un cache derivado).
        BuildCache();
        IsInitialized = true;
    }
}
```

> No confundir con `MudComponentBase.IsInitialized` interno del framework — esa flag es nuestra, propia del componente.

---

## 8. Two-way binding declarativo (consumer side)

Tu componente expone `Expanded` + `ExpandedChanged`. El consumer escribe:

```razor
<MudExample @bind-Expanded="vm.IsExpanded" />
```

Equivalente a:

```razor
<MudExample Expanded="vm.IsExpanded"
            ExpandedChanged="@((bool v) => vm.IsExpanded = v)" />
```

El framework Razor genera el callback automáticamente cuando ve `@bind-X`.

---

## 9. Cuándo NO usar `ParameterState<T>`

- **Parámetros constantes en el lifecycle del componente**: `Variant`, `Color`, `Size` que el consumer setea una vez y no cambia. `[Parameter] public Variant Variant { get; set; }` simple es suficiente.
- **Cascading parameters**: usan `[CascadingParameter]` con su propio mecanismo. NO los registres como `ParameterState`.
- **Render fragments** (`RenderFragment ChildContent`): no son parámetros mutables del usuario, son contenido pasado al render.

---

## 10. `@ref` y `BL0005` — qué NO hacer

**Prohibido** setear parámetros de un componente vía `@ref`:

```razor
@* MAL — el analyzer BL0005 lo detecta como error *@
<MudExample @ref="_example" />

@code {
    private MudExample _example = default!;

    private void DoSomething()
    {
        _example.Expanded = true;   // ❌ BL0005: no setear parámetros via @ref
    }
}
```

**Bien** — usa binding declarativo con un campo en el padre:

```razor
<MudExample @bind-Expanded="_isExpanded" />

@code {
    private bool _isExpanded;

    private void DoSomething() => _isExpanded = true;
}
```

O si necesitas **invocar un método público del componente** (no setear parámetro), eso sí está permitido via `@ref`:

```razor
<MudExample @ref="_example" />

@code {
    private MudExample _example = default!;

    private async Task DoSomething()
    {
        await _example.ToggleAsync();   // ✅ método público OK
    }
}
```

---

## 11. Ejemplo extremo — DataGrid filter parameter (caso real del repo)

`MudDataGrid<T>` tiene un parámetro `FilterDefinitions` que cambia frecuentemente cuando el usuario filtra. El `ParameterState<T>` evita re-renders innecesarios:

```csharp
private readonly ParameterState<List<IFilterDefinition<T>>> _filterDefinitionsState;

[Parameter]
[Category(CategoryTypes.DataGrid.Filtering)]
public List<IFilterDefinition<T>> FilterDefinitions { get; set; } = new();

[Parameter]
public EventCallback<List<IFilterDefinition<T>>> FilterDefinitionsChanged { get; set; }

public MudDataGrid()
{
    using var registerScope = CreateRegisterScope();

    _filterDefinitionsState = registerScope.RegisterParameter<List<IFilterDefinition<T>>>(nameof(FilterDefinitions))
        .WithParameter(() => FilterDefinitions)
        .WithEventCallback(() => FilterDefinitionsChanged)
        .WithChangeHandler(OnFilterDefinitionsChangedAsync)
        .WithComparer(() => new FilterDefinitionsListComparer<T>());
}

private async Task OnFilterDefinitionsChangedAsync(ParameterChangedEventArgs<List<IFilterDefinition<T>>> args)
{
    // Re-evaluar la query, NO directamente en el setter del parámetro.
    await ReloadServerDataAsync();
}
```

Nota la `WithComparer`: comparar listas por referencia es inútil aquí (el consumer típicamente reasigna la lista entera). El comparer custom recorre los items.

---

## 12. Test de un componente con `ParameterState<T>`

`bUnit` interactúa con `ParameterState<T>` de forma transparente — el test no necesita saber del framework interno:

```csharp
[Test]
public async Task ExpansionPanelShouldRaiseExpandedChanged()
{
    var raised = 0;
    var comp = Context.RenderComponent<MudExpansionPanel>(parameters => parameters
        .Add(p => p.Expanded, false)
        .Add(p => p.ExpandedChanged, EventCallback.Factory.Create<bool>(this, _ => raised++)));

    // Click el header → toggle
    await comp.Find(".mud-expansion-panel-header").ClickAsync(new MouseEventArgs());

    raised.Should().Be(1);
    comp.Instance.Expanded.Should().BeTrue();   // el state interno se sincronizó
}
```

Más patrones bUnit → `references/testing_with_bunit.md`.

---

## 13. Checklist al añadir un parámetro nuevo a un componente

- [ ] Auto-property pura (`{ get; set; }`), sin lógica.
- [ ] Anotada con `[Parameter]` y opcionalmente `[ParameterState]`.
- [ ] `[Category(CategoryTypes.X.Y)]` para clasificación en docs auto-generadas.
- [ ] `<summary>` XML describiendo comportamiento (no "Gets or sets").
- [ ] `<remarks>` con default si aplica.
- [ ] Si es two-way bindable: añadir `EventCallback<T> XChanged` en pareja.
- [ ] Si es mutable durante el lifecycle: registrar como `ParameterState<T>` en el ctor.
- [ ] Change handler en `OnXChangedAsync`, NO en el setter.
- [ ] Test bUnit que verifica que el parámetro se respeta y dispara `XChanged`.
- [ ] Página de docs en `MudBlazor.Docs/Pages/Components/<X>/` actualizada con un ejemplo del nuevo parámetro.

---

[← Volver al SKILL.md](../SKILL.md)

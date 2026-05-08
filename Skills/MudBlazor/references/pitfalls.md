# Pitfalls — catálogo de problemas comunes

> Esta referencia se carga cuando el usuario reporta un error, un comportamiento inesperado, o pregunta "¿por qué X no funciona?". Cubre los gotchas más frecuentes con MudBlazor 9.x, especialmente los que aparecen al integrarlo en una app Blazor Web App (.NET 10) como Aldeworks.

[← Volver al SKILL.md](../SKILL.md)

---

## 1. Static SSR no soportado

**Síntoma**: Componentes Mud que renderizan en blanco o con estilos rotos. Eventos (`OnClick`) no responden. La consola muestra warnings tipo "no JSRuntime available".

**Causa**: MudBlazor requiere render mode interactivo (Server, WebAssembly o Auto). En Blazor Web App (.NET 8+) por defecto las páginas son Static SSR.

**Solución**: añade `@rendermode InteractiveServer` (o `InteractiveAuto`/`InteractiveWebAssembly`) a la página o al componente raíz:

```razor
@page "/my-page"
@rendermode InteractiveServer

<MudContainer>
    <!-- componentes Mud aquí -->
</MudContainer>
```

O global en `App.razor`:

```razor
<HeadOutlet @rendermode="InteractiveServer" />
<Routes @rendermode="InteractiveServer" />
```

En Aldeworks ya está configurado en `App.razor`. Si encuentras esto, revisa si la página tiene un `@rendermode` distinto que esté bloqueando.

---

## 2. `OnInitializedAsync` se ejecuta dos veces (prerender)

**Síntoma**: las llamadas async dentro de `OnInitializedAsync` se ejecutan dos veces — una en SSR (sin DOM, sin JSRuntime), otra al hidratar. Si tu lógica depende del DOM, falla en la primera.

**Causa**: Blazor Web App con `prerender=true` (default) ejecuta el componente dos veces: una server-side para SSR, otra client-side al hidratar.

**Solución (regla del cliente — memoria `feedback_blazor_ssr_prerender`)**:

```csharp
protected override async Task OnInitializedAsync()
{
    if (!RendererInfo.IsInteractive)
    {
        return;   // Skip first (SSR) pass; espera a la hidratación.
    }

    var data = await Service.LoadAsync();
    StateHasChanged();
}
```

Aplica este guard **siempre** en componentes de Aldeworks que hagan llamadas async en `OnInitializedAsync`.

---

## 3. `EditContext.IsModified()` y el botón Save

**Síntoma**: el botón Save siempre está habilitado o nunca cambia tras un Save success. El usuario puede pulsar "Guardar" sobre el mismo formulario sin cambios → toaster duplicado.

**Causa**: Blazor por defecto detecta modificación a nivel de input. Si actualizas el VM programáticamente (sin que el usuario teclee), `IsModified()` no se entera; igual al revés, tras Save success no resetea solo.

**Solución (regla del cliente — memoria `feedback_save_button_dirty`)**:

```razor
<EditForm EditContext="@editContext" OnValidSubmit="HandleSubmitAsync">
    <MudButton ButtonType="ButtonType.Submit"
               Disabled="@(!editContext.IsModified())">
        Guardar
    </MudButton>
</EditForm>

@code {
    private EditContext editContext = default!;
    private VmType vm = new();

    protected override void OnInitialized()
    {
        editContext = new EditContext(vm);
    }

    private async Task HandleSubmitAsync()
    {
        var result = await mediator.Send(new SaveCommand(vm));
        if (result.IsSuccess)
        {
            editContext.MarkAsUnmodified();   // resetea el dirty state
        }
    }

    // Para cambios programáticos:
    private void ApplyDefaults()
    {
        vm.Foo = "default";
        editContext.NotifyFieldChanged(FieldIdentifier.Create(() => vm.Foo));
    }
}
```

---

## 4. Sticky headers — NO `position: sticky` con MudBlazor

**Síntoma**: pones `position: sticky` en el header de una tabla y el comportamiento es errático: a veces no se queda fijo, a veces tapa otros elementos al scrollear.

**Causa**: `position: sticky` requiere que **todos los ancestros** sean overflow visible y que el container directo tenga el scroll. En layouts Mud anidados (MudPaper > MudCard > MudCardContent...), suele haber un `overflow: hidden` o `overflow: auto` interpuesto que rompe el sticky.

**Solución (regla del cliente — memoria `feedback_sticky_headers`)**: usa flex column + scroll interno en el body:

```razor
<MudPaper Style="height: 600px; display: flex; flex-direction: column;">
    <div class="aldeworks-list-header">
        <!-- header fijo -->
    </div>
    <div class="aldeworks-list-body" style="flex: 1; overflow-y: auto;">
        <!-- contenido scrolleable -->
    </div>
</MudPaper>
```

`MudDataGrid` tiene `FixedHeader="true"` que gestiona esto internamente. **Úsalo** si vas a hacer un listado:

```razor
<MudDataGrid T="Item" Items="@items" FixedHeader="true" Height="600px">
    <!-- columnas -->
</MudDataGrid>
```

---

## 5. `ParameterState<T>.Value` en `OnInitialized` es default

**Síntoma**: dentro de `OnInitialized` lees `_someState.Value` y siempre es `false` / `0` / `null`, incluso aunque el consumer pase un valor.

**Causa**: `ParameterState<T>.Value` se sincroniza tras `OnParametersSet`, **no** en `OnInitialized`. En `OnInitialized` los parámetros aún no han llegado.

**Solución**: mueve la lógica que depende de parámetros a `OnParametersSet` (con guard de "primer paso" si solo lo quieres una vez):

```csharp
private bool _initialized;

protected override void OnParametersSet()
{
    base.OnParametersSet();

    if (!_initialized)
    {
        // Lógica que depende de _someState.Value
        _initialized = true;
    }
}
```

---

## 6. `MudSelect<T>` con tipos complejos no compara bien

**Síntoma**: seleccionas un item, el dropdown se cierra, pero el `MudSelect` no muestra el item como seleccionado. Si reabres, no hay item destacado.

**Causa**: `MudSelect<T>` usa `EqualityComparer<T>.Default`. Para tipos custom (records con `init`, clases sin `Equals` override), el round-trip a través del binding crea nuevas referencias y la comparación falla.

**Solución 1**: Usar `record` en lugar de `class` (records tienen value equality automático):

```csharp
public record UserDto(int Id, string FullName);
```

**Solución 2**: usar id primitive (`int`, `string`) en lugar del objeto entero:

```razor
<MudSelect T="int" @bind-Value="vm.SelectedUserId" ToStringFunc="@(id => GetUserName(id))">
    @foreach (var user in users)
    {
        <MudSelectItem T="int" Value="@user.Id">@user.FullName</MudSelectItem>
    }
</MudSelect>

@code {
    private string GetUserName(int id) => users.FirstOrDefault(u => u.Id == id)?.FullName ?? string.Empty;
}
```

**Solución 3**: sobreescribir `Equals` y `GetHashCode` (útil cuando no puedes cambiar la clase a record):

```csharp
public class UserDto : IEquatable<UserDto>
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;

    public bool Equals(UserDto? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as UserDto);
    public override int GetHashCode() => Id.GetHashCode();
}
```

---

## 7. `MultiSelection="true"` con `@bind-Value` (en lugar de `@bind-SelectedValues`)

**Síntoma**: en multi-select, el primer click marca el item, pero los siguientes desmarcan el anterior y marcan solo el nuevo. Comportamiento de single-select aunque `MultiSelection="true"`.

**Causa**: con `MultiSelection="true"`, el binding correcto es `@bind-SelectedValues` (plural), no `@bind-Value`.

**Solución**:

```razor
<MudSelect T="int"
           Label="Roles"
           MultiSelection="true"
           @bind-SelectedValues="vm.SelectedRoleIds">
    @* ... *@
</MudSelect>
```

`vm.SelectedRoleIds` debe ser `IEnumerable<int>` (típicamente `HashSet<int>` para conjuntos).

---

## 8. `MudNumericField<decimal>` y la cultura

**Síntoma**: el usuario teclea `4.59` esperando 4.59 y el componente lo convierte en `459` (cuatrocientos cincuenta y nueve) o lanza error de parsing.

**Causa**: la cultura del request es `es-ES` (`,` como decimal). El usuario tecleó `.` (cultura `en-US`) → parsing falla y queda lo último válido.

**Solución 1 (preferida en formularios técnicos)**: usar `InvariantCulture`:

```razor
<MudNumericField T="decimal"
                 @bind-Value="vm.Margin"
                 Culture="@CultureInfo.InvariantCulture"
                 Format="F2" />
```

**Solución 2**: educar al usuario y usar la cultura de la app:

```razor
<MudNumericField T="decimal"
                 @bind-Value="vm.Margin"
                 Culture="@CultureInfo.GetCultureInfo("es-ES")"
                 Format="F2" />
```

Indica al usuario "use coma como decimal" en placeholder o helper text.

---

## 9. `Min`/`Max` en `MudNumericField<decimal>` requieren sufijo `m`

**Síntoma**: error de compilación `cannot convert double to decimal` o el rango no se respeta.

**Causa**: `Min="0"` y `Max="99.99"` se infieren como `double`. Cuando `T="decimal"`, el implicit conversion falla o el operator de comparison decimal-vs-double da resultados raros.

**Solución**:

```razor
<MudNumericField T="decimal"
                 Min="0m"
                 Max="99.99m"
                 Step="0.01m" />
```

Mismo principio para `float` (`f`) y `long` (`L`).

---

## 10. `MudDialog` no se abre / no aparece

**Síntoma**: llamas a `DialogService.ShowAsync<MyDialog>()` y no pasa nada.

**Causa más común**: olvidaste poner `<MudDialogProvider />` en el árbol (típicamente en `MainLayout.razor`).

**Solución**:

```razor
@* MainLayout.razor *@
<MudThemeProvider />
<MudPopoverProvider />
<MudDialogProvider />          @* ← imprescindible *@
<MudSnackbarProvider />

<MudLayout>
    @Body
</MudLayout>
```

Otras causas:

- Llamas `ShowAsync` sin `await`: la promise no se ejecuta. Siempre `await DialogService.ShowAsync<...>(...)`.
- El componente `MyDialog` no hereda de `MudDialog` ni tiene `<MudDialog>...</MudDialog>` como root: el provider no sabe cómo renderizarlo.

---

## 11. Snackbar duplicado en pantallas que se re-renderizan rápido

**Síntoma**: un toaster aparece N veces tras una operación.

**Causa**: `Snackbar.Add` se llama desde un evento que dispara dos veces (por SSR + hidratación, por ejemplo).

**Solución**: configura `PreventDuplicates`:

```csharp
builder.Services.AddMudServices(config =>
{
    config.SnackbarConfiguration.PreventDuplicates = true;
});
```

O añade el guard `if (!RendererInfo.IsInteractive) return;` (ver pitfall #2).

---

## 12. `@ref` para setear parámetros — `BL0005` analyzer error

**Síntoma**: build error `BL0005: Component parameter 'X' should not be set outside of its component`.

**Causa**: estás haciendo `_componentRef.SomeParameter = value` desde el padre. Esto bypaseea el sistema de parámetros de Blazor y deja al componente en un estado inconsistente.

**Solución**: usa binding declarativo:

```razor
<MudExample @bind-Expanded="_isExpanded" />

@code {
    private bool _isExpanded;

    private void Toggle() => _isExpanded = !_isExpanded;
}
```

Si necesitas invocar **un método público** del componente, eso sí es válido vía `@ref`:

```razor
<MudExample @ref="_example" />

@code {
    private MudExample _example = default!;

    private async Task ToggleAsync() => await _example.ToggleAsync();
}
```

---

## 13. RTL no se aplica correctamente

**Síntoma**: pones `RightToLeft="true"` en el componente y el texto sigue alineado a la izquierda.

**Causa**: en MudBlazor 9.x el `RightToLeft` se cascadea desde un ancestro (`MudRTLProvider` o `MudThemeProvider`). No es un parámetro local del componente.

**Solución**: añade `<MudRTLProvider RightToLeft="true">` en el árbol (típicamente en `MainLayout.razor`, condicional al idioma del usuario):

```razor
<MudRTLProvider RightToLeft="@_isRtl">
    <MudThemeProvider Theme="@_theme" IsDarkMode="@_isDark" />
    <!-- resto -->
</MudRTLProvider>
```

En componentes propios que dependen de la dirección, declara:

```csharp
[CascadingParameter]
public bool RightToLeft { get; set; }
```

Y úsalo en el `CssBuilder`:

```csharp
protected string Classname =>
    new CssBuilder("aldeworks-xxx")
        .AddClass("aldeworks-xxx-rtl", RightToLeft)
        .Build();
```

---

## 14. `MudDataGrid<T>` server-side rendering — `LoadServerData` se llama dos veces

**Síntoma**: tu `Func<GridState<T>, Task<GridData<T>>>` se invoca dos veces al cargar la página, generando dos llamadas a la BD.

**Causa**: Blazor Web App con prerender. Igual que pitfall #2.

**Solución**: dentro del callback, guard:

```csharp
private async Task<GridData<MyItem>> LoadServerData(GridState<MyItem> state)
{
    if (!RendererInfo.IsInteractive)
    {
        return new GridData<MyItem> { Items = Array.Empty<MyItem>(), TotalItems = 0 };
    }

    var page = await mediator.Send(new ListItemsQuery(state.Page + 1, state.PageSize));
    return new GridData<MyItem>
    {
        Items = page.Data.Items,
        TotalItems = page.Data.TotalCount
    };
}
```

---

## 15. Estilos no se aplican tras `dotnet build`

**Síntoma**: cambios en SCSS o CSS no aparecen en el browser tras un build.

**Causa**: usaste `/p:SkipBunCompile=true` en el build, que omite la compilación de assets frontend.

**Solución**: para cambios de estilos, **NO** uses ese flag:

```bash
dotnet build src/MudBlazor/MudBlazor.csproj
# Sin /p:SkipBunCompile=true
```

Si Bun falla, restaura el tool:

```bash
dotnet tool restore --tool-manifest .config/dotnet-tools.json
```

---

## 16. `EditForm` + `MudTextField` + DataAnnotations no valida

**Síntoma**: el VM tiene `[Required]`, `[StringLength]`, etc. pero al pulsar Submit pasa la validación y el handler recibe valores inválidos.

**Causa más común**: olvidaste `<DataAnnotationsValidator />` dentro del `EditForm`.

**Solución**:

```razor
<EditForm Model="@vm" OnValidSubmit="HandleSubmitAsync">
    <DataAnnotationsValidator />     @* ← imprescindible *@

    <MudTextField @bind-Value="vm.Name" For="@(() => vm.Name)" Label="Nombre" />
    @* ... *@

    <MudButton ButtonType="ButtonType.Submit">Guardar</MudButton>
</EditForm>
```

Otras causas:

- `For="@(() => vm.Name)"` mal escrito (lambda con `()` extra). Debe ser `() => vm.Name`.
- El VM no es público (validador no lo introspectiona).
- `OnValidSubmit` no está conectado (usaste `OnSubmit` en su lugar — ese se dispara siempre, válido o no).

---

## 17. Validación FluentValidation no engancha con `For`

**Síntoma**: configuras un validador FluentValidation, pero el `MudTextField` no muestra errores ni bloquea Submit.

**Causa**: FluentValidation con Blazor requiere un componente bridge (`<FluentValidationValidator />` del paquete `Blazored.FluentValidation`) en lugar de `<DataAnnotationsValidator />`. **MudBlazor no integra FluentValidation directamente vía `For`** — `For` es para DataAnnotations.

**Solución 1**: Usa `Validation` parameter directamente (no `For`):

```razor
<MudTextField @bind-Value="vm.Email"
              Validation="@(new Func<string, IEnumerable<string>>(ValidateEmail))" />

@code {
    private IValidator<string> emailValidator = new InlineValidator<string>().RuleFor(x => x).EmailAddress().And;
    private IEnumerable<string> ValidateEmail(string email)
    {
        var result = emailValidator.Validate(email);
        return result.Errors.Select(e => e.ErrorMessage);
    }
}
```

**Solución 2**: usa `MudForm` (no `EditForm`) con FluentValidation:

```razor
<MudForm @ref="form" Validation="@(modelValidator.ValidateValue)">
    <MudTextField @bind-Value="vm.Email" For="@(() => vm.Email)" />
    <MudButton OnClick="@(async () => { await form.Validate(); if (form.IsValid) await Submit(); })">
        Guardar
    </MudButton>
</MudForm>
```

Este patrón está documentado en `mudblazor.com/components/form` con ejemplo `FluentValidationFormExample`.

---

## 18. `MudDatePicker` y la cultura de fecha

**Síntoma**: el usuario selecciona una fecha en el calendario, el campo muestra `5/8/2026` pero internamente queda como `8/5/2026` (día y mes invertidos).

**Causa**: el `Format` por defecto es `"dd/MM/yyyy"` en culturas europeas y `"M/d/yyyy"` en US. El parsing va contra la cultura del request.

**Solución**: setea `Culture` y `DateFormat` explícitos:

```razor
<MudDatePicker @bind-Date="vm.BirthDate"
               Culture="@CultureInfo.GetCultureInfo("es-ES")"
               DateFormat="dd/MM/yyyy" />
```

---

## 19. Formularios largos: `MudTabs` reset el VM al cambiar pestaña

**Síntoma**: usuario rellena la pestaña 1, cambia a la 2, vuelve a la 1 → la 1 está en blanco.

**Causa**: por defecto `MudTabs` re-renderiza el panel activo y desmonta los inactivos (`KeepPanelsAlive="false"`). Los componentes pierden estado local.

**Solución**: `KeepPanelsAlive="true"`:

```razor
<MudTabs KeepPanelsAlive="true">
    <MudTabPanel Text="General">
        <!-- form fields tab 1 -->
    </MudTabPanel>
    <MudTabPanel Text="Permisos">
        <!-- form fields tab 2 -->
    </MudTabPanel>
</MudTabs>
```

**Coste**: render inicial de todos los panels (más lento al cargar la página la primera vez). Si el formulario es muy pesado, considera dividir en páginas separadas en lugar de tabs.

---

## 20. Memory leak con `ISnackbar` o `IDialogService` en componentes que se desmontan

**Síntoma**: tras navegar varias veces a una página, la consola del navegador muestra warnings de memoria. Snackbars o dialogs aparecen aunque hayas cerrado la página.

**Causa**: estás capturando referencias al componente actual en el callback de `Snackbar.Add` o `DialogService.ShowAsync`. Cuando el componente se desmonta, la referencia sigue viva en la cola del Snackbar/Dialog.

**Solución**: implementa `IDisposable` o `IAsyncDisposable` y limpia:

```csharp
@implements IDisposable

@code {
    private CancellationTokenSource _cts = new();

    private async Task DoSomethingAsync()
    {
        try
        {
            await SomeService.LongOperation(_cts.Token);
            Snackbar.Add("Done", Severity.Success);
        }
        catch (OperationCanceledException) { /* expected on dispose */ }
    }

    public void Dispose()
    {
        _cts.Cancel();
        _cts.Dispose();
    }
}
```

---

## 21. Versión 6.x → 9.x — APIs renombradas

Si encuentras código que asume parámetros antiguos (típicamente al actualizar de Aldeworks v1 con MudBlazor 6 al actual con 9):

| MudBlazor 6.x | MudBlazor 9.x | Nota |
|---|---|---|
| `Disabled="true"` (en `MudTextField` con label flotante) | `Disabled="true"` igual, pero `Variant` cambia el comportamiento del label | OK |
| `IsOpen` / `IsOpenChanged` (varios) | `Open` / `OpenChanged` | Renombrado a positivo |
| `DisableUnderLine` (en TextField) | `Underline` (negado) | Renombrado a positivo |
| `MudAutocomplete.Value` setter en setter | Solo `@bind-Value` o `SetValueAsync` | Lógica fuera de setters |

Si te topas con un parámetro que no existe en 9.x, busca en GitHub Releases del upstream el changelog de la versión donde se cambió.

---

## 22. Tests que pasan local y fallan en CI

**Síntoma**: `dotnet test` local pasa, en CI (GitHub Actions) falla con timeouts o asserts intermitentes.

**Causas comunes**:

1. **Race condition con render**: el assert se evalúa antes de que termine el render. Solución: `await comp.WaitForAssertion(() => ..., timeout: TimeSpan.FromSeconds(3));`.
2. **Caching de `Find()`**: el componente re-renderizó y la referencia es stale. Solución: re-querear (ver `testing_with_bunit.md` §4).
3. **Estado estático no restaurado**: otro test corrió antes y dejó `MudGlobal.Defaults` modificado. Solución: `[TearDown]` que restaure.
4. **`Task.Delay` con timing diferente**: CI es más lento. Solución: `FakeTimeProvider` en lugar de `Task.Delay`.

---

## 23. "El tema custom no se aplica al recargar la página"

**Síntoma**: el usuario configura un tema custom, navega a otra página, vuelve, y el tema vuelve al default.

**Causa**: el `MudTheme` y `_isDark` están en estado del componente `MainLayout`. En cada navegación se reinicializan.

**Solución**: persistir y cargar de un `ICascadingValueSource` o de un servicio Singleton (`IThemeService`) que mantenga el estado entre navegaciones:

```csharp
public class ThemeService
{
    public MudTheme CurrentTheme { get; private set; } = new();
    public bool IsDark { get; private set; }

    public event Action? OnChange;

    public async Task ApplyAsync(MudTheme theme, bool isDark)
    {
        CurrentTheme = theme;
        IsDark = isDark;
        await UserPreferencesService.SaveAsync(theme, isDark);
        OnChange?.Invoke();
    }
}
```

```razor
@inject ThemeService ThemeService

<MudThemeProvider Theme="@ThemeService.CurrentTheme" IsDarkMode="@ThemeService.IsDark" />
```

Aldeworks ya tiene este patrón en `AN.WebPlatform.Theming` — si encuentras un caso de tema que no persiste, revisa si la página está bypaseando el service.

---

## Cuándo NO es un pitfall — falsos positivos

A veces lo que parece un pitfall es comportamiento intencional:

- **`MudButton Disabled="true"` con `Href`** colapsa a `<button>`: intencional, un anchor disabled no es accesible.
- **`MudIconButton` con solo icono** no tiene tooltip por defecto: el dev tiene que añadir `aria-label` o `Title` (la accesibilidad es responsabilidad del consumer).
- **`MudSelect<T>` no busca por defecto**: para autocomplete usa `MudAutocomplete<T>`, no `MudSelect`.
- **`MudTextField` no se actualiza al teclear hasta perder foco**: ese es el comportamiento `Immediate="false"` por defecto. Si quieres reactividad, `Immediate="true"` + `DebounceInterval` para no spamear.

---

[← Volver al SKILL.md](../SKILL.md)

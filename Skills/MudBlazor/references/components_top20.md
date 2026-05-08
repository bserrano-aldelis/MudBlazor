# Componentes top 20 en Aldeworks — guía operativa

> Esta referencia se carga cuando el usuario pregunta por uno de los 20 componentes que ya usa Aldeworks. Para cada uno: API canónica, ejemplo correcto, gotchas, integración con `EditForm` / `DataAnnotations` / `FluentValidation`, accesibilidad y RTL.
>
> Fuente del código: `C:\dev\MudBlazor\src\MudBlazor\Components\<ComponentName>\` (cada componente vive en su propia carpeta con `.razor` + `.razor.cs`).

[← Volver al SKILL.md](../SKILL.md)

---

## 1. `MudTextField<T>` — 63 usos en Aldeworks

Campo de entrada genérico (texto, fecha, etc.) con label flotante Material.

### API esencial

| Parámetro | Tipo | Por defecto | Notas |
|---|---|---|---|
| `Value` / `@bind-Value` | `T` | `default` | Valor enlazado. |
| `Label` | `string?` | `null` | Etiqueta flotante. |
| `Variant` | `Variant` | `Text` | `Text` / `Filled` / `Outlined`. |
| `Margin` | `Margin` | `None` | `None` / `Dense` / `Normal`. |
| `Required` | `bool` | `false` | Marca visual + validación si hay `RequiredError`. |
| `RequiredError` | `string?` | `null` | Mensaje cuando `Required && string.IsNullOrEmpty(Value)`. |
| `Validation` | `object?` | `null` | `Func<T, IEnumerable<string>>`, `Func<T, Task<IEnumerable<string>>>`, regex, `ValidationAttribute[]` o `IValidator<T>` (FluentValidation). |
| `For` | `Expression<Func<T>>?` | `null` | Para `EditForm` + `DataAnnotations`: `For="@(() => model.Property)"`. |
| `Lines` | `int` | `1` | `> 1` lo convierte en multiline (`<textarea>`). |
| `Mask` | `IMask?` | `null` | Mascarillas (`PatternMask`, `RegexMask`, `MultiMask`, `BlockMask`). |
| `Adornment` | `Adornment` | `None` | `Start` / `End`. Combina con `AdornmentIcon` o `AdornmentText`. |
| `Immediate` | `bool` | `false` | `true` actualiza el binding en `oninput`; `false` solo en `onchange`. |
| `DebounceInterval` | `double` | `0` | Milisegundos de debounce. |

### Ejemplo canónico (formulario con `EditForm`)

```razor
<EditForm Model="@vm" OnValidSubmit="HandleSubmitAsync">
    <DataAnnotationsValidator />

    <MudTextField @bind-Value="vm.Name"
                  Label="Nombre"
                  Variant="Variant.Outlined"
                  Required="true"
                  RequiredError="El nombre es obligatorio"
                  For="@(() => vm.Name)" />

    <MudButton Variant="Variant.Filled"
               Color="Color.Primary"
               ButtonType="ButtonType.Submit"
               Disabled="@(!editContext.IsModified())">
        Guardar
    </MudButton>
</EditForm>
```

`For="@(() => vm.Name)"` engancha la validación de DataAnnotations del VM (decorado con `[Required]`, `[StringLength]`, etc.).

### Validación con FluentValidation

```razor
<MudTextField @bind-Value="vm.Email"
              Label="Email"
              Validation="@(new Func<string, IEnumerable<string>>(ValidateEmail))" />

@code {
    private IEnumerable<string> ValidateEmail(string email)
    {
        var validator = new InlineValidator<string>();
        validator.RuleFor(x => x).EmailAddress().WithMessage("Email no válido");
        var result = validator.Validate(email);
        return result.Errors.Select(e => e.ErrorMessage);
    }
}
```

### Pitfalls

- **`Immediate="true"` + `DebounceInterval="0"`** dispara render por cada tecla → costoso en formularios grandes. Pon `DebounceInterval="300"` para autocompletes / búsquedas reactivas.
- **`For` sin `EditForm`** no hace nada. Si no usas `EditForm`, valida con `Validation` directamente.
- **`Lines > 1`** cambia el HTML a `<textarea>`, lo que rompe `Mask` (que asume `<input>`).
- **Tipos no string**: `MudTextField<int>` requiere `Converter` o se romperá la entrada parcial ("1", "12", "123" intermedios). Para números prefiere `MudNumericField<T>`.

---

## 2. `MudButton` — 19 usos

### API esencial

| Parámetro | Tipo | Por defecto | Notas |
|---|---|---|---|
| `Variant` | `Variant` | `Text` | `Text` / `Filled` / `Outlined`. |
| `Color` | `Color` | `Default` | `Primary`, `Secondary`, `Tertiary`, `Info`, `Success`, `Warning`, `Error`, `Dark`, `Inherit`. |
| `Size` | `Size` | `Medium` | `Small` / `Medium` / `Large`. |
| `OnClick` | `EventCallback<MouseEventArgs>` | — | Handler. |
| `Disabled` | `bool` | `false` | Deshabilita. |
| `ButtonType` | `ButtonType` | `Button` | `Button` / `Submit` / `Reset`. Solo importa dentro de `<form>` / `<EditForm>`. |
| `StartIcon` / `EndIcon` | `string?` | `null` | Path SVG (`Icons.Material.Filled.Save`). |
| `IconColor` | `Color` | `Inherit` | Color del icono. |
| `Href` | `string?` | `null` | Si está set, renderiza `<a>` en lugar de `<button>`. |
| `Target` | `string?` | `null` | `_blank`, `_self`. Con `Href`. |
| `Rel` | `string?` | `null` | Override del `rel="noopener"` automático. |

### Ejemplo canónico

```razor
<MudButton Variant="Variant.Filled"
           Color="Color.Primary"
           StartIcon="@Icons.Material.Filled.Save"
           OnClick="SaveAsync">
    Guardar
</MudButton>
```

### Patrón "Save deshabilitado si no hay cambios" (regla del cliente)

```razor
<EditForm EditContext="@editContext" OnValidSubmit="HandleSubmitAsync">
    <MudButton ButtonType="ButtonType.Submit"
               Color="Color.Primary"
               Variant="Variant.Filled"
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
        await mediator.Send(new SaveCommand(vm));
        editContext.MarkAsUnmodified();   // tras éxito
    }
}
```

### Pitfalls

- **`Href` + `Disabled="true"`** colapsa a `<button>` (no `<a>`) intencionalmente — un anchor disabled no es accesible. Si quieres link "deshabilitado visual", duplica el comportamiento con un `<MudButton>` puro.
- **`OnClick` sync vs async**: si tu handler es async (`Task SaveAsync()`), el botón no se deshabilita solo durante la operación. Patrón: setea un `_isSaving` flag y átalo a `Disabled`.
- **Submit dentro de `EditForm`** dispara `OnValidSubmit` solo si la validación pasa. Si no, dispara `OnInvalidSubmit`. Si no manejas ninguno, el botón parece "no hacer nada".

---

## 3. `MudSelect<T>` y `MudSelectItem<T>` — 15 + 16 usos

Dropdown con búsqueda opcional y selección múltiple.

### API esencial (`MudSelect`)

| Parámetro | Tipo | Por defecto | Notas |
|---|---|---|---|
| `Value` / `@bind-Value` | `T` | `default` | Valor (single). |
| `SelectedValues` / `@bind-SelectedValues` | `IEnumerable<T>?` | `null` | Multi-select. |
| `MultiSelection` | `bool` | `false` | Activa multi. |
| `Label` | `string?` | `null` | Label flotante. |
| `Variant` | `Variant` | `Text` | Material variants. |
| `ToStringFunc` | `Func<T, string>?` | `null` | Render del item en el campo cerrado. |
| `SelectedValuesChanged` | `EventCallback<IEnumerable<T>>` | — | Solo multi. |
| `Dense` | `bool` | `false` | Items compactos. |
| `Clearable` | `bool` | `false` | Botón "X" para limpiar. |
| `For` | `Expression<Func<T>>?` | `null` | Validación DataAnnotations (single only). |

### Ejemplo single-select con objetos custom

```razor
<MudSelect T="UserDto"
           @bind-Value="vm.SelectedUser"
           Label="Usuario asignado"
           ToStringFunc="@(u => u?.FullName ?? string.Empty)"
           Variant="Variant.Outlined">
    @foreach (var user in users)
    {
        <MudSelectItem T="UserDto" Value="@user">
            @user.FullName  (@user.UserName)
        </MudSelectItem>
    }
</MudSelect>
```

### Ejemplo multi-select

```razor
<MudSelect T="int"
           Label="Roles"
           MultiSelection="true"
           @bind-SelectedValues="vm.SelectedRoleIds">
    @foreach (var role in availableRoles)
    {
        <MudSelectItem T="int" Value="@role.Id">@role.Name</MudSelectItem>
    }
</MudSelect>
```

### Pitfalls

- **Comparación de `T` complejos**: `MudSelect<UserDto>` compara con `Equals`. Si `UserDto` no es record y no implementa `Equals`/`GetHashCode`, el item nunca aparece "seleccionado" tras un round-trip. Solución: declararlo `record`, o sobrescribir `Equals`, o usar `T=int` con `ToStringFunc` resolviendo el id → nombre.
- **`MultiSelection="true"` + `@bind-Value`**: el binding es `@bind-SelectedValues`, no `@bind-Value`. Mezclarlos da estado inconsistente.
- **`MudSelectItem` fuera de `MudSelect`**: no tiene sentido, no renderiza nada útil.

Para autocomplete reactivo (lista grande, búsqueda async) usa `MudAutocomplete<T>`, no `MudSelect`.

---

## 4. `MudCheckBox<T>` — 15 usos

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Value` / `@bind-Value` | `T` | `bool`, `bool?`, o un enum. |
| `Label` | `string?` | Texto al lado. |
| `LabelPlacement` | `Placement` | `End` (default), `Start`. |
| `Color` | `Color` | Color del check activo. |
| `UncheckedColor` | `Color` | Color del check inactivo. |
| `Dense` | `bool` | Compacto. |
| `TriState` | `bool` | Permite `null` además de `true`/`false` (requiere `T = bool?`). |
| `For` | `Expression<Func<T>>?` | DataAnnotations. |

### Ejemplo

```razor
<MudCheckBox T="bool"
             @bind-Value="vm.IsActive"
             Label="Activo"
             Color="Color.Primary" />
```

### Tri-state

```razor
<MudCheckBox T="bool?" @bind-Value="vm.HasAccess" TriState="true" Label="Acceso" />
```

`null` representa "indeterminado" (visualmente un guion). Útil para filtros de listado donde "no me importa" es un estado válido.

---

## 5. `MudNumericField<T>` — 14 usos

Campo numérico con spin buttons y validación de rango.

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Value` / `@bind-Value` | `T` | `int`, `long`, `decimal`, `double`, `float`, etc. (numeric primitives). |
| `Min` / `Max` | `T` | Rango. Si el usuario escribe fuera, se clampa al perder foco. |
| `Step` | `T` | Incremento de los spin buttons. |
| `Format` | `string?` | Formato de display (`F2`, `N0`, etc.). |
| `Culture` | `CultureInfo?` | Cultura para parsing y display (importante para decimales con `,` vs `.`). |
| `HideSpinButtons` | `bool` | Oculta los spin. |
| `Immediate` | `bool` | Igual que `MudTextField`. |

### Ejemplo (margen %)

```razor
<MudNumericField T="decimal"
                 @bind-Value="vm.DefaultMarginDistribution"
                 Label="Margen distribución (%)"
                 Min="0m"
                 Max="99.99m"
                 Step="0.5m"
                 Format="F2"
                 Variant="Variant.Outlined"
                 Required="true"
                 RequiredError="Indica un margen entre 0 y 99,99 %" />
```

### Pitfalls

- **Cultura**: si tu app usa `es-ES`, el separador decimal es `,`. Si el usuario teclea `4.59` esperando 4.59 (con punto), el parsing falla en `es-ES`. Solución: setear `Culture="@CultureInfo.InvariantCulture"` para inputs numéricos críticos, o aceptar la cultura del usuario y educar.
- **`Min`/`Max` como `decimal`**: requieren sufijo `m` (`0m`, `99.99m`). Sin sufijo el compilador los infiere como `double` y peta.
- **Spin buttons** no se ocultan en mobile (es nativo del browser). Si quieres look limpio, `HideSpinButtons="true"`.

---

## 6. `MudSwitch<T>` — 13 usos

Toggle ON/OFF.

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Value` / `@bind-Value` | `T` (típicamente `bool`) | Estado. |
| `Label` | `string?` | Etiqueta. |
| `Color` / `UncheckedColor` | `Color` | Colores. |
| `Disabled` | `bool` | Deshabilitar. |
| `LabelPlacement` | `Placement` | Posición del label. |

### Ejemplo

```razor
<MudSwitch T="bool" @bind-Value="vm.MustChangePassword" Label="Forzar cambio de contraseña" Color="Color.Primary" />
```

Identidad visual con `MudCheckBox` — diferencia es semántica:

- **Switch**: cambio que tiene efecto inmediato (preferencia, modo).
- **Checkbox**: selección que se confirma al guardar formulario.

---

## 7. `MudDivider` — 13 usos

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Vertical` | `bool` | `false` por defecto (horizontal). |
| `FlexItem` | `bool` | Para usar dentro de `MudStack`/flex containers. |
| `Light` | `bool` | Versión más sutil. |
| `DividerType` | `DividerType` | `FullWidth` / `Inset` / `Middle`. |

### Ejemplo

```razor
<MudPaper>
    <MudText Typo="Typo.h6">Sección A</MudText>
    <!-- contenido -->
    <MudDivider Class="my-3" />
    <MudText Typo="Typo.h6">Sección B</MudText>
    <!-- contenido -->
</MudPaper>
```

---

## 8. `MudIconButton` — 9 usos

Botón solo con icono (acciones en filas, toolbar de tarjetas).

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Icon` | `string` | Path SVG (`Icons.Material.Filled.Edit`). |
| `Color` | `Color` | Color del icono. |
| `Size` | `Size` | Tamaño. |
| `OnClick` | `EventCallback<MouseEventArgs>` | Handler. |
| `Disabled` | `bool` | Deshabilitar. |
| `Edge` | `Edge` | `False` / `Start` / `End` — alinea con padding cero al borde de un container. |
| `Title` (`title` HTML) | `string?` | **Tooltip nativo**, accesibilidad básica. Ver pitfall. |

### Ejemplo

```razor
<MudIconButton Icon="@Icons.Material.Filled.Edit"
               Color="Color.Primary"
               OnClick="@(() => OpenEditDialog(item.Id))"
               Title="Editar" />
```

### Accesibilidad — pitfall importante

`MudIconButton` por sí solo **no es accesible para screen readers** porque solo lleva icono visual. **Siempre** añade `aria-label` o `Title`:

```razor
<MudIconButton Icon="..." aria-label="Eliminar configuración" />
```

Si quieres tooltip enriquecido (mejor que `title` nativo), envuelve en `MudTooltip`:

```razor
<MudTooltip Text="Eliminar configuración">
    <MudIconButton Icon="@Icons.Material.Filled.Delete" aria-label="Eliminar configuración" />
</MudTooltip>
```

---

## 9. `MudProgressCircular` — 7 usos

Loader inline.

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Color` | `Color` | Color del trazo. |
| `Size` | `Size` | `Small`/`Medium`/`Large`. |
| `Indeterminate` | `bool` | `true` = girando indefinidamente. `false` = barra de progreso (necesita `Value`/`Min`/`Max`). |
| `Value` / `Min` / `Max` | `double` | Solo en modo determinate. |

### Ejemplo (botón con loader)

```razor
<MudButton OnClick="LoadAsync" Disabled="@_isLoading">
    @if (_isLoading)
    {
        <MudProgressCircular Color="Color.Default" Size="Size.Small" Indeterminate="true" Class="me-2" />
    }
    Cargar
</MudButton>
```

---

## 10. `MudText` — 5 usos

Tipografía Material.

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Typo` | `Typo` | `h1`...`h6`, `subtitle1`/`2`, `body1`/`2`, `button`, `caption`, `overline`, `inherit`. |
| `Align` | `Align` | `Inherit`/`Left`/`Center`/`Right`/`Justify`/`Start`/`End`. |
| `Color` | `Color` | Color del texto. |
| `GutterBottom` | `bool` | Margin-bottom estándar. |
| `Inline` | `bool` | `<span>` en lugar de `<p>`. |

### Ejemplo

```razor
<MudText Typo="Typo.h4" GutterBottom="true">Configuración de tarifas</MudText>
<MudText Typo="Typo.body1" Color="Color.Secondary">
    Define los márgenes default por producto.
</MudText>
```

Para titulares importantes, prefiere `<h1>`/`<h2>` semánticos directamente cuando puedas; usa `MudText` cuando necesitas la integración con el theme (Typography del MudTheme).

---

## 11. `MudTabs` y `MudTabPanel` — 1 + 3 usos

Pestañas.

### Ejemplo canónico

```razor
<MudTabs Outlined="true" Position="Position.Top" Rounded="true" Border="true" ApplyEffectsToContainer="true">
    <MudTabPanel Text="General">
        <!-- form fields -->
    </MudTabPanel>
    <MudTabPanel Text="Permisos">
        <!-- roles -->
    </MudTabPanel>
    <MudTabPanel Text="Auditoría" BadgeData="@auditCount" BadgeColor="Color.Info">
        <!-- audit log -->
    </MudTabPanel>
</MudTabs>
```

### Pitfalls

- Por defecto **renderiza solo el panel activo** (`KeepPanelsAlive="false"`). Si tienes formularios con state que se pierden al cambiar pestaña, pon `KeepPanelsAlive="true"`. Coste: render inicial de todos los panels (puede ser caro si son pesados).
- **`Text="..."` no es un `RenderFragment`** (es string). Para iconos en la pestaña usa `IconColor`/`Icon`/`BadgeData`. Para títulos custom (con render fragments), usa `<TabPanelTemplate>...</TabPanelTemplate>`.

---

## 12. `MudAvatar` — 3 usos

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Size` | `Size` | `Small`/`Medium`/`Large`/`Custom`. |
| `Color` | `Color` | Background. |
| `Variant` | `Variant` | `Filled`/`Outlined`/`Text`. |
| `Image` | `string?` | URL de imagen. |
| `Square` | `bool` | Forma. |
| `ChildContent` | `RenderFragment?` | Iniciales o icono cuando no hay imagen. |

### Ejemplo

```razor
<MudAvatar Image="@user.PhotoUrl" Size="Size.Medium">
    @user.Initials
</MudAvatar>
```

Patrón Aldeworks: ver `C:\dev\AN.Aldeworks\src\platform\AN.WebPlatform.Identity.UI\Users\EditUser.razor` (computed `Initials` property).

---

## 13. `MudThemeProvider` — 2 usos (uno por host)

Cascading provider del tema. Va **una sola vez** en el árbol, normalmente en `MainLayout.razor` o `App.razor`. Si lo pones más de una vez, cada provider gana en su sub-árbol y el resultado es inconsistente.

### Ejemplo

```razor
<MudThemeProvider Theme="@_theme" IsDarkMode="@_isDark" />
<MudDialogProvider />
<MudSnackbarProvider />
<MudPopoverProvider />

@Body

@code {
    private MudTheme _theme = new();
    private bool _isDark;

    protected override async Task OnInitializedAsync()
    {
        // Cargar preferencia del usuario (Aldeworks: Users.ThemePreset / ThemeMode)
        var prefs = await ThemeService.GetUserThemeAsync();
        _theme = prefs.Theme;
        _isDark = prefs.Mode == ThemeMode.Dark;
    }
}
```

Detalle de paletas/typography custom → `theming_guide.md`.

---

## 14. `MudPaper` — 2 usos

Wrapper visual con shadow + radius (la "tarjeta material").

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Elevation` | `int` | 0-25 (sombra). |
| `Outlined` | `bool` | Sin sombra, borde sutil. |
| `Square` | `bool` | Sin border-radius. |
| `Width` / `Height` | `string?` | CSS. |

### Ejemplo

```razor
<MudPaper Elevation="2" Class="pa-4">
    <MudText Typo="Typo.h6">Resumen</MudText>
    <!-- contenido -->
</MudPaper>
```

`pa-4` es la utility de padding 16px. MudBlazor incluye un sistema de utilities tipo Tailwind (ver `Class="ma-2 pa-4 d-flex align-center"`).

---

## 15. `MudMenu` y `MudMenuItem` — 2 usos

Menú contextual / dropdown de acciones.

### Ejemplo

```razor
<MudMenu Icon="@Icons.Material.Filled.MoreVert" Color="Color.Inherit" Dense="true">
    <MudMenuItem OnClick="@(() => Edit(item.Id))" Icon="@Icons.Material.Filled.Edit">Editar</MudMenuItem>
    <MudMenuItem OnClick="@(() => Deactivate(item.Id))" Icon="@Icons.Material.Filled.Block">Desactivar</MudMenuItem>
    <MudDivider />
    <MudMenuItem OnClick="@(() => ViewAudit(item.Id))" Icon="@Icons.Material.Filled.History">Ver auditoría</MudMenuItem>
</MudMenu>
```

### Pitfalls

- Sin `aria-label` en el `MudMenu` con solo `Icon`: añade `AriaLabel="Acciones del registro"`.
- Si el menú abre por click pero quieres también por hover: `ActivationEvent="MouseEvent.MouseOver"`.

---

## 16. `MudColorPicker` — 2 usos

Picker de color para custom theming.

### Ejemplo

```razor
<MudColorPicker @bind-Value="vm.PrimaryColor"
                Label="Color primario"
                Variant="Variant.Outlined"
                ColorPickerView="ColorPickerView.Spectrum"
                ShowPreview="true" />
```

Aldeworks lo usa en pantallas de Theming (`AN.WebPlatform.Theming/Admin`). Cuando el usuario cambia el color, hay que regenerar el `MudTheme` en runtime y persistirlo.

---

## 17. `MudAlert` — 2 usos

Banner de mensaje.

### API

| Parámetro | Tipo | Notas |
|---|---|---|
| `Severity` | `Severity` | `Normal` / `Info` / `Success` / `Warning` / `Error`. |
| `Variant` | `Variant` | Filled/Outlined/Text. |
| `ShowCloseIcon` | `bool` | Botón "X". |
| `CloseIconClicked` | `EventCallback<MouseEventArgs>` | Handler del cierre. |

### Ejemplo

```razor
@if (!string.IsNullOrEmpty(_errorMessage))
{
    <MudAlert Severity="Severity.Error"
              Variant="Variant.Filled"
              ShowCloseIcon="true"
              CloseIconClicked="@(() => _errorMessage = null)">
        @_errorMessage
    </MudAlert>
}
```

Para mensajes transitorios (Save success, error en PUT), prefiere **`ISnackbar`** (toasts) en lugar de `MudAlert` inline.

---

## 18. `MudSpacer` — 1 uso

Empujador en flex containers (`MudStack`, `MudAppBar`, `MudToolBar`).

```razor
<MudAppBar>
    <MudText Typo="Typo.h6">Mi App</MudText>
    <MudSpacer />
    <MudIconButton Icon="@Icons.Material.Filled.AccountCircle" />
</MudAppBar>
```

Equivalente a `<div style="flex-grow: 1;"></div>`. Sin él, los elementos colapsan al inicio.

---

## Componentes que NO están en el top 20 pero llegarán pronto a Aldeworks

Estos van a aparecer en los próximos PRs (Tarificador, configuraciones, exports). Conviene tenerlos calientes:

- **`MudDataGrid<T>`** — listado avanzado con filtering, sorting, grouping, virtualization. Sustituirá al patrón actual del DefaultViewer cuando se implemente la entrada R1 del roadmap (perfiles personalizables del DefaultViewer).
- **`MudDialog`** + **`IDialogService.ShowAsync<T>(...)`** — diálogos modales para alta/edición rápida sin navegar a otra página.
- **`MudSnackbar`** + **`ISnackbar.Add(...)`** — toasts de feedback (Save success, errores).
- **`MudAutocomplete<T>`** — autocomplete reactivo. Imprescindible para los pickers del Tarificador (productos legacy, clientes legacy).
- **`MudFileUpload<T>`** — para futuras subidas de fotos / documentos.
- **`MudStepper`** — wizard multi-paso (cotización compleja).
- **`MudChart`** — gráficas (cuadro de mando V2).

Cuando alguno se incorpore, se actualizará esta referencia con su API canónica.

---

## Patrón general — checklist al usar un componente Mud

1. ¿Hay un `[Parameter]` que ya cubre lo que necesitas? **No reinventes**: revisa el `.razor.cs` del componente.
2. ¿Has añadido `Label` (formularios) o `aria-label` (botones icon-only) para accesibilidad?
3. Si el formulario es de edición, ¿has atado `Disabled` del Save a `EditContext.IsModified()`?
4. Si tu componente padre va a SSR pre-render, ¿has metido `if (!RendererInfo.IsInteractive) return;` en el inicio de `OnInitializedAsync`?
5. Si declaras `Class="..."`, ¿usas las utilities (`ma-2`, `pa-4`, `d-flex`) en lugar de styles inline?
6. Si el render depende de breakpoint, ¿has usado `MudHidden` o `IBrowserViewportService` en lugar de `@media` query manual?
7. Si tu componente es nuevo en el fork, ¿estás siguiendo `ParameterState<T>` (ver `parameter_state.md`)?

---

[← Volver al SKILL.md](../SKILL.md)

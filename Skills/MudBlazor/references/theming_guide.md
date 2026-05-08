# Theming guide — paletas, dark/light, tipografía, custom CSS

> Esta referencia se carga cuando el usuario pregunta por theming, paletas, switch dark/light, tipografía, custom CSS, design tokens o `MudThemeProvider`. Cubre cómo personalizar MudBlazor en Aldeworks (que persiste preferencias en `Users.ThemePreset` y `Users.ThemeMode`) y cómo extender el theme dentro del fork.

[← Volver al SKILL.md](../SKILL.md)

Fuentes:
- `C:\dev\MudBlazor\src\MudBlazor\Themes\MudTheme.cs`
- `C:\dev\MudBlazor\src\MudBlazor\Themes\Models\` (todos los modelos)
- `C:\dev\MudBlazor\src\MudBlazor\Styles\` (SCSS)
- `C:\dev\AN.AldeWorks\src\platform\AN.WebPlatform.Theming\` (consumer-side en Aldeworks)

---

## 1. Estructura del `MudTheme`

```csharp
public class MudTheme
{
    public Palette PaletteLight { get; set; }       // colores en modo claro
    public Palette PaletteDark { get; set; }        // colores en modo oscuro
    public Shadow Shadows { get; set; }             // 25 niveles de sombra
    public Typography Typography { get; set; }      // fuentes, tamaños, weights
    public LayoutProperties LayoutProperties { get; set; }   // border radius, drawer width, app bar height
    public ZIndex ZIndex { get; set; }              // capas (drawer, appbar, dialog, popover, snackbar, tooltip)
    public PseudoCss PseudoCss { get; set; }        // selectores `:root`, scrollbar, etc.
}
```

Cada propiedad raíz tiene su propia clase con docenas de propiedades. Se serializa a CSS variables (`var(--mud-palette-primary)`) y se aplica via `MudThemeProvider`.

---

## 2. Aplicación canónica (provider único en `MainLayout`)

```razor
@* MainLayout.razor *@
@inherits LayoutComponentBase

<MudThemeProvider @ref="_themeProvider"
                  Theme="@_theme"
                  IsDarkMode="@_isDark"
                  IsDarkModeChanged="OnDarkModeChanged" />
<MudPopoverProvider />
<MudDialogProvider />
<MudSnackbarProvider />

<MudLayout>
    <MudAppBar Elevation="1">
        <MudIconButton Icon="@Icons.Material.Filled.Menu"
                       Color="Color.Inherit"
                       OnClick="@ToggleDrawer" />
        <MudText Typo="Typo.h6">Aldeworks</MudText>
        <MudSpacer />
        <MudIconButton Icon="@(_isDark ? Icons.Material.Filled.LightMode : Icons.Material.Filled.DarkMode)"
                       Color="Color.Inherit"
                       OnClick="@ToggleDarkMode"
                       aria-label="Cambiar tema claro/oscuro" />
    </MudAppBar>
    <MudDrawer @bind-Open="_drawerOpen" Elevation="2">
        <NavMenu />
    </MudDrawer>
    <MudMainContent>
        @Body
    </MudMainContent>
</MudLayout>

@code {
    private MudThemeProvider _themeProvider = default!;
    private MudTheme _theme = new();
    private bool _isDark;
    private bool _drawerOpen = true;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Cargar preferencia del usuario desde BD (Aldeworks: Users.ThemeMode + Users.ThemePreset)
            var prefs = await ThemeService.GetCurrentUserPreferencesAsync();
            _theme = ThemeFactory.Build(prefs.PresetName);
            _isDark = prefs.Mode == ThemeMode.Dark
                ?? await _themeProvider.GetSystemPreference();   // null = "auto" → seguir SO
            StateHasChanged();
        }
    }

    private async Task ToggleDarkMode()
    {
        _isDark = !_isDark;
        await ThemeService.SaveUserPreferenceAsync(_isDark ? ThemeMode.Dark : ThemeMode.Light);
    }

    private Task OnDarkModeChanged(bool value) => Task.CompletedTask;   // sync con preference si sistema cambia
}
```

**Reglas**:

- **Un solo `MudThemeProvider`** en toda la app, en el layout más alto. Si lo metes en sub-componentes, sus paletas no aplicarán globalmente.
- **`MudPopoverProvider`, `MudDialogProvider`, `MudSnackbarProvider`** van junto al `MudThemeProvider` — son providers root también.
- **Detección "auto"** (seguir SO): `_themeProvider.GetSystemPreference()` consulta `prefers-color-scheme` del navegador.

---

## 3. Paletas — Light y Dark

`Palette` (clase base de `PaletteLight` y `PaletteDark`) tiene ~50 propiedades. Las clave:

### Colores principales

| Propiedad | Uso |
|---|---|
| `Primary` / `PrimaryContrastText` / `PrimaryDarken` / `PrimaryLighten` | Color de marca dominante. CTAs, links principales. |
| `Secondary` / + variantes | Color complementario. |
| `Tertiary` / + variantes | Tercer nivel para casos raros. |
| `Info` / `Success` / `Warning` / `Error` / `Dark` | Semánticos. Severidades de Alerts/Snackbars. |

### Backgrounds y surfaces

| Propiedad | Uso |
|---|---|
| `Background` | Body de la app (debajo del Layout). |
| `BackgroundGray` | Variante secundaria de fondo. |
| `Surface` | `MudPaper`, `MudCard`, dialogs. |
| `DrawerBackground` / `AppbarBackground` / `TableLines` | Wrappers de layout específicos. |

### Texto

| Propiedad | Uso |
|---|---|
| `TextPrimary` | Body text principal. |
| `TextSecondary` | Body text secundario / captions. |
| `TextDisabled` | Texto deshabilitado. |
| `ActionDefault` / `ActionDisabled` / `ActionDisabledBackground` | Estados de botones/iconos. |

### Bordes y sombras

| Propiedad | Uso |
|---|---|
| `LinesDefault` | Bordes de inputs, cards (default). |
| `LinesInputs` | Bordes específicos de inputs. |
| `Divider` | Color de `MudDivider`. |
| `OverlaysDefault` | Overlay de dialogs/drawers. |

### Custom palette canónica de Aldelis (ejemplo)

```csharp
public static class AldelisTheme
{
    public static MudTheme Build()
    {
        return new MudTheme
        {
            PaletteLight = new PaletteLight
            {
                Primary = "#1565C0",            // Azul Aldelis
                PrimaryContrastText = "#FFFFFF",
                Secondary = "#FFA000",          // Naranja acento
                Tertiary = "#558B2F",
                Background = "#F5F7FA",
                Surface = "#FFFFFF",
                AppbarBackground = "#FFFFFF",
                AppbarText = "#1565C0",
                DrawerBackground = "#FAFAFA",
                TextPrimary = "#1A1A1A",
                TextSecondary = "#5F6368"
            },
            PaletteDark = new PaletteDark
            {
                Primary = "#42A5F5",
                Secondary = "#FFB74D",
                Tertiary = "#9CCC65",
                Background = "#121212",
                Surface = "#1E1E1E",
                AppbarBackground = "#1E1E1E",
                AppbarText = "#FFFFFF",
                DrawerBackground = "#1A1A1A",
                TextPrimary = "#E0E0E0",
                TextSecondary = "#9E9E9E"
            },
            Typography = new Typography
            {
                Default = new DefaultTypography
                {
                    FontFamily = new[] { "Roboto", "Helvetica", "Arial", "sans-serif" },
                    FontSize = ".875rem",
                    FontWeight = "400",
                    LineHeight = "1.43",
                    LetterSpacing = ".01071em"
                },
                H1 = new H1Typography { FontSize = "2.5rem", FontWeight = "300", LineHeight = "1.167" },
                H6 = new H6Typography { FontSize = "1.125rem", FontWeight = "500", LineHeight = "1.6" }
                // ...el resto siguiendo Material spec
            },
            LayoutProperties = new LayoutProperties
            {
                DefaultBorderRadius = "6px",
                DrawerWidthLeft = "260px",
                AppbarHeight = "64px"
            },
            ZIndex = new ZIndex
            {
                Drawer = 1100,
                AppBar = 1200,
                Dialog = 1300,
                Popover = 1400,
                Snackbar = 1500,
                Tooltip = 1600
            }
        };
    }
}
```

---

## 4. Tipografía

```csharp
public class Typography
{
    public DefaultTypography Default { get; set; }
    public H1Typography H1 { get; set; }
    public H2Typography H2 { get; set; }
    public H3Typography H3 { get; set; }
    public H4Typography H4 { get; set; }
    public H5Typography H5 { get; set; }
    public H6Typography H6 { get; set; }
    public Subtitle1Typography Subtitle1 { get; set; }
    public Subtitle2Typography Subtitle2 { get; set; }
    public Body1Typography Body1 { get; set; }
    public Body2Typography Body2 { get; set; }
    public ButtonTypography Button { get; set; }
    public CaptionTypography Caption { get; set; }
    public OverlineTypography Overline { get; set; }
}
```

Cada `*Typography` tiene: `FontFamily`, `FontSize`, `FontWeight`, `LineHeight`, `LetterSpacing`, `TextTransform`.

**Cómo se consume**: `<MudText Typo="Typo.h1">...` lee `Typography.H1` del theme.

**Custom font**: añade el `<link>` o `@font-face` en `App.razor` o `index.html`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;700&display=swap">
```

Y luego en el theme:

```csharp
Typography = new Typography
{
    Default = new DefaultTypography
    {
        FontFamily = new[] { "Inter", "system-ui", "sans-serif" }
    }
}
```

---

## 5. CSS variables generadas

`MudThemeProvider` traduce el `MudTheme` a CSS variables en `:root`:

```css
:root {
    --mud-palette-primary: #1565C0;
    --mud-palette-primary-rgb: 21, 101, 192;
    --mud-palette-primary-text: #FFFFFF;
    --mud-palette-primary-darken: rgb(...);
    --mud-palette-primary-lighten: rgb(...);
    --mud-palette-secondary: #FFA000;
    /* ...todas las del Palette */

    --mud-typography-default-family: 'Inter', system-ui, sans-serif;
    --mud-typography-default-size: .875rem;
    /* ... */

    --mud-default-borderradius: 6px;
    --mud-drawer-width-left: 260px;
    --mud-appbar-height: 64px;

    --mud-zindex-drawer: 1100;
    --mud-zindex-appbar: 1200;
    /* ... */
}
```

Modo dark añade variantes que sobreescriben en `.mud-theme-dark`:

```css
.mud-theme-dark {
    --mud-palette-primary: #42A5F5;
    /* ... */
}
```

**Tu CSS custom** debe usar estas variables, **no hardcodear colores**:

```css
/* ✅ Bien */
.aldeworks-card-highlight {
    background: var(--mud-palette-primary);
    color: var(--mud-palette-primary-text);
    border-radius: var(--mud-default-borderradius);
}

/* ❌ Mal — no respeta dark mode ni custom theme */
.aldeworks-card-highlight {
    background: #1565C0;
    color: white;
    border-radius: 6px;
}
```

---

## 6. Dark mode — patrones avanzados

### 6.1. Detección automática (seguir SO)

```razor
<MudThemeProvider @ref="_themeProvider" @bind-IsDarkMode="_isDark" />

@code {
    private MudThemeProvider _themeProvider = default!;
    private bool _isDark;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            _isDark = await _themeProvider.GetSystemPreference();
            StateHasChanged();
        }
    }
}
```

### 6.2. Reaccionar a cambios del SO en runtime

```razor
@code {
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await _themeProvider.WatchSystemPreference(OnSystemPreferenceChanged);
        }
    }

    private async Task OnSystemPreferenceChanged(bool isDark)
    {
        if (UserPreferenceMode == ThemeMode.Auto)
        {
            _isDark = isDark;
            StateHasChanged();
        }
    }
}
```

### 6.3. Tres modos: Light / Dark / Auto

Aldeworks ya tiene este patrón. La columna `Users.ThemeMode` permite `'Light'`, `'Dark'`, `'Auto'`. En `Auto`, `_isDark` se atan a `prefers-color-scheme`.

---

## 7. Persistencia en cookie + BD (patrón Aldeworks)

`AN.WebPlatform.Identity` ya tiene un `ThemeCookieSeederMiddleware`. Cuando el usuario hace login:

1. Lee `Users.ThemePreset` y `Users.ThemeMode` de BD.
2. Setea cookies (`aw.theme.preset`, `aw.theme.mode`) que el front lee al inicializar.
3. El front aplica el tema antes del primer render → no hay flash de tema incorrecto.

Cuando el usuario cambia tema en runtime:

1. Llama a `ThemeService.SaveUserPreferenceAsync(...)`.
2. El service hace UPDATE en `Users.*` y refresca la cookie.
3. La página se re-renderiza con el nuevo `_theme`.

Si extiendes el modelo de Theming en el fork (paleta nueva, propiedad nueva en `MudTheme`), añade el campo en `Users.ThemePresetExtras` (o equivalente JSON) en `_Database Scripts/.../10 - Schema/`.

---

## 8. `LayoutProperties`

```csharp
public class LayoutProperties
{
    public string DefaultBorderRadius { get; set; } = "4px";
    public string DrawerWidthLeft { get; set; } = "240px";
    public string DrawerWidthRight { get; set; } = "240px";
    public string DrawerHeightTop { get; set; } = "64px";
    public string DrawerHeightBottom { get; set; } = "64px";
    public string AppbarHeight { get; set; } = "64px";
    public string DrawerMiniWidthLeft { get; set; } = "56px";
    public string DrawerMiniWidthRight { get; set; } = "56px";
}
```

Cambia drawer width o app bar height aquí. **NO** los hardcodes en CSS.

---

## 9. `ZIndex` — capas

```csharp
public class ZIndex
{
    public int Drawer { get; set; } = 1100;
    public int AppBar { get; set; } = 1200;
    public int Dialog { get; set; } = 1300;
    public int Popover { get; set; } = 1400;
    public int Snackbar { get; set; } = 1500;
    public int Tooltip { get; set; } = 1600;
}
```

**Si tienes un componente custom que se solapa con MudBlazor** (un overlay propio, un menú custom), úsalos:

```css
.aldeworks-custom-overlay {
    z-index: var(--mud-zindex-dialog);
}
```

**No subas a 9999** "por si acaso" — rompes el orden con popovers, snackbars y tooltips.

---

## 10. `PseudoCss` — escotillas para casos raros

```csharp
public class PseudoCss
{
    public string Scrollbar { get; set; } = "...";   // CSS custom para ::-webkit-scrollbar
}
```

Permite inyectar CSS arbitrario en `:root` desde el theme. **Úsalo con parsimonia** — la mayoría de necesidades se cubren con CSS variables.

---

## 11. Custom CSS de la app (Aldeworks)

CSS-in-app fuera del theme va en `wwwroot/css/site.css` (o equivalente). **Reglas**:

1. Usa variables del theme (`var(--mud-palette-primary)`, `var(--mud-default-borderradius)`).
2. Si necesitas overrides puntuales de un componente Mud, usa selectores `.mud-button.aldeworks-special` (clase custom + `.mud-button` para especificidad). **Nunca** `!important`.
3. Si tu override aplica a muchos sitios, considera proponerlo como `Variant` nuevo en el fork (más limpio, type-safe, documentado).

---

## 12. Roadmap del cliente — multi-idioma + perfiles

Dos entradas relevantes para theming en el roadmap del cliente (ver `Docs/Tarificador/00 - README.md` cuando se cierre):

- **R1**: Perfiles personalizables del DefaultViewer por usuario (no es theming puro pero comparte el patrón de "preferencia persistida → cookie → CSS dinámico").
- **R2**: Multi-idioma con selector por usuario. Implica una columna `Users.LanguagePreference` + middleware de cultura. **No** afecta directamente al theming, pero la persistencia de preferencias compartirá infraestructura.

---

## 13. Theming en componentes nuevos del fork

Si añades un componente al fork que necesita colorear estado:

```csharp
// MudAldeworksXxx.razor.cs
protected string Classname =>
    new CssBuilder("aldeworks-xxx")
        .AddClass($"aldeworks-xxx-{Color.ToDescriptionString()}", Color != Color.Default)
        .AddClass("aldeworks-xxx-outlined", Outlined)
        .AddClass(Class)
        .Build();
```

Y en el SCSS (`src/MudBlazor/Styles/components/_aldeworks-xxx.scss`):

```scss
.aldeworks-xxx {
    border-radius: var(--mud-default-borderradius);
    background: var(--mud-palette-surface);
    color: var(--mud-palette-text-primary);

    @each $color in $mud-palette-colors {
        &-#{$color} {
            background: var(--mud-palette-#{$color});
            color: var(--mud-palette-#{$color}-text);
        }
    }

    &-outlined {
        background: transparent;
        border: 1px solid var(--mud-palette-lines-default);
    }
}
```

Importa el partial en `src/MudBlazor/Styles/MudBlazor.scss`.

---

[← Volver al SKILL.md](../SKILL.md)

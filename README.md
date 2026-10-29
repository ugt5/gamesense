# Gamesense UI Library

The Gamesense UI Library is an open-source, pixel-accurate recreation of the iconic Gamesense (skeet.cc) cheat interface built natively for Luau and the Roblox engine. Designed with precision typography, dark theme gradients, compact modular geometry, and an event-driven architecture, it provides an uncompromising user experience for script creators and tool developers.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Hierarchy](#architecture-and-hierarchy)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [Library Core](#library-core)
  - [Window](#window)
  - [Tabs](#tabs)
  - [SubSections](#subsections)
  - [Sections](#sections)
  - [Controls](#controls)
    - [Toggle](#toggle)
    - [Slider](#slider)
    - [Dropdown](#dropdown)
    - [MultiBox](#multibox)
    - [ColorPicker](#colorpicker)
    - [Keybind](#keybind)
    - [Button](#button)
    - [TextBox](#textbox)
    - [Label](#label)
    - [List](#list)
- [State Management and Flags](#state-management-and-flags)
- [Configuration System (JSON)](#configuration-system-json)
- [System Utilities](#system-utilities)
  - [Notifications](#notifications)
  - [Watermark](#watermark)
  - [Connection Management](#connection-management)
  - [Player Viewport Utilities](#player-viewport-utilities)
- [Theming and Color System](#theming-and-color-system)
- [Lifecycle and Resource Cleanup](#lifecycle-and-resource-cleanup)
- [Complete Working Example](#complete-working-example)
- [License](#license)

---

## Overview

Gamesense UI Library delivers the authentic desktop cheat aesthetic to Roblox, prioritizing:

- **Visual Accuracy**: Pixel-perfect replication of Gamesense borders, gradient headers, checkable toggles, drag sliders, and dual-column section layouts.
- **Performance**: Event-driven rendering and minimal tween overhead. Inactive frames, popups, and dropdown menus detach or clamp visibility during idle states.
- **Robust State Engine**: Centralized flag repository (`Library.Flags`) enabling instantaneous read, write, and bidirectional data binding across controls.
- **Built-in Config Serialization**: JSON configuration management with custom HSV color serialization, allowing one-line export and import of user profiles.
- **Graceful Lifecycle Management**: Complete connection tracking and instance pooling, enabling zero-leak unloads via `Library:Unload()`.

---

## Architecture and Hierarchy

The library organizes visual and logical elements in a strict hierarchy:

```text
Library (Singleton / Root Controller)
  |
  +-- ScreenGui (Protected CoreGui / gethui container)
  |     +-- Watermark HUD (Top-right metric bar)
  |     +-- Notification Container (Top-left & Middle stacks)
  |     +-- Standalone Popups (Active ColorPicker frames, Dropdown menus, Keybind selectors)
  |
  +-- Window (Main draggable / resizable shell)
        +-- Header Bar (Title, accent divider, close keybind handler)
        +-- Tab Navigation Bar (Left-side icon strip)
        +-- Content Container
              +-- Tab (Active page controller)
                    |
                    +-- [Optional] SubSection (Nested secondary horizontal navigation)
                    |     +-- Category Tabs (ImageButton strip)
                    |     +-- Sub-page columns
                    |
                    +-- [Optional] ImageDropdown (Weapon / profile icon selector)
                    |
                    +-- Left Column / Right Column
                          +-- Section (Framed groupbox with title bar and border)
                                +-- Controls:
                                      - Toggle (with optional inline ColorPicker / Keybind)
                                      - Slider
                                      - Dropdown
                                      - MultiBox
                                      - Button
                                      - TextBox
                                      - Label (with optional inline ColorPicker / Keybind)
                                      - List
```

### Visual Structure

- **Main Window**: Bound to screen center on initialization. Supports user repositioning via title bar dragging and bidirectional viewport clamping.
- **Columns**: Sections are arranged along two parallel vertical tracks: `Left` and `Right`.
- **Dynamic Sizing**: Sections can maintain fixed pixel heights (`Size = UDim2.new(1, 0, 0, height)`) or expand dynamically to consume available vertical space (`Fill = true`).
- **Attached Modifiers**: Toggles and Labels support direct attachment of secondary controls (`ColorPicker` and `Keybind`) positioned horizontally within the same row item.

---

## Quick Start

### 1. Bootstrapping the Library

```lua
local rawLibrary = game:HttpGet("https://raw.githubusercontent.com/ugt5/gamesense/refs/heads/main/gamesense.lua")
local chunk = loadstring(rawLibrary, "@Gamesense")
local Library = chunk() or getgenv().Library
```

### 2. Creating a Window and Tab

```lua
local Window = Library:Window({
    Name = "gamesense",
    Size = UDim2.new(0, 700, 0, 612),
    CloseBind = Enum.KeyCode.Insert,
})

local CombatTab = Window:CreateTab({
    Icon = "rbxassetid://15453335745"
})
```

### 3. Adding Sections and Controls

```lua
local MainSection = CombatTab:Section({
    Name = "Aimbot Settings",
    Side = "Left",
    Fill = true,
})

MainSection:Toggle({
    Name = "Enabled",
    Default = false,
    Flag = "AimbotEnabled",
    Callback = function(state)
        print("Aimbot state:", state)
    end,
})
```

### 4. Initializing the Library

Always invoke `Library:Init()` after defining all tabs, sections, and controls. This initializes viewport listeners, displays the watermark, and establishes master input processing:

```lua
Library:Init()
```

---

## API Reference

### Library Core

The root singleton table managing global runtime state, connections, and system subsystems.

#### Properties

| Field | Type | Description |
| :--- | :--- | :--- |
| `Library.Flags` | `table` | Key-value table mapping flag names to their respective control instances. |
| `Library.Connections` | `table` | Array of active `RBXScriptConnection` references tracked for cleanup. |
| `Library.Objects` | `table` | Registry of instantiated visual frames and UI components. |
| `Library.Theme.Default` | `table` | Table containing active palette values (`Accent`, `SecondAccent`, `TextColor`, `Risky`). |
| `Library.UI.CloseBind` | `Enum.KeyCode` | Active keybind used to toggle window visibility. Defaults to `Enum.KeyCode.Insert`. |
| `Library.UI.TweenSpeed` | `number` | Duration in seconds used for GUI animations and fade transitions (default: `0.15`). |

#### Methods

##### `Library:Window(Options: table) -> Window`
Instantiates the primary window shell. Must be invoked before any tabs or sections can be registered.

##### `Library:Init() -> nil`
Finalizes layout calculations, spawns the watermark, and registers viewport recalculation connections. Call once after all interface elements have been defined.

##### `Library:Unload() -> nil`
Gracefully cleans up the entire library session:
1. Resets the active camera subject back to the local character Humanoid.
2. Disconnects all tracked RBXScriptConnections in `Library.Connections`.
3. Destroys all visual instances in `Library.Objects`.
4. Destroys the root `ScreenGui`.

##### `Library:Disable() -> nil`
Iterates through all registered controls in `Library.Flags` and calls `:Set(false)` on all toggleable features.

##### `Library:Notify(Options: table) -> Notification`
Pushes a toast notification banner to the top-left or middle screen region.

##### `Library:CreateWatermark() -> nil`
Constructs the HUD watermark bar. Automatically invoked by `Library:Init()`.

##### `Library:ToggleWatermark(State: boolean) -> nil`
Enables or disables visibility of the watermark HUD.

##### `Library:UpdateWatermark(Text: string) -> nil`
Overrides the watermark text. Supports Roblox RichText formatting.

##### `Library:GetConfig() -> string`
Serializes the state of all flags into a minified JSON string. Color pickers are serialized as normalized HSV tables.

##### `Library:LoadConfig(ConfigJSON: string) -> nil`
Parses a JSON configuration string and invokes `:Set()` on all matching flags.

##### `Library:UpdateConfigList(ListInstance: List, Mode: string) -> nil`
Populates or clears a `List` control with file basenames located in `gamesense/Configs`. Mode must be `"Add"` or `"Remove"`.

##### `Library:UpdateColor(ColorType: string, ColorValue: Color3) -> nil`
Updates a theme color globally across all registered gradients, strokes, and texts. `ColorType` must be `"Accent"`, `"SecondAccent"`, `"TextColor"`, or `"Risky"`.

##### `Library:Connection(Signal: RBXScriptSignal, Func: function, Name?: string, Table?: table) -> RBXScriptConnection`
Registers an RBXScriptSignal connection wrapped in a protected call with automated error reporting. Tracked in `Library.Connections` for automatic disconnection on unload.

##### `Library:ViewPlayer(Player: Player) -> nil`
Toggles camera focus between the specified player character Humanoid and the local player character.

##### `Library:NewFlag() -> string`
Generates a unique fallback flag name (e.g., `"UnknownFlag1"`).

---

### Window

The primary window frame object returned by `Library:Window(Options)`.

#### Constructor Options

```lua
local Window = Library:Window({
    Name = "gamesense",
    Size = UDim2.new(0, 700, 0, 612),
    MinResize = UDim2.new(0, 500, 0, 400),
    MaxResize = UDim2.new(0, 10000, 0, 10000),
    CloseBind = Enum.KeyCode.Insert,
})
```

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"gamesense"` | Title displayed in the window header bar. |
| `Size` | `UDim2` | `UDim2.new(0, 700, 0, 612)` | Initial dimensions of the window outline frame. |
| `MinResize` | `UDim2` | `UDim2.new(0, 500, 0, 400)` | Minimum resize boundary permitted when resizing is enabled. |
| `MaxResize` | `UDim2` | `UDim2.new(0, 10000, 0, 10000)` | Maximum resize boundary permitted. |
| `CloseBind` | `Enum.KeyCode` | `Enum.KeyCode.Insert` | Keyboard key utilized to show and hide the user interface. |

#### Methods

##### `Window:CreateTab(Options: table) -> Tab`
Registers a new navigation category on the left sidebar.

```lua
local Tab = Window:CreateTab({
    Icon = "rbxassetid://15453335745"
})
```

##### `Window:SetTab(Index: number) -> nil`
Switches the active displayed tab programmatically by 1-based index.

```lua
Window:SetTab(1)
```

---

### Tabs

Navigation containers displayed on the main content canvas. Created via `Window:CreateTab(Options)`.

#### Constructor Options

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Icon` | `string` | `"rbxassetid://8547236654"` | Roblox asset URL representing the sidebar tab icon. |

#### Methods

##### `Tab:Section(Options: table) -> Section`
Creates a standard dual-column groupbox section directly within the tab canvas.

##### `Tab:SubSection(Options: table) -> ...SectionItem`
Creates a horizontal sub-navigation strip across the top of the tab, returning unpacked `SectionItem` controllers for each option icon.

##### `Tab:ImageDropdown(Options: table) -> ImageDropdown`
Creates a weapon/profile dropdown selector featuring icon previews above the left column.

##### `Tab:Activate() -> nil`
Transitions the tab to the active state, playing smooth column entrance tweens.

##### `Tab:Deactivate() -> nil`
Hides the tab and transitions columns off-canvas.

---

### SubSections

SubSections provide nested multi-page navigation within a single main tab (such as dividing Visuals into Players, World, and Models).

#### Constructor Options

```lua
local Sub1, Sub2, Sub3 = Tab:SubSection({
    Name = "Category",
    Options = {
        "rbxassetid://18334627891",
        "rbxassetid://18334630306",
        "rbxassetid://18334626899",
    },
    Flag = "VisualCategory",
    Callback = function(selectedIndex)
        print("Switched to sub-category:", selectedIndex)
    end,
})
```

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Sub Section"` | Title text displayed on the subsection outline container. |
| `Options` | `table` | `{}` | Array of image asset strings for each sub-category icon button. |
| `Flag` | `string` | `Library:NewFlag()` | Optional identifier for state serialization. |
| `Callback` | `function` | `function() end` | Callback triggered when the active sub-tab index changes. |

#### Return Value

`Tab:SubSection()` returns multiple `SectionItem` controllers via `table.unpack()`. Each controller possesses its own independent column container and exposes the `:Section()` builder:

```lua
local SectionA = Sub1:Section({
    Name = "Esp Options",
    Side = "Left",
    Fill = true,
})
```

---

### Sections

Framed groupbox containers that host interactive controls.

#### Constructor Options

```lua
local Section = Tab:Section({
    Name = "General",
    Side = "Left",
    Fill = true,
    Size = UDim2.new(1, 0, 0, 200),
    Icon = nil,
})
```

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Section"` | Header text rendered with bold formatting in the frame gradient cutout. |
| `Side` | `string` | `"Left"` | Column alignment. Must be `"Left"` or `"Right"`. |
| `Fill` | `boolean` | `false` | When `true`, automatically expands the section height to consume remaining column space. |
| `Size` | `UDim2 | number` | `UDim2.new(1, 0, 0, 40)` | Fixed vertical dimension when `Fill` is `false`. Accepts UDim2 or number offset. |
| `Icon` | `string | nil` | `nil` | Optional icon asset rendered in the section title bar. |

#### Methods on Section

All controls are created via methods called on an instantiated `Section`:

- `Section:Toggle(Options)`
- `Section:Slider(Options)`
- `Section:Dropdown(Options)`
- `Section:MultiBox(Options)`
- `Section:Button(Options)`
- `Section:TextBox(Options)`
- `Section:Label(Options)`
- `Section:List(Options)`
- `Section:UpdateSection()`: Forces layout recalculation and updates internal scroll arrows if overflowing.

---

### Controls

#### Toggle

A two-state checkbox control with an accent gradient fill upon activation. Supports inline attachment of a `ColorPicker` and/or `Keybind`.

```lua
local AimToggle = Section:Toggle({
    Name = "Enable Aimbot",
    Default = false,
    Risky = false,
    Hidden = false,
    Flag = "AimbotToggle",
    Callback = function(state)
        print("Aimbot state:", state)
    end,
})

-- Optional attached inline controls:
AimToggle:ColorPicker({
    Default = Color3.fromRGB(255, 0, 0),
    Flag = "AimbotColor",
})

AimToggle:Keybind({
    Default = Enum.KeyCode.E,
    Mode = "Hold",
    Flag = "AimbotKey",
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Toggle"` | Label displayed alongside the checkbox. |
| `Default` | `boolean` | `false` | Initial boolean activation state. |
| `Risky` | `boolean` | `false` | When `true`, renders the text with the theme risky warning color (`Library.Theme.Default.Risky`). |
| `Hidden` | `boolean` | `false` | When `true`, instantiates the control in a hidden, collapsed state. |
| `Flag` | `string` | `Library:NewFlag()` | Flag key used in `Library.Flags`. |
| `Callback` | `function` | `function(state) end` | Invoked with the new boolean state whenever toggled. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `boolean` | Returns the current boolean state. |
| `:Set(Value)` | `Value: boolean` | `nil` | Sets the toggle state, runs animations, and invokes the callback. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the control row with height tweening. |
| `:GetName()` | None | `string` | Returns the display name string. |
| `:GetFlag()` | None | `string` | Returns the flag identifier. |
| `:GetState()` | None | `boolean` | Internal getter for state. |
| `:ColorPicker(Options)` | `Options: table` | `ColorPicker` | Attaches an inline color picker to the toggle row. |
| `:Keybind(Options)` | `Options: table` | `Keybind` | Attaches an inline keybind button to the toggle row. |

---

#### Slider

A draggable slider bar supporting floating-point precision, customizable suffixes, threshold labels, and step buttons.

```lua
local FOVSlider = Section:Slider({
    Name = "Field of View",
    Min = 0,
    Max = 180,
    Default = 90,
    Decimal = 1,
    Ending = "°",
    UseIcons = true,
    Disable = {"Disabled", 0, 0},
    Risky = false,
    OverrideLimit = false,
    Flag = "AimbotFOV",
    Callback = function(value)
        print("FOV set to:", value)
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Slider"` | Display name text above the slider bar. |
| `Min` | `number` | `0` | Minimum allowed value. |
| `Max` | `number` | `100` | Maximum allowed value. |
| `Default` | `number` | `1` | Initial numeric value. |
| `Decimal` | `number` | `1` | Rounding precision. Set to `1` for integers, `0.1` for one decimal place, `0.01` for two places. |
| `Ending` | `string` | `""` | Suffix string appended to the displayed numeric value (e.g., `"%"`, `"°"`, `" studs"`). |
| `UseIcons` | `boolean` | `true` | Renders `-` and `+` decrement and increment buttons on the sides. |
| `Disable` | `table` | `{}` | Format `{Label, MinThreshold, MaxThreshold}`. Displays `Label` instead of a number when within threshold. |
| `Risky` | `boolean` | `false` | When `true`, styles label using the risky theme color. |
| `OverrideLimit` | `boolean` | `false` | Allows dragging or setting values beyond the specified Max boundary. |
| `Hidden` | `boolean` | `false` | Instantiates control in a hidden state. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(value) end` | Invoked with the numeric value whenever modified. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `number` | Returns the current slider value. |
| `:Set(Value)` | `Value: number` | `nil` | Updates the slider value, positions the visual bar, and invokes callback. |
| `:Min()` | None | `number` | Returns the configured minimum boundary. |
| `:Max()` | None | `number` | Returns the configured maximum boundary. |
| `:GetName()` | None | `string` | Returns the display name. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the slider row. |

---

#### Dropdown

A compact single-selection dropdown menu with automated popup placement and smooth opening tweens.

```lua
local HitboxDropdown = Section:Dropdown({
    Name = "Target Hitbox",
    Content = {"Head", "Neck", "Chest", "Pelvis"},
    Default = "Head",
    Hidden = false,
    Flag = "TargetHitbox",
    Callback = function(selected)
        print("Selected hitbox:", selected)
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Dropdown"` | Display name label. Set to `""` for a compact headless dropdown. |
| `Content` | `table` | `{}` | Array of string options to display. |
| `Default` | `string` | `"None"` | Initially selected string. |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(selected) end` | Invoked with the selected string on change. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `string` | Returns the currently selected option string. |
| `:Set(State)` | `State: string` | `nil` | Selects the matching option and invokes the callback. |
| `:AddValue(Value)` | `Value: string` | `nil` | Dynamically appends a new option string to the menu list. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the dropdown row. |

---

#### MultiBox

A multi-selection dropdown menu allowing any number of options to be simultaneously toggled. Selected entries are concatenated in the preview box.

```lua
local TargetsMulti = Section:MultiBox({
    Name = "Target Prioritization",
    Content = {"Visible Only", "Enemies", "Knocked", "Airborne"},
    Default = {"Visible Only", "Enemies"},
    Flag = "TargetFlags",
    Callback = function(selectedTable)
        print("Active targets:", table.concat(selectedTable, ", "))
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview MultiBox"` | Display name label. |
| `Content` | `table` | `{}` | Array of available string options. |
| `Default` | `table | string` | `"None"` | Array of initially selected strings (or single string). |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(selectedList) end` | Invoked with an array of selected string keys. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `table` | Returns an array of selected string options (e.g., `{"Option 1", "Option 2"}`). |
| `:Set(Values)` | `Values: table` | `nil` | Activates all options specified in the `Values` array and deactivates others. |
| `:AddValue(Value)` | `Value: string` | `nil` | Appends a new selectable option string. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the multibox row. |

---

#### ColorPicker

A popup window providing visual RGB saturation/value matrix selection, hue bar scrolling, and an optional alpha transparency slider.

Can be attached inline to Toggles and Labels, or instantiated standalone via `Library:ColorPicker()`.

```lua
-- Attached to a Toggle:
local ChamsToggle = Section:Toggle({Name = "Player ESP", Flag = "ESPEnabled"})

local ESPColor = ChamsToggle:ColorPicker({
    Name = "ESP Color",
    Default = Color3.fromRGB(0, 170, 255),
    Alpha = 0.2,
    AlphaBar = true,
    Flag = "ESPColor",
    Callback = function(color, alpha)
        print("ESP Color changed:", color, "Alpha:", alpha)
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Color Picker"` | Title text displayed in the color picker popup frame. |
| `Default` | `Color3` | `Library.Theme.Default.Accent` | Initial Color3 value. |
| `Alpha` | `number` | `0` | Initial transparency value (`0` = fully opaque, `1` = fully transparent). |
| `AlphaBar` | `boolean` | `true` | When `true`, includes the vertical alpha transparency selection slider. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(color, alpha) end` | Invoked with `(Color3, AlphaNumber)` on every change. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `table` | Returns `{Color = Color3, Transparency = number}`. |
| `:Set(Color, Transparency?)` | `Color: Color3 | table`, `Transparency?: number` | `nil` | Sets active color and alpha. Accepts a `Color3` (with optional transparency number) or a config HSV table `{Hue, Sat, Val, Alpha}`. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the color picker row. |

---

#### Keybind

A hotkey listener supporting key capture, custom mouse buttons, display abbreviations, and multiple activation modes (`Toggle`, `Hold`, `Always on`, `On hotkey`).

Can be attached inline to Toggles and Labels, or created via `Library:Keybind()`.

```lua
local Toggle = Section:Toggle({Name = "Silent Aim", Flag = "SilentAim"})

local AimKey = Toggle:Keybind({
    Default = Enum.KeyCode.MouseRight,
    Mode = "Hold",
    UseMode = true,
    HideFromList = false,
    Flag = "SilentAimKey",
    Callback = function(key)
        print("Silent aim bound to:", key.Name)
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Default` | `EnumItem | string` | `Enum.KeyCode.Backspace` | Initial keybind. Accepts `Enum.KeyCode.*`, `Enum.UserInputType.*`, or string key name. |
| `Mode` | `string` | `"Toggle"` | Activation mode: `"Toggle"`, `"Hold"`, `"Always on"`, or `"On hotkey"`. |
| `UseMode` | `boolean` | `true` | When `true`, right-clicking the keybind displays a context menu to change mode. |
| `HideFromList` | `boolean` | `false` | When `true`, excludes keybind from HUD lists. |
| `Blacklisted` | `table` | `{}` | Array of `Enum.KeyCode` items forbidden from assignment. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(key) end` | Invoked when the bound key is reassigned. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `string` | Returns the bound key name string (e.g., `"Insert"`, `"Q"`, `"MouseButton2"`). |
| `:Active()` | None | `boolean` | Returns `true` if the keybind is currently active under its current activation mode. |
| `:Set(Key)` | `Key: EnumItem | string` | `nil` | Programmatically assigns the bound key. |
| `:SetMode(Mode)` | `Mode: string` | `nil` | Sets the activation mode (`"Always on"`, `"Toggle"`, `"Hold"`, `"On hotkey"`). |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the keybind row. |

---

#### Button

A full-width interactive button with hover animations, optional confirmation prompt, and risky styling.

```lua
Section:Button({
    Name = "Reset Player Cache",
    Confirmation = true,
    Risky = false,
    Callback = function()
        print("Confirmed and executed reset.")
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview Button"` | Label text displayed on the button. |
| `Confirmation` | `boolean` | `false` | When `true`, clicking once prompts "Are you sure?". Second click executes callback. |
| `Risky` | `boolean` | `false` | When `true`, styles button with warning accent color. |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Callback` | `function` | `function() end` | Callback executed when clicked (or confirmed). |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the button row. |
| `:UpdateSize(Size)` | `Size: UDim2` | `nil` | Updates the button dimensions. |
| `:UpdatePosition(Position)` | `Position: UDim2` | `nil` | Updates the button position. |

---

#### TextBox

A single-line input field supporting character limits, numeric filtering, focus clearing, and enter-to-commit modes.

```lua
Section:TextBox({
    Name = "Config Name",
    Default = "Legit_01",
    Max = 24,
    NumbersOnly = false,
    ClearOnFocus = false,
    CheckIfPressedEnter = true,
    Flag = "ConfigNameInput",
    Callback = function(text)
        print("Config name committed:", text)
    end,
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Name` | `string` | `"Preview TextBox"` | Header label displayed above the input frame. |
| `Default` | `string` | `""` | Initial string content. |
| `Max` | `number` | `32` | Maximum character length allowed. |
| `NumbersOnly` | `boolean` | `false` | When `true`, strips non-numeric characters automatically. |
| `ClearOnFocus` | `boolean` | `false` | When `true`, empties text when the user clicks inside. |
| `CheckIfPressedEnter`| `boolean` | `false` | When `true`, fires callback only when the Return key is pressed. |
| `TypedCheck` | `boolean` | `false` | When `true`, validates typing via pattern checks. |
| `Risky` | `boolean` | `false` | Renders with risky styling. |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(text) end` | Invoked when text changes or is committed. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `string` | Returns current string content. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the textbox row. |

---

#### Label

A static or dynamic text display row supporting RichText tags, optional click callbacks, and attached inline ColorPickers or Keybinds.

```lua
local StatusLabel = Section:Label({
    Message = "Status: <font color='rgb(153, 196, 39)'>Connected</font>",
    Side = "Left",
    Risky = false,
})

StatusLabel:ColorPicker({
    Default = Color3.fromRGB(153, 196, 39),
    Flag = "AccentColorOverride",
})
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Message` | `string` | `"Preview Label"` | Text content to display. Supports Roblox RichText tags. |
| `Side` | `string` | `"Left"` | Column alignment (`"Left"` or `"Right"`). |
| `Risky` | `boolean` | `false` | Renders using the risky theme color. |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Callback` | `function` | `function() end` | Optional callback fired when the label is clicked. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:GetName()` | None | `string` | Returns the current label message text. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the label row. |
| `:ColorPicker(Options)`| `Options: table` | `ColorPicker` | Attaches an inline color picker to the label. |
| `:Keybind(Options)` | `Options: table` | `Keybind` | Attaches an inline keybind to the label. |
| `:Update()` | None | `nil` | Triggers a visual re-render of label elements. |

---

#### List

A scrollable item list box equipped with search arrows, dynamic item insertion/removal, and optional leading thumbnail icons. Frequently used for config file selection and player rosters.

```lua
local ConfigList = Section:List({
    Size = 160,
    Hidden = false,
    Flag = "SelectedConfig",
    Callback = function(selectedName)
        print("Selected profile:", selectedName)
    end,
})

-- Adding an item with thumbnail:
ConfigList:AddValue("Legit_v1", {
    Image = "rbxassetid://12206409737",
    Color = Color3.fromRGB(153, 196, 39),
})

-- Removing an item:
ConfigList:RemoveValue("Legit_v1")
```

##### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `Size` | `number` | `100` | Height of the scrollable list container in pixels. |
| `Hidden` | `boolean` | `false` | Instantiates control collapsed. |
| `Flag` | `string` | `Library.NewFlag()` | Flag identifier. |
| `Callback` | `function` | `function(selected) end` | Invoked when an item in the list is clicked. |

##### Methods

| Method | Parameters | Return | Description |
| :--- | :--- | :--- | :--- |
| `:Get()` | None | `string` | Returns the currently selected item string. |
| `:AddValue(Value, Icon?)` | `Value: string`, `Icon?: table` | `nil` | Appends a new item. `Icon` is an optional table `{Image: string, Color?: Color3, Size?: UDim2, Position?: UDim2}`. |
| `:RemoveValue(Value)` | `Value: string` | `nil` | Removes the matching string entry from the list. |
| `:SetVisible(Bool)` | `Bool: boolean` | `nil` | Shows or hides the list row. |
| `:UpdateSection()` | None | `nil` | Recalculates scroll boundaries and section heights. |

---

## State Management and Flags

Every control registered with a `Flag` string option registers itself in the central `Library.Flags` table:

```lua
-- Reading a flag:
local isAimbotOn = Library.Flags["AimbotEnabled"]:Get()
local fovRadius  = Library.Flags["AimbotFOV"]:Get()
local targetPart = Library.Flags["TargetHitbox"]:Get()
local selected   = Library.Flags["TargetFlags"]:Get() -- array of strings

-- Updating a flag programmatically:
Library.Flags["AimbotEnabled"]:Set(true)
Library.Flags["AimbotFOV"]:Set(120)
Library.Flags["TargetHitbox"]:Set("Chest")
```

### Keybind Querying

Keybind instances provide an `:Active()` method that evaluates current key hold/toggle state according to the selected activation mode:

```lua
RunService.RenderStepped:Connect(function()
    if Library.Flags["AimbotEnabled"]:Get() and Library.Flags["AimbotKey"]:Active() then
        -- Execute aimbot step
    end
end)
```

---

## Configuration System (JSON)

Gamesense UI Library features an automated JSON configuration serialization engine that reads and restores all flags.

### Serialization Mechanics

- **Scalars and Strings**: Booleans, numbers, and strings are saved directly.
- **ColorPickers**: Serialized as a 4-element array containing normalized HSV values and alpha transparency: `{Hue, Saturation, Value, Transparency}`.
- **Internal Status Flags**: Flags ending with `_Status` are filtered out automatically.

### File Storage Integration

Using the executor file API (`writefile`, `readfile`, `isfolder`, `makefolder`, `listfiles`):

```lua
local ConfigSection = ConfigTab:Section({Name = "Profiles", Fill = true})
local ProfileList = ConfigSection:List({Size = 140, Flag = "ActiveProfile"})

-- Populate file list
local function RefreshProfiles()
    ProfileList:UpdateSection()
    Library:UpdateConfigList(ProfileList, "Remove")
    Library:UpdateConfigList(ProfileList, "Add")
end

-- Save Profile
ConfigSection:Button({
    Name = "Save Config",
    Callback = function()
        local name = Library.Flags["ActiveProfile"]:Get()
        if name and name ~= "" then
            writefile("gamesense/Configs/" .. name .. ".cfg", Library:GetConfig())
            Library:Notify({Message = "Saved profile: " .. name, Delay = 3})
        end
    end,
})

-- Load Profile
ConfigSection:Button({
    Name = "Load Config",
    Callback = function()
        local name = Library.Flags["ActiveProfile"]:Get()
        local path = "gamesense/Configs/" .. name .. ".cfg"
        if isfile(path) then
            Library:LoadConfig(readfile(path))
            Library:Notify({Message = "Loaded profile: " .. name, Delay = 3})
        end
    end,
})

-- Refresh list
ConfigSection:Button({
    Name = "Refresh List",
    Callback = RefreshProfiles,
})
```

---

## System Utilities

### Notifications

Toast notifications stack vertically in the top-left or middle of the screen with smooth sliding tweens and customizable delays.

```lua
Library:Notify({
    Message = "hit <font color='rgb(153, 196, 39)'>TargetPlayer</font> for <font color='rgb(255, 80, 80)'>100</font> damage",
    Delay = 4,
    Position = "Top Left", -- or "Middle"
})
```

### Watermark

The HUD watermark sits at the top right of the viewport and renders branding, current FPS, and local time.

```lua
-- Toggle watermark display:
Library:ToggleWatermark(true)

-- Manual text update:
Library:UpdateWatermark("gamesense | user: admin | 144 FPS | " .. os.date("%X"))
```

### Connection Management

Use `Library:Connection` to wrap event listeners. Connections registered through this method are safely disconnected during `Library:Unload()`:

```lua
Library:Connection(Players.PlayerAdded, function(player)
    print("Player joined:", player.Name)
end, "PlayerJoinListener")
```

### Player Viewport Utilities

`Library:ViewPlayer(player: Player)` switches the local player camera subject to follow the target player character. Invoking it a second time reverts focus to the local player.

```lua
Library:ViewPlayer(targetPlayer)
```

---

## Theming and Color System

Gamesense employs a dark charcoal color scheme with vibrant accent highlights.

### Default Palette

| Role | Property | Default Color |
| :--- | :--- | :--- |
| Primary Accent | `Library.Theme.Default.Accent` | `Color3.fromRGB(153, 196, 39)` (Gamesense Green) |
| Secondary Accent | `Library.Theme.Default.SecondAccent` | `Color3.fromRGB(124, 158, 32)` |
| Body Text | `Library.Theme.Default.TextColor` | `Color3.fromRGB(205, 205, 205)` |
| Risky Option | `Library.Theme.Default.Risky` | `Color3.fromRGB(165, 165, 120)` |

### Runtime Recoloring

Call `Library:UpdateColor()` to modify theme colors across all active gradients, frames, and controls:

```lua
-- Change accent to azure blue:
Library:UpdateColor("Accent", Color3.fromRGB(59, 175, 230))
Library:UpdateColor("SecondAccent", Color3.fromRGB(39, 140, 195))
```

---

## Lifecycle and Resource Cleanup

The library provides robust teardown functionality to ensure clean restarts during script development:

```lua
-- Clean unload via hotkey or script button:
SettingsSection:Button({
    Name = "Unload",
    Confirmation = true,
    Callback = function()
        Library:Unload()
    end,
})

-- Master disable: sets all flags to false without destroying GUI:
SettingsSection:Button({
    Name = "Disable All Features",
    Callback = function()
        Library:Disable()
    end,
})
```

---

## Complete Working Example

```lua
local rawLibrary = game:HttpGet("https://raw.githubusercontent.com/ugt5/gamesense/refs/heads/main/gamesense.lua")
local chunk = loadstring(rawLibrary, "@Gamesense")
local Library = chunk() or getgenv().Library

-- 1. Initialize Window
local Window = Library:Window({
    Name = "gamesense",
    Size = UDim2.new(0, 700, 0, 612),
    CloseBind = Enum.KeyCode.Insert,
})

-- 2. Create Tabs
local CombatTab   = Window:CreateTab({Icon = "rbxassetid://15453335745"})
local VisualsTab  = Window:CreateTab({Icon = "rbxassetid://15453344494"})
local SettingsTab = Window:CreateTab({Icon = "rbxassetid://15453349637"})

-- -------------------------------------------------------------------------
-- Combat Tab
-- -------------------------------------------------------------------------
local AimbotSection = CombatTab:Section({Name = "Aimbot", Side = "Left", Fill = true})
local TriggerSection = CombatTab:Section({Name = "Triggerbot", Side = "Right", Fill = true})

local AimToggle = AimbotSection:Toggle({
    Name = "Enabled",
    Default = false,
    Flag = "Aimbot_Enabled",
    Callback = function(state)
        print("Aimbot state:", state)
    end,
})

AimToggle:Keybind({
    Default = Enum.KeyCode.E,
    Mode = "Hold",
    Flag = "Aimbot_Key",
})

AimToggle:ColorPicker({
    Default = Color3.fromRGB(153, 196, 39),
    Flag = "Aimbot_FOVColor",
})

AimbotSection:Slider({
    Name = "Field of View",
    Min = 0,
    Max = 180,
    Default = 60,
    Decimal = 1,
    Ending = "°",
    Flag = "Aimbot_FOV",
})

AimbotSection:Dropdown({
    Name = "Target Bone",
    Content = {"Head", "Neck", "UpperTorso", "LowerTorso"},
    Default = "Head",
    Flag = "Aimbot_Bone",
})

AimbotSection:MultiBox({
    Name = "Target Filters",
    Content = {"Visible Only", "Team Check", "Ignore Shields"},
    Default = {"Visible Only", "Team Check"},
    Flag = "Aimbot_Filters",
})

TriggerSection:Toggle({
    Name = "Triggerbot Enabled",
    Default = false,
    Flag = "Trigger_Enabled",
})

TriggerSection:Slider({
    Name = "Reaction Delay",
    Min = 0,
    Max = 200,
    Default = 25,
    Decimal = 1,
    Ending = " ms",
    Flag = "Trigger_Delay",
})

-- -------------------------------------------------------------------------
-- Visuals Tab (with SubSections)
-- -------------------------------------------------------------------------
local PlayersSub, WorldSub = VisualsTab:SubSection({
    Name = "Category",
    Options = {
        "rbxassetid://18334627891",
        "rbxassetid://18334630306",
    },
    Flag = "VisualSubCategory",
})

local ESPSection = PlayersSub:Section({Name = "ESP Options", Side = "Left", Fill = true})
local VisualSection = PlayersSub:Section({Name = "Chams", Side = "Right", Fill = true})

ESPSection:Toggle({
    Name = "Player Box",
    Default = false,
    Flag = "ESP_Box",
}):ColorPicker({
    Default = Color3.fromRGB(255, 255, 255),
    Flag = "ESP_BoxColor",
})

ESPSection:Toggle({
    Name = "Player Skeleton",
    Default = false,
    Flag = "ESP_Skeleton",
})

VisualSection:Dropdown({
    Name = "Material",
    Content = {"Flat", "Wireframe", "Glow"},
    Default = "Flat",
    Flag = "Chams_Material",
})

-- -------------------------------------------------------------------------
-- Settings Tab
-- -------------------------------------------------------------------------
local ConfigSection = SettingsTab:Section({Name = "Configuration", Side = "Left", Fill = true})
local SystemSection = SettingsTab:Section({Name = "System", Side = "Right", Fill = true})

local ProfileList = ConfigSection:List({Size = 130, Flag = "Profile_List"})

ConfigSection:TextBox({
    Name = "New Profile Name",
    Default = "default",
    Flag = "Profile_NewName",
})

ConfigSection:Button({
    Name = "Save Config",
    Callback = function()
        local name = Library.Flags["Profile_NewName"]:Get()
        if name and #name > 0 then
            writefile("gamesense/Configs/" .. name .. ".cfg", Library:GetConfig())
            ProfileList:AddValue(name)
            Library:Notify({Message = "Saved profile: " .. name, Delay = 3})
        end
    end,
})

ConfigSection:Button({
    Name = "Load Config",
    Callback = function()
        local selected = ProfileList:Get()
        local path = "gamesense/Configs/" .. selected .. ".cfg"
        if selected and isfile(path) then
            Library:LoadConfig(readfile(path))
            Library:Notify({Message = "Loaded profile: " .. selected, Delay = 3})
        end
    end,
})

SystemSection:Label({Message = "Menu Keybind"}):Keybind({
    Default = Enum.KeyCode.Insert,
    UseMode = false,
    Callback = function(key)
        Library.UI.CloseBind = key
    end,
})

SystemSection:Button({
    Name = "Unload Interface",
    Confirmation = true,
    Callback = function()
        Library:Unload()
    end,
})

-- 3. Finalize and Init
Library:Init()

Library:Notify({
    Message = "Welcome to <font color='rgb(153, 196, 39)'>gamesense</font>",
    Delay = 5,
    Position = "Top Left",
})
```

# Gamesense UI Framework for Roblox (Luau)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Roblox](https://img.shields.io/badge/Platform-Roblox-black?logo=roblox)](https://roblox.com)
[![Language: Luau](https://img.shields.io/badge/Language-Luau-00A2FF?logo=lua)](https://luau-lang.org)

A high-fidelity implementation of the classic Gamesense / Skeet graphical user interface for Roblox executors and Luau environments. Engineered with modular column-based layouts, responsive drag-and-resize mechanics, inline control attachments, integrated search lists, a native JSON configuration system, and low-overhead rendering.

---

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [API Reference](#api-reference)
  - [Window Configuration](#1-window-configuration)
  - [Tabs and SubSections](#2-tabs-and-subsections)
  - [Sections (Columns & Layouts)](#3-sections-columns--layouts)
  - [Interactive Controls](#4-interactive-controls)
    - [Toggle](#toggle)
    - [Slider](#slider)
    - [Dropdown (Single-Selection)](#dropdown-single-selection)
    - [MultiBox (Multi-Selection)](#multibox-multi-selection)
    - [ColorPicker](#colorpicker)
    - [Keybind](#keybind)
    - [Button](#button)
    - [TextBox](#textbox)
    - [Label](#label)
    - [List](#list)
- [State Management and Flags](#state-management-and-flags)
- [Configuration System (JSON Serialization)](#configuration-system-json-serialization)
- [System Utilities (Watermark & Notifications)](#system-utilities)
- [Lifecycle & Cleanup](#lifecycle--cleanup)
- [Complete Working Example](#complete-working-example)
- [License](#license)

---

## Overview

- **Pixel-Accurate Visuals**: 1px border hierarchy, stippled canvas backgrounds, and classic Gamesense color schemes.
- **Dynamic Layout Engine**: Dual-column architecture (`Left` and `Right`) supporting vertical auto-filling, run-time dragging, and interactive resizing.
- **Inline Chaining**: Attach dedicated `Keybind` and `ColorPicker` widgets directly to `Toggle` and `Label` components.
- **Searchable Listbox**: Virtualized item lists with integrated real-time text filtering and optional thumbnail/icon support.
- **Zero-Dependency Config Serialization**: Native JSON export and import for all registered flag states without external dependencies.
- **Fault-Tolerant Unload**: Complete lifecycle management that disconnects global listeners and purges GUI instances on script termination.

---

## Quick Start

Fetch and execute the raw source directly within your executor environment:

```lua
local source = game:HttpGet("https://raw.githubusercontent.com/ugt5/gamesense/refs/heads/main/gamesense.lua")

-- Strip embedded demo routine if present
local cutIndex = source:find("local Window = Library:Window", 1, true)
if cutIndex then
    source = source:sub(1, cutIndex - 1) .. "\nreturn Library\n"
end

local Library = loadstring(source)() or getgenv().Library
```

---

## Architecture

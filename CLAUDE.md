# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hyprland animated notification plugin that creates sliding notifications with particle effects. The plugin integrates with notification daemons like dunst and SwayNC, intercepting their notifications to display them with custom animations.

## Architecture

### Core Components

- **main.cpp**: Plugin entry point and coordinator
- **notification.hpp/cpp**: Core notification rendering with Cairo/Pango
- **particle_system.hpp/cpp**: Particle effects for "break" animations
- **animation_manager.hpp/cpp**: Manages multiple notifications and rendering
- **dbus_listener.hpp/cpp**: DBus integration for intercepting notifications
- **window_interceptor.hpp/cpp**: Hides original notification windows
- **notification_data.hpp**: Data structures for notification content

### Integration Pattern

The plugin works by:

1. Registering as a DBus notification service
2. Intercepting notifications from dunst/SwayNC
3. Hiding the original notification windows
4. Rendering custom animated notifications with particle effects

## Critical Build Requirements

### Plugin Entry Points (ESSENTIAL)

Hyprland expects specific camelCase function names:

```cpp
extern "C" {
    APICALL EXPORT const char* pluginAPIVersion() {
        return HYPRLAND_API_VERSION;
    }

    APICALL EXPORT PLUGIN_DESCRIPTION_INFO pluginInit(HANDLE handle) {
        // Plugin initialization
        return {"Plugin Name", "Description", "Author", "Version"};
    }

    APICALL EXPORT void pluginExit() {
        // Plugin cleanup
    }
}
```

### Header Paths (NixOS/Nix Development)

Use `hyprland/` prefix for all Hyprland headers:

```cpp
#include <hyprland/src/plugins/PluginAPI.hpp>
#include <hyprland/src/Compositor.hpp>
#include <hyprland/src/desktop/Window.hpp>
```

### Build System (MANDATORY)

Use CMake with pkg-config, not manual Makefiles:

```cmake
cmake_minimum_required(VERSION 3.16)
project(notification-plugin VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(PkgConfig REQUIRED)

pkg_check_modules(deps REQUIRED IMPORTED_TARGET
    hyprland
    pixman-1
    wayland-server
    cairo
    pango
    glib-2.0
    gio-2.0
)

add_library(plugin SHARED
    src/main.cpp
    src/notification.cpp
    src/particle_system.cpp
    src/animation_manager.cpp
    src/dbus_listener.cpp
    src/window_interceptor.cpp
)

target_link_libraries(plugin PRIVATE PkgConfig::deps)
target_compile_options(plugin PRIVATE -Wall -Wextra -fPIC)
```

## Build Commands

### Dependencies (NixOS)

```bash
# Development shell
nix develop

# Or add to configuration.nix:
environment.systemPackages = with pkgs; [
  hyprland cairo pango glib libnotify pkg-config gcc cmake
];
```

### Build Commands

```bash
# CMake build (REQUIRED - do not use make)
cmake -B build -S .
cmake --build build

# Install to Hyprland plugins
mkdir -p ~/.config/hypr/plugins/
cp build/plugin.so ~/.config/hypr/plugins/notification-plugin.so
```

### Testing

```bash
# Load plugin
hyprctl plugin load ~/.config/hypr/plugins/notification-plugin.so

# Test basic notification
hyprctl dispatch notification "Test message"

# Configure integration
hyprctl dispatch notification_config enable_dunst
hyprctl dispatch notification_config enable_swaync

# Debug commands
nm build/plugin.so | grep plugin  # Verify symbol exports
ldd build/plugin.so              # Check dependencies
```

## Version Compatibility

Target Hyprland 0.50.0 (commit: cb6589db98325705cef5dcaf92ccdf41ab21386d):

- aquamarine 0.9.2
- hyprlang 0.6.3  
- hyprutils 0.8.2
- hyprcursor 0.1.13
- hyprgraphics 0.1.5

Always include version checking in pluginInit():

```cpp
const std::string HASH = __hyprland_api_get_hash();
if (HASH != GIT_COMMIT_HASH) {
    throw std::runtime_error("API version mismatch");
}
```

## Code Conventions

### File Organization

- Header files define interfaces and data structures
- Implementation files contain Cairo/OpenGL rendering logic
- Separate concerns: DBus, rendering, animation, window management

### Key Technologies

- **Hyprland Plugin API**: Core plugin integration
- **Cairo/Pango**: Text and graphics rendering
- **DBus/GIO**: Notification daemon integration
- **C++23**: Modern C++ with smart pointers and RAII

### Animation System

- State machine: SLIDING_IN → BREAKING → SHOWING → SLIDING_OUT
- Urgency-based timing and effects (LOW/NORMAL/CRITICAL)
- Particle systems for visual effects during "break" phase

## Configuration Files

- `config/dunst_integration.conf`: Dunst configuration to disable native display
- `config/swaync_integration.conf`: SwayNC configuration for plugin integration
- Load plugin in `~/.config/hypr/hyprland.conf` with: `plugin = ~/.config/hypr/plugins/notification-plugin.so`

## Development Notes

### Rendering Pipeline

- Cairo surface creation for each notification
- Pango layout for text rendering
- Particle system updates in render loop
- OpenGL integration through Hyprland's renderer

### DBus Integration

- Implements `org.freedesktop.Notifications` interface
- Parses notification parameters (app_name, summary, body, urgency, etc.)
- Handles notification actions and hints

### Memory Management

- RAII with smart pointers for notification objects
- Cairo surface/context cleanup in destructors
- Bounded particle systems to prevent memory leaks

## Quality Assurance

- Always run pre-commit hooks to format and check code
- Run GitHub Actions before committing and pushing
- Never use emoji icons in documentation
- Start with minimal plugin to test loading before adding functionality
- Verify symbol exports: `nm plugin.so | grep plugin`
- Check dependencies: `ldd plugin.so`

## Emergency Debugging

When plugin crashes occur:

```bash
# 1. Check Hyprland logs
journalctl -u hyprland --since "5 minutes ago"

# 2. Check symbol exports
nm plugin.so | grep plugin

# 3. Check dependencies  
ldd plugin.so

# 4. Test minimal load
hyprctl plugin load $(pwd)/plugin.so

# 5. Check for core dumps
coredumpctl list hyprland
```
# CLAUDE.md - Hyprland Plugin Development Guide

This file provides comprehensive guidance for Claude Code when developing Hyprland plugins, based on extensive troubleshooting and research into best practices.

## Critical Hyprland Plugin Development Rules

### 🚨 **ESSENTIAL: Plugin Entry Point Functions**

Hyprland expects **specific function names in camelCase format**:

```cpp
extern "C" {
    APICALL EXPORT const char* pluginAPIVersion() {
        return HYPRLAND_API_VERSION;
    }

    APICALL EXPORT PLUGIN_DESCRIPTION_INFO pluginInit(HANDLE handle) {
        // Plugin initialization code
        return {"Plugin Name", "Description", "Author", "Version"};
    }

    APICALL EXPORT void pluginExit() {
        // Plugin cleanup code
    }
}
```

**❌ WRONG**: `PLUGIN_API_VERSION()`, `PLUGIN_INIT()`, `PLUGIN_EXIT()`
**✅ CORRECT**: `pluginAPIVersion()`, `pluginInit()`, `pluginExit()`

### 🚨 **ESSENTIAL: Header Inclusion Paths**

When using Nix development environment, headers MUST include the `hyprland/` prefix:

```cpp
// ✅ CORRECT for Nix environments
#include <hyprland/src/plugins/PluginAPI.hpp>
#include <hyprland/src/Compositor.hpp>
#include <hyprland/src/desktop/Window.hpp>

// ❌ WRONG - will cause compilation failures
#include <src/plugins/PluginAPI.hpp>
#include <src/Compositor.hpp>
```

### 🚨 **ESSENTIAL: Use CMake Build System**

**DO NOT** use manual compilation commands. Use CMake with pkg-config like official plugins:

```cmake
cmake_minimum_required(VERSION 3.16)
project(hyprland-plugin VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(PkgConfig REQUIRED)

# Use pkg-config to find dependencies like official plugins
pkg_check_modules(deps REQUIRED IMPORTED_TARGET
    hyprland
    pixman-1
    wayland-server
)

add_library(plugin SHARED
    main.cpp
    # ... other source files
)

target_link_libraries(plugin PRIVATE PkgConfig::deps)
target_compile_options(plugin PRIVATE -Wall -Wextra -fPIC)
```

## Troubleshooting Plugin Crashes

### Common Crash Causes and Solutions

#### 1. **Symbol Naming Issues**
**Symptom**: "missing apiver/init func" errors
**Solution**: Use camelCase function names (see above)

#### 2. **Header Path Issues**  
**Symptom**: "No such file or directory" for headers
**Solution**: Add `hyprland/` prefix to all Hyprland header includes

#### 3. **ABI Mismatch**
**Symptom**: Plugin loads but crashes Hyprland immediately
**Solution**: Ensure exact version matching between build headers and runtime Hyprland:

```bash
# Check Hyprland version
hyprctl version

# Pin flake.nix to exact commit
hyprland = {
  url = "github:hyprwm/Hyprland/EXACT_COMMIT_HASH";
  inputs.nixpkgs.follows = "nixpkgs";
};
```

#### 4. **C++ Symbol Mangling**
**Symptom**: Exported functions exist but aren't recognized
**Solution**: Always use `extern "C"` wrapper around plugin functions

### Debugging Process

1. **Create Minimal Test Plugin**:
```cpp
#include <hyprland/src/plugins/PluginAPI.hpp>

inline HANDLE PHANDLE = nullptr;

extern "C" {
    APICALL EXPORT const char* pluginAPIVersion() {
        return HYPRLAND_API_VERSION;
    }

    APICALL EXPORT PLUGIN_DESCRIPTION_INFO pluginInit(HANDLE handle) {
        PHANDLE = handle;
        
        // Start with minimal functionality
        HyprlandAPI::addNotification(PHANDLE, "[Test] Plugin loaded!", 
                                     CHyprColor{0.2, 1.0, 0.2, 1.0}, 3000);
        
        return {"Test Plugin", "Minimal test", "Test", "1.0.0"};
    }

    APICALL EXPORT void pluginExit() {
        // Clean exit
    }
}
```

2. **Verify Symbol Export**:
```bash
nm plugin.so | grep -E "(plugin|API)"
# Should show: pluginAPIVersion, pluginInit, pluginExit
```

3. **Check Dependencies**:
```bash
ldd plugin.so
# Verify all libraries are found
```

## Nix Flake Configuration

### Proper Flake Structure for Hyprland Plugins

```nix
{
  description = "Hyprland Plugin";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
    
    # Pin to specific commit for ABI compatibility
    hyprland = {
      url = "github:hyprwm/Hyprland/COMMIT_HASH";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, flake-utils, hyprland }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
        hyprlandDev = hyprland.packages.${system}.hyprland-debug;

        plugin = pkgs.stdenv.mkDerivation rec {
          pname = "hyprland-plugin";
          version = "1.0.0";
          src = ./.;

          nativeBuildInputs = with pkgs; [
            cmake
            gcc
            pkg-config
          ];

          buildInputs = with pkgs; [
            pixman
            libdrm
            hyprlandDev
            wayland
          ] ++ hyprlandDev.buildInputs;

          configurePhase = ''
            runHook preConfigure
            cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
            runHook postConfigure
          '';

          buildPhase = ''
            runHook preBuild
            cmake --build build
            runHook postBuild
          '';

          installPhase = ''
            runHook preInstall
            mkdir -p $out/lib
            cp build/plugin.so $out/lib/libhyprland-plugin.so
            runHook postInstall
          '';
        };

      in {
        packages.default = plugin;
        
        devShells.default = pkgs.mkShell {
          nativeBuildInputs = with pkgs; [
            cmake
            gcc
            pkg-config
            gdb
            valgrind
          ];

          buildInputs = with pkgs; [
            hyprlandDev
            pixman
            libdrm
            wayland
          ] ++ hyprlandDev.buildInputs;

          PKG_CONFIG_PATH = "${pkgs.lib.makeSearchPathOutput "lib" "lib/pkgconfig" [
            pkgs.pixman
            pkgs.libdrm
            hyprlandDev
          ]}";

          HYPRLAND_HEADERS = "${hyprlandDev}/include";
        };
      }
    );
}
```

### Flake Dependency Issues

**Common Problem**: Circular dependencies in Hyprland ecosystem
**Solution**:
1. Delete `flake.lock` completely
2. Run `nix flake lock` to regenerate
3. Pin to working Hyprland commit
4. Test build immediately after regeneration

## Research-Based Best Practices

### Official Plugin Patterns

Based on research of `github:hyprwm/hyprland-plugins`:

1. **Use CMake with pkg-config** - This is how all official plugins build
2. **Follow camelCase naming** - Required by plugin loader  
3. **Use proper error handling** - Always check return values
4. **Implement version checking** - Compare `HYPRLAND_API_VERSION`

### Plugin Development Workflow

1. **Start with CMake template** (not manual compilation)
2. **Create minimal test plugin first** 
3. **Verify symbols and loading** before adding functionality
4. **Use proper header paths** with `hyprland/` prefix
5. **Test with exact version matching**

### Common Anti-Patterns to Avoid

❌ **Manual compilation with g++ flags** - Use CMake
❌ **Hardcoded header paths** - Use pkg-config  
❌ **Mixed build systems** - Be consistent
❌ **Ignoring version mismatches** - Always check compatibility
❌ **Complex Nix builds in CI** - Keep CI simple

## Development Environment Setup

### Required Tools

```bash
# Development shell should include:
nativeBuildInputs = with pkgs; [
  cmake          # Build system
  gcc            # Compiler
  pkg-config     # Dependency management
  gdb            # Debugging
  valgrind       # Memory checking
];

buildInputs = with pkgs; [
  hyprlandDev    # Hyprland headers and libs
  pixman         # Graphics library
  libdrm         # Direct rendering
  wayland        # Wayland protocol
];
```

### Testing Strategy

1. **Symbol Export Test**: Verify function names with `nm`
2. **Dependency Test**: Check library linking with `ldd`
3. **Minimal Load Test**: Test basic plugin loading
4. **Functionality Test**: Verify API calls work
5. **Memory Test**: Use valgrind for leak detection

## Plugin Architecture Patterns

### Proper Plugin Structure

```cpp
// main.cpp - Plugin entry point
#include <hyprland/src/plugins/PluginAPI.hpp>
#include "PluginClass.hpp"

inline HANDLE PHANDLE = nullptr;
std::unique_ptr<PluginClass> g_pPlugin;

extern "C" {
    APICALL EXPORT const char* pluginAPIVersion() {
        return HYPRLAND_API_VERSION;
    }

    APICALL EXPORT PLUGIN_DESCRIPTION_INFO pluginInit(HANDLE handle) {
        PHANDLE = handle;
        
        // Version check
        const std::string HASH = __hyprland_api_get_hash();
        if (HASH != GIT_COMMIT_HASH) {
            HyprlandAPI::addNotification(PHANDLE, "[Plugin] Version mismatch!", 
                                       CHyprColor{1.0, 0.2, 0.2, 1.0}, 5000);
            throw std::runtime_error("Version mismatch");
        }

        // Initialize plugin
        g_pPlugin = std::make_unique<PluginClass>(PHANDLE);
        
        return {"Plugin Name", "Description", "Author", "Version"};
    }

    APICALL EXPORT void pluginExit() {
        if (g_pPlugin) {
            g_pPlugin.reset();
        }
    }
}
```

### Memory Management

- **Plugin objects**: Use `std::unique_ptr`
- **Hyprland objects**: Use raw pointers (managed by Hyprland)
- **Event callbacks**: Store shared_ptr to ensure cleanup
- **Always register/unregister** event hooks properly

## Version Compatibility

### Hyprland Version Pinning

```bash
# Check current Hyprland version
hyprctl version
# Output: Hyprland 0.50.0 built from branch at commit cb6589db98...

# Pin flake to exact commit
hyprland = {
  url = "github:hyprwm/Hyprland/cb6589db98325705cef5dcaf92ccdf41ab21386d";
  inputs.nixpkgs.follows = "nixpkgs";
};
```

### API Compatibility Checking

```cpp
// Always include version check in pluginInit
const std::string HASH = __hyprland_api_get_hash();
if (HASH != GIT_COMMIT_HASH) {
    // Handle version mismatch
    throw std::runtime_error("API version mismatch");
}
```

## Emergency Debugging Commands

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

## Quick Reference

### ✅ Plugin Development Checklist

- [ ] Use CMake build system
- [ ] Include `hyprland/` prefix in headers  
- [ ] Use camelCase function names (`pluginInit`, etc.)
- [ ] Wrap functions in `extern "C"`
- [ ] Pin Hyprland to exact commit in flake.nix
- [ ] Test with minimal plugin first
- [ ] Verify symbol export with `nm`
- [ ] Check dependencies with `ldd`
- [ ] Include version checking in `pluginInit`
- [ ] Use proper memory management patterns

This guide represents comprehensive research into Hyprland plugin development best practices and should resolve the majority of common plugin development issues.
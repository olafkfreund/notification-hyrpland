# Hyprland Animated Notification Plugin

A high-performance Hyprland plugin that creates animated sliding notifications with particle effects. The plugin seamlessly integrates with existing notification daemons like dunst and SwayNC, intercepting their notifications to display them with custom animations while preserving all their functionality.

## Features

- **Sliding Animations**: Notifications slide in from the right, "break" in the center with visual effects, then slide out to the left
- **Particle Effects**: Dynamic particle systems create visual "break" effects when notifications reach their display position
- **Urgency-Based Behavior**: Different animation speeds, durations, and effects based on notification urgency levels
- **Seamless Integration**: Works transparently with dunst, SwayNC, and other notification daemons
- **Action Support**: Preserves notification actions and interactive elements
- **Icon Display**: Supports notification icons with automatic scaling
- **Multi-Monitor**: Automatically adapts to monitor dimensions and stacks notifications appropriately

## Architecture

### Core Components

The plugin is structured with clear separation of concerns:

- **main.cpp**: Plugin entry point, initialization, and Hyprland API integration
- **notification.hpp/cpp**: Core notification rendering using Cairo and Pango
- **particle_system.hpp/cpp**: Particle effects system for visual "break" animations
- **animation_manager.hpp/cpp**: Manages multiple notifications and render loop integration
- **dbus_listener.hpp/cpp**: DBus integration for intercepting notification daemon messages
- **window_interceptor.hpp/cpp**: Hides original notification windows to prevent duplication
- **notification_data.hpp**: Data structures and enums for notification content and state

### Animation State Machine

Each notification follows a predictable state progression:

1. **SLIDING_IN**: Notification slides from off-screen right to center position
2. **BREAKING**: Visual "break" effect with particle explosion and screen shake
3. **SHOWING**: Static display period with urgency-based duration
4. **SLIDING_OUT**: Notification slides from center to off-screen left with fade
5. **FINISHED**: Cleanup and removal from animation manager

### Integration Strategy

The plugin operates by:

1. Registering as a DBus notification service on the session bus
2. Implementing the `org.freedesktop.Notifications` interface
3. Intercepting notification calls before they reach the native daemon
4. Hiding original notification windows through Hyprland's window management
5. Rendering custom animated notifications with particle effects
6. Processing notification actions and maintaining compatibility

## Prerequisites

### System Requirements

- Hyprland 0.50.0 or newer (commit: cb6589db98325705cef5dcaf92ccdf41ab21386d)
- Linux system with Wayland support
- Modern GPU with OpenGL support for optimal performance

### Dependencies

The plugin requires several development libraries:

#### Core Dependencies
- **hyprland**: Plugin API and compositor integration
- **cairo**: 2D graphics rendering
- **pango**: Text layout and font rendering
- **glib**: Core application building blocks
- **gio**: High-level I/O and DBus communication
- **pixman**: Low-level pixel manipulation
- **wayland**: Wayland protocol support

#### Build Dependencies
- **cmake**: Build system (version 3.16 or newer)
- **pkg-config**: Dependency management
- **gcc**: C++ compiler with C++23 support

#### Optional Dependencies
- **dunst**: Notification daemon for integration testing
- **swaync**: Alternative notification daemon
- **libnotify**: For testing with notify-send

### NixOS Users

For NixOS environments, use the provided flake or add to your configuration:

```nix
environment.systemPackages = with pkgs; [
  hyprland cairo pango glib libnotify pkg-config gcc cmake
];
```

## Building

### Standard Build Process

The plugin uses CMake with pkg-config for dependency resolution:

```bash
# Configure build
cmake -B build -S .

# Compile
cmake --build build

# Install to Hyprland plugins directory
mkdir -p ~/.config/hypr/plugins/
cp build/plugin.so ~/.config/hypr/plugins/notification-plugin.so
```

### NixOS Build

For NixOS users with the provided flake:

```bash
# Enter development shell
nix develop

# Build using CMake
cmake -B build -S .
cmake --build build

# Or build the package directly
nix build
```

### Build System Details

The plugin uses a modern CMake configuration that:

- Automatically discovers dependencies using pkg-config
- Compiles with C++23 standard for modern language features
- Links against all required libraries dynamically
- Includes proper compiler flags for position-independent code
- Supports both debug and release build configurations

## Installation

### Loading the Plugin

Add the plugin to your Hyprland configuration:

```bash
# ~/.config/hypr/hyprland.conf
plugin = ~/.config/hypr/plugins/notification-plugin.so
```

### Restart Hyprland

Restart Hyprland or reload the configuration:

```bash
hyprctl reload
```

### Verify Installation

Check that the plugin loaded successfully:

```bash
hyprctl plugin list
```

## Configuration

### Notification Daemon Integration

The plugin works best when configured with your existing notification daemon:

#### Dunst Integration

Configure dunst to work transparently with the plugin by creating or modifying `~/.config/dunst/dunstrc`:

```ini
[global]
    # Let plugin handle visual display
    show_indicators = false
    indicate_hidden = false
    notification_limit = 0
    
    # Positioning handled by plugin
    geometry = "0x0-0+0"
    transparency = 100
    
    # Keep DBus integration active
    startup_notification = true

[urgency_low]
    timeout = 2

[urgency_normal]
    timeout = 4

[urgency_critical]
    timeout = 0
```

#### SwayNC Integration

For SwayNC users, configure in `~/.config/swaync/config.json`:

```json
{
  "notification-window-width": 0,
  "widgets": [],
  "fit-to-screen": true,
  "timeout": 4,
  "timeout-low": 2,
  "timeout-critical": 0
}
```

### Plugin Configuration

The plugin accepts configuration commands through Hyprland dispatchers:

```bash
# Enable specific daemon integration
hyprctl dispatch notification_config enable_dunst
hyprctl dispatch notification_config enable_swaync

# Disable window interception if needed
hyprctl dispatch notification_config disable_interception
```

## Usage

### Basic Testing

Test the plugin with manual notifications:

```bash
# Basic notification
hyprctl dispatch notification "Hello World"

# Test with notify-send
notify-send "Test Title" "Test message body"

# Test urgency levels
notify-send -u low "Low priority"
notify-send -u normal "Normal priority" 
notify-send -u critical "Critical alert"
```

### Advanced Usage

The plugin automatically handles:

- **Multiple notifications**: Stacks vertically with appropriate spacing
- **Notification actions**: Preserves clickable actions from original notifications
- **Icons**: Displays application icons when provided
- **Rich content**: Supports markup in notification body text
- **Urgency handling**: Adjusts animation speed and duration based on urgency

### Debugging

Monitor plugin operation:

```bash
# Check plugin status
hyprctl plugin list

# View Hyprland logs
journalctl -u hyprland --since "5 minutes ago"

# Test plugin symbols
nm ~/.config/hypr/plugins/notification-plugin.so | grep plugin

# Check dependencies
ldd ~/.config/hypr/plugins/notification-plugin.so
```

## Development

### Project Structure

```
notification-hyrpland/
├── src/                          # Source code
│   ├── main.cpp                  # Plugin entry point
│   ├── notification.hpp/cpp      # Core notification rendering
│   ├── particle_system.hpp/cpp   # Particle effects
│   ├── animation_manager.hpp/cpp # Animation coordination
│   ├── dbus_listener.hpp/cpp     # DBus integration
│   ├── window_interceptor.hpp/cpp# Window management
│   └── notification_data.hpp     # Data structures
├── config/                       # Integration configurations
│   ├── dunst_integration.conf    # Dunst setup
│   └── swaync_integration.conf   # SwayNC setup
├── CMakeLists.txt                # Build configuration
├── flake.nix                     # Nix build environment
└── README.md                     # This file
```

### Code Conventions

The project follows modern C++ practices:

- **C++23 Standard**: Utilizes modern language features
- **RAII**: Automatic resource management with smart pointers
- **Const Correctness**: Immutable data where possible
- **Exception Safety**: Proper error handling and cleanup
- **Header Organization**: Clear separation of interface and implementation

### Build System

Uses CMake with pkg-config for:

- **Dependency Discovery**: Automatic library detection
- **Cross-Platform Support**: Works across different Linux distributions
- **Version Management**: Ensures compatibility with required library versions
- **Installation Handling**: Proper plugin installation procedures

### Plugin API Integration

Follows Hyprland plugin best practices:

- **Proper Entry Points**: Uses required camelCase function names
- **Version Checking**: Validates API compatibility at runtime
- **Event Handling**: Registers for necessary Hyprland events
- **Resource Cleanup**: Proper plugin shutdown procedures

## Performance

### Optimization Features

The plugin is designed for minimal performance impact:

- **Hardware Acceleration**: Uses GPU-accelerated rendering where available
- **Efficient Particle Systems**: Bounded particle counts prevent memory bloat
- **Render Loop Integration**: Syncs with Hyprland's existing render cycle
- **Memory Management**: RAII ensures no memory leaks
- **Event-Driven**: Only processes when notifications are active

### Resource Usage

Typical resource consumption:

- **CPU**: Minimal during idle, brief spikes during animations
- **Memory**: 5-15MB depending on active notification count
- **GPU**: Negligible impact on modern hardware
- **Network**: None (local DBus communication only)

## Troubleshooting

### Common Issues

#### Plugin Fails to Load

Check symbol exports and dependencies:

```bash
nm plugin.so | grep plugin
ldd plugin.so
```

Ensure correct function naming (camelCase) and complete dependencies.

#### Notifications Not Appearing

Verify notification daemon integration:

```bash
# Test basic notification
notify-send "Test"

# Check DBus service
dbus-send --session --type=method_call --print-reply \
  --dest=org.freedesktop.Notifications \
  /org/freedesktop/Notifications \
  org.freedesktop.Notifications.GetServerInformation
```

#### Performance Issues

Monitor resource usage:

```bash
# Check CPU usage
top -p $(pgrep hyprland)

# Monitor memory
ps aux | grep hyprland

# GPU usage (if available)
nvidia-smi  # or appropriate GPU monitoring tool
```

### Build Issues

#### Header Not Found

Ensure proper header paths for NixOS:

```cpp
// Correct for Nix environments
#include <hyprland/src/plugins/PluginAPI.hpp>

// Not: #include <src/plugins/PluginAPI.hpp>
```

#### CMake Configuration Errors

Verify pkg-config can find dependencies:

```bash
pkg-config --exists hyprland cairo pango glib-2.0
pkg-config --modversion hyprland
```

#### Version Compatibility

Ensure exact version matching between build headers and runtime:

```bash
# Check runtime version
hyprctl version

# Verify build environment matches
```

## Contributing

### Development Setup

1. Fork the repository
2. Set up development environment with required dependencies
3. Create feature branch from main
4. Implement changes following project conventions
5. Test thoroughly with multiple notification scenarios
6. Submit pull request with detailed description

### Testing Guidelines

Before submitting changes:

- Test basic notification display
- Verify urgency level handling
- Check multiple notification stacking
- Test notification daemon integration
- Verify no memory leaks with valgrind
- Ensure proper plugin loading/unloading

### Code Quality

The project maintains high code quality through:

- **Static Analysis**: Regular linting and code analysis
- **Memory Safety**: Valgrind testing for leak detection
- **Integration Testing**: Verification with multiple notification daemons
- **Performance Testing**: Benchmarking animation performance
- **Documentation**: Comprehensive code documentation

## License

This project is licensed under the MIT License. See the LICENSE file for full terms.

## Acknowledgments

- Hyprland development team for the excellent plugin API
- Cairo and Pango communities for robust graphics libraries
- Notification daemon developers for standardized DBus interfaces
- NixOS community for packaging and distribution support

## Related Projects

- **Hyprland**: The compositor this plugin extends
- **dunst**: Lightweight notification daemon
- **SwayNC**: Notification center for Sway/Hyprland
- **libnotify**: Standard notification library

For more information about Hyprland plugin development, see the comprehensive plugin development guide included with this project.
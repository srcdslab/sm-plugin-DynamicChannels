# SourcePawn Plugin Development Guidelines - Dynamic Channels

## Repository Overview
This repository contains a SourcePawn plugin for SourceMod called "Dynamic Game_Text Channels". The plugin provides automatic game_text channel assignment to prevent conflicts between map-used channels and plugin-used channels. It works by monitoring game_text entities created by maps and dynamically assigning available channels to plugins through a native function API.

**Key Components:**
- `DynamicChannels.sp` - Main plugin that manages channel assignments
- `DynamicChannels.inc` - Native function interface for other plugins
- `ExamplePlugin.sp` - Demonstrates proper usage of the DynamicChannels API

## Technical Environment
- **Language**: SourcePawn
- **Platform**: SourceMod 1.12+ (latest stable release)
- **Compiler**: SourcePawn compiler (spcomp) via native GitHub Actions CI
- **Build Tools**: GitHub Actions workflow (configured in `.github/workflows/ci.yml`)
- **Target Games**: Source engine games (tested on CS:GO)
- **Dependencies**: SourceMod SDK, SDKHooks, DHooks extensions

## Code Style & Standards
- **Indentation**: Use tabs equivalent to 4 spaces
- **Variables**: camelCase for local variables and function parameters
- **Functions**: PascalCase for function names and global variables
- **Global Variables**: Prefix with "g_" (e.g., `g_iMaxEntities`)
- **Constants**: Use descriptive names in UPPER_CASE
- **Spacing**: Delete trailing spaces
- **Required Pragmas**: Always include `#pragma semicolon 1` and `#pragma newdecls required`

## Documentation Requirements
- **Plugin Headers**: Use minimal plugin info structure (name, author, description, version, url)
- **Native Functions**: Document all natives in .inc files with:
  - Function description
  - Parameter descriptions with types and ranges
  - Return value description
  - Usage examples when complex
- **Complex Logic**: Add comments for non-obvious algorithms (e.g., channel assignment logic)
- **No Header Comments**: Avoid unnecessary file headers or excessive commenting

## Best Practices

### Memory Management
- Use `delete` for cleanup without null checks (SourceMod handles null automatically)
- Use `delete` instead of `.Clear()` for StringMap/ArrayList to prevent memory leaks
- Always recreate containers after deletion rather than clearing them
- Properly close handles with `CloseHandle()` or `delete`

### Data Structures
- Prefer StringMap/ArrayList over fixed arrays for dynamic data
- Use methodmaps for native function organization
- Implement proper bounds checking for array access

### Database Operations
- **All SQL queries must be asynchronous** using methodmap approach
- Use transactions for multiple related queries
- Always escape strings and validate input to prevent SQL injection
- Implement proper error handling for database operations

### Event-Driven Programming
- Follow SourceMod's event-based model
- Implement `OnPluginStart()` for initialization
- Implement `OnPluginEnd()` for cleanup when necessary
- Use proper event hooks and timers efficiently

### Performance Optimization
- Minimize timer usage where possible
- Cache expensive operation results
- Avoid unnecessary loops and string operations in frequently called functions
- Consider server tick rate impact when designing algorithms
- Aim to improve complexity from O(n) to O(1) where possible

## Project Structure
```
├── scripting/
│   ├── DynamicChannels.sp          # Main plugin
│   ├── ExamplePlugin.sp            # Usage example
│   └── include/
│       └── DynamicChannels.inc     # Native definitions
├── .github/
│   ├── workflows/ci.yml            # Build automation
│   └── copilot-instructions.md     # This file
└── README.md                      # Plugin documentation
```

## DynamicChannels-Specific Architecture

### Core Functionality
- **Channel Monitoring**: Hooks game_text entity creation and property changes
- **Dynamic Assignment**: Assigns available channels (0-5) to plugin groups
- **Conflict Prevention**: Ensures plugins don't use map-reserved channels
- **Overflow Handling**: Manages scenarios where channel demand exceeds supply

### Key Globals
- `g_bMapChannels[6]` - Tracks which channels are used by the map
- `g_iGroupChannels[6]` - Stores current plugin group to channel assignments
- `g_bChannelsOverflowing` - Indicates when all channels are in use
- `g_bBadMapChannels` - Flags maps using invalid channel numbers (>5 or <0)

### DHooks Implementation
The plugin uses DHooks to monitor `AcceptInput` calls on game_text entities:
- Watches for "AddOutput" inputs that change channel properties
- Updates channel tracking when map entities modify their channels
- Recalculates plugin group assignments when map channels change

### Native Function Design
```sourcepawn
native int GetDynamicChannel(int group);
```
- Groups 0-5 correspond to different plugin "slots"
- Returns consistent channel number for the same group during a session
- Automatically reassigns if map channel usage changes
- Throws native error for invalid group numbers

## Build & Validation Process

### Building
1. **GitHub Actions CI**: Push or open a PR to trigger the `.github/workflows/ci.yml` build
2. **Manual Compilation**: `spcomp DynamicChannels.sp -iinclude/`
3. **Output**: Generates `.smx` files in the plugins directory

### Validation Steps
1. **Syntax Check**: Ensure code compiles without errors or warnings
2. **Testing Environment**: Test on a development server with various maps
3. **Channel Validation**: Use `sm_debugchannels` command to verify assignments
4. **Memory Profiling**: Check for leaks using SourceMod's built-in profiler
5. **Multi-Plugin Testing**: Test with multiple plugins using GetDynamicChannel()

### Testing Scenarios
- Maps with no game_text entities
- Maps using various channel combinations (0-5)
- Maps with invalid channels (>5 or <0)
- Multiple plugins requesting different groups simultaneously
- Channel overflow scenarios (>6 total channels needed)

## Version Control
- **Versioning**: Use semantic versioning (MAJOR.MINOR.PATCH)
- **Plugin Version**: Keep plugin version in sync with repository tags
- **Commit Messages**: Clearly describe functionality changes
- **Branching**: Use feature branches for new functionality

## Error Handling & Debugging
- **Native Errors**: Use `ThrowNativeError()` for invalid parameters
- **Logging**: Use `LogError()` for critical issues, `LogMessage()` for warnings
- **Admin Notifications**: Implement in-game warnings for root admins when appropriate
- **Debug Commands**: Provide admin commands for troubleshooting (like `sm_debugchannels`)

## Configuration Management
- **ConVars**: Use descriptive names with plugin prefix (e.g., `sm_dynamic_channels_warnings`)
- **Auto Config**: Use `AutoExecConfig()` for automatic config file generation
- **Default Values**: Choose sensible defaults that work for most servers

## Extension Dependencies
- **SDKHooks**: Used for entity monitoring and property access
- **DHooks**: Required for hooking game functions (AcceptInput)
- **GameData**: Relies on sdktools.games for function signatures

## Common Pitfalls to Avoid
1. **Long-term Storage**: Don't store channel numbers long-term; always call GetDynamicChannel() when needed
2. **Direct Channel Usage**: Never use hardcoded channels; always use GetDynamicChannel()
3. **Synchronous DB**: All database operations must be asynchronous
4. **Memory Leaks**: Don't use .Clear() on containers; use delete and recreate
5. **Entity Validation**: Always validate entity indices before accessing properties

## Future Development Considerations
- Maintain backward compatibility with the GetDynamicChannel() native
- Consider impact on server performance when adding features
- Ensure new features work across different Source engine games
- Document any changes to the native interface thoroughly
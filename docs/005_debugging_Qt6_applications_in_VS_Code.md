# Debugging Qt6 Applications in VS Code

## Prerequisites
- Qt6 SDK installed (with debug symbols/libraries)
- CMake (recommended for Qt6 projects)
- VS Code with extensions: **C/C++** (ms-vscode.cpptools), **CMake Tools**, optionally **Qt tools** (e.g., tonka3000.qtvsctools)
- Compiler with debugger: GCC/GDB (Linux), MinGW/GDB (Windows), or MSVC/cppvsdbg (Windows), LLDB (macOS)

## 1. Project Setup with CMake

`CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.16)
project(MyQtApp)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

add_executable(MyQtApp main.cpp mainwindow.cpp mainwindow.ui)
target_link_libraries(MyQtApp PRIVATE Qt6::Widgets)
```

Important: build in **Debug** mode (`CMAKE_BUILD_TYPE=Debug`) — Release builds strip symbols and optimize away variables, breaking step-through debugging.

## 2. Configure CMake Tools

In `.vscode/settings.json`:
```json
{
  "cmake.configureSettings": {
    "CMAKE_PREFIX_PATH": "/path/to/Qt/6.x.x/gcc_64",
    "CMAKE_BUILD_TYPE": "Debug"
  },
  "cmake.buildDirectory": "${workspaceFolder}/build"
}
```

Select the kit (compiler) via Command Palette: **CMake: Select a Kit**. Choose the build variant: **CMake: Select Variant** → Debug.

## 3. launch.json Configuration

### Linux/macOS (GDB/LLDB)
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Qt6 App (GDB)",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/MyQtApp",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}/build",
      "environment": [
        {
          "name": "QT_QPA_PLATFORM",
          "value": "xcb"
        },
        {
          "name": "QT_DEBUG_PLUGINS",
          "value": "1"
        }
      ],
      "externalConsole": false,
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "preLaunchTask": "CMake: build",
      "setupCommands": [
        {
          "description": "Enable pretty-printing for gdb",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ]
    }
  ]
}
```

### Windows (MSVC)
```json
{
  "name": "Debug Qt6 App (MSVC)",
  "type": "cppvsdbg",
  "request": "launch",
  "program": "${workspaceFolder}/build/Debug/MyQtApp.exe",
  "args": [],
  "cwd": "${workspaceFolder}/build/Debug",
  "environment": [
    { "name": "PATH", "value": "C:\\Qt\\6.x.x\\msvc2019_64\\bin;${env:PATH}" }
  ],
  "stopAtEntry": false,
  "preLaunchTask": "CMake: build"
}
```

### Windows (MinGW/GDB)
Same as Linux config, but set `"miDebuggerPath": "C:\\Qt\\Tools\\mingw1120_64\\bin\\gdb.exe"` and ensure Qt's `bin` directory (containing the DLLs) is in the `PATH` environment variable in `launch.json`.

## 4. tasks.json (Build Task)
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "CMake: build",
      "type": "shell",
      "command": "cmake --build ${workspaceFolder}/build",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    }
  ]
}
```

## 5. Debugging Workflow

1. Set breakpoints by clicking left of line numbers (red dot).
2. Press **F5** or use Run and Debug panel, select your configuration.
3. Use the debug toolbar: Continue (F5), Step Over (F10), Step Into (F11), Step Out (Shift+F11), Restart, Stop.
4. Inspect variables in the **Variables** pane, hover over variables in code, or add to **Watch**.
5. Use the **Debug Console** to evaluate expressions (GDB/LLDB syntax for `cppdbg`, e.g., `-exec print myVar`).

## 6. Qt-Specific Debugging Tips

**Pretty-printers for Qt types**: GDB doesn't natively format `QString`, `QList`, etc. nicely. Solutions:
- Use Qt's bundled GDB pretty-printers (found in `Qt/6.x.x/gcc_64/lib/` or via `qtcreatorcdbext` for MSVC).
- Add to `setupCommands` in `launch.json`:
```json
{
  "description": "Load Qt pretty printers",
  "text": "source /path/to/qt5printers.py",
  "ignoreFailures": true
}
```

**Signal/Slot debugging**: Set breakpoints inside slot implementations directly; stepping through `connect()`/signal emission internals is usually not useful — focus on slot bodies.

**QML debugging**: Requires separate setup — add `-qmljsdebugger=port:1234` to `args`, and use the `qml` debug type with a separate launch config, or use Qt Creator's QML profiler instead since VS Code's QML support is limited.

**Plugin loading issues**: Set `QT_DEBUG_PLUGINS=1` env var to diagnose missing platform plugins (common cause of "could not load xcb platform plugin" errors).

**Windows DLL not found errors**: Ensure Qt's `bin` folder is on `PATH` in the debug environment, or copy required DLLs (`Qt6Core.dll`, `Qt6Gui.dll`, `Qt6Widgets.dll`, platform plugins from `plugins/platforms/`) next to the executable.

## 7. Common Issues

| Issue | Fix |
|---|---|
| Breakpoints not hit | Confirm Debug build, check `program` path matches actual binary |
| "No symbol table" | Rebuild in Debug mode, verify `CMAKE_BUILD_TYPE` |
| Qt types show raw memory | Install/load pretty-printers |
| App crashes on plugin load | Set `QT_QPA_PLATFORM` and `QT_DEBUG_PLUGINS=1` |
| MOC-generated code errors | Run `CMake: Clean Reconfigure` and rebuild |

## 8. Optional: CodeLLDB for macOS
Install **CodeLLDB** extension and use `"type": "lldb"` instead of `cppdbg` for better LLDB integration on macOS.
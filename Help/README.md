# Hello-IDA-Plugins by Luigi Origa
# Help - Hello IDA Plugin Manager

Help is the central configuration plugin for the **Hello IDA** plugin suite. It provides a GUI to enable/disable, configure shortcuts, and manage menu entries for all other Hello IDA plugins installed in the repository.

**Shortcut:** `Ctrl-Shift-H`

## What It Does

Each Hello IDA plugin registers its actions (menu items, shortcuts, functions) through a shared configuration system (`TPluginManager`). The Help plugin reads these configurations and presents a unified interface where you can:

- **Enable / Disable** individual plugins or actions
- **Change keyboard shortcuts** for any action
- **Show / Hide menu entries** under the `Edit > Hello` menu
- **View plugin versions** and descriptions

Changes take effect immediately — no need to restart IDA.

## Managed Plugins

| Plugin | Description |
|--------|-------------|
| **BeepBeep** | Audio notification plugin |
| **ColQ** | Database query engine with SQL-like syntax |
| **SearchBinary** | Binary pattern search |
| **SearchTables** | Table structure search |
| **SearchXOR** | XOR-encoded data search |
| **SimTricore** | TriCore instruction simulator |

## Configuration

Each plugin exposes these configurable fields per action:

| Setting | Description |
|---------|-------------|
| Action Enabled | Enable/disable the action entirely |
| Menu | Menu path (e.g. `Edit/Hello/ColQ/`) |
| Menu Enabled | Show/hide from the IDA menu |
| Shortcut | Keyboard shortcut (e.g. `Ctrl-Shift-Q`) |
| Shortcut Enabled | Activate/deactivate the shortcut |

## Requirements

- IDA Pro 9.0+ (64-bit)

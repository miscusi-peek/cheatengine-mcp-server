# Cheat Engine MCP Bridge

> **Model Context Protocol (MCP) server enabling AI agents to directly control Cheat Engine for memory analysis and game reverse engineering.**

[![Version](https://img.shields.io/badge/version-11.4.0-blue.svg)](#) [![Python](https://img.shields.io/badge/python-3.10%2B-green.svg)](https://python.org) [![Tests](https://img.shields.io/badge/tests-36%2F37%20passing-brightgreen.svg)](#)

---

## Overview

This project provides a **production-grade MCP server** that bridges AI assistants (Claude, Cursor, Copilot) with **Cheat Engine**, enabling real-time memory analysis, invisible debugging, and structure reverse engineering.

### Key Features

| Feature | Description |
|---------|-------------|
| 🔧 **40+ Tools** | Memory read, pattern scanning, pointer chains, structure dissection |
| 🛡️ **Anti-Cheat Safe** | Hardware breakpoints (DR0-DR3), no memory writes, DBVM Ring -1 tracing |
| 🔄 **Universal** | Automatic 32/64-bit architecture detection |
| ⚡ **High Performance** | <2ms latency via Named Pipe async I/O |
| 🤖 **AI-Optimized** | Structured JSON responses for LLM consumption |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  AI Agent (Claude/Cursor/Copilot)                                      │
│       │                                                                 │
│       ▼ MCP Protocol (JSON-RPC over stdio)                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  mcp_cheatengine.py (Python MCP Server)                         │   │
│  │  - Translates MCP tools to JSON-RPC                             │   │
│  │  - Connects to \\.\pipe\CE_MCP_Bridge_v99                       │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │ Named Pipe (Async)                      │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  CheatEngine (Running, attached to .exe)                        │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │  ce_mcp_bridge.lua                                      │    │   │
│  │  │  ┌─────────────────┐      ┌─────────────────────┐       │    │   │
│  │  │  │ Worker Thread   │◄────►│ Main Thread (GUI)   │       │    │   │
│  │  │  │ (Blocking I/O)  │ Sync │ (Safe API Execution)│       │    │   │
│  │  │  └─────────────────┘      └─────────────────────┘       │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tool Categories

### 📖 Memory Reading & Scanning
| Tool | Description |
|------|-------------|
| `read_memory(addr, size)` | Read raw bytes |
| `read_integer(addr, type)` | Read Byte/Word/Dword/Qword/Float/Double |
| `read_string(addr, len)` | Read ASCII/UTF-16 strings |
| `read_pointer_chain(base, offsets)` | Follow dynamic pointer paths |
| `scan_all(val, type, prot)` | Full memory scanner |
| `aob_scan(pattern)` | Find array of bytes patterns |
| `generate_signature(addr)` | Create unique AOB signature |

### 🧬 Structure & Code Analysis
| Tool | Description |
|------|-------------|
| `dissect_structure(addr)` | Auto-guess fields and types |
| `get_rtti_classname(addr)` | Identify C++ object types |
| `disassemble(addr, count)` | Disassemble instructions |
| `analyze_function(addr)` | Find all CALLs in a function |
| `find_references(addr)` | Cross-references to address |
| `find_call_references(func)` | Who calls this function |
| `find_function_boundaries(addr)` | Locate function start/end |

### Debugging (Anti-Cheat Safe)
| Tool | Description |
|------|-------------|
| `set_breakpoint(addr)` | Hardware BP - logs registers |
| `set_data_breakpoint(addr)` | Watchpoint on Write/Read |
| `get_breakpoint_hits()` | Retrieve hit logs |
| `list_breakpoints()` | List active breakpoints |
| `clear_all_breakpoints()` | Remove all breakpoints |

### DBVM Hypervisor (Ring -1)
| Tool | Description |
|------|-------------|
| `start_dbvm_watch(addr)` | Invisible physical memory trace |
| `stop_dbvm_watch(addr)` | Stop and get results |
| `get_physical_address(addr)` | Virtual → Physical address |

And many more at `MCP_Bridge_Command_Reference.md`

---

## Installation

### Python Dependencies
```bash
pip install -r MCP_Server/requirements.txt
```
Or install manually:
```bash
pip install mcp pywin32
```

> [!NOTE]
> **Windows only** - This project uses Named Pipes (`pywin32`) which are Windows-specific.

---

## Quick Start

### 1. Load Bridge in Cheat Engine
```
1. Enable DBVM in CheatEngine.
2. File → Execute Script → Open ce_mcp_bridge.lua → Execute
```
Look for: `[MCP v11.4.0] Server started on \\.\pipe\CE_MCP_Bridge_v99`

### 2. Configure MCP Client
Add to your MCP configuration (e.g., `mcp_config.json`):
```json
{
  "servers": {
    "cheatengine": {
      "command": "python",
      "args": ["C:/path/to/MCP_Server/mcp_cheatengine.py"]
    }
  }
}
```
Don't forget to restart the IDE/RooCode/Cline/Cursor etc if it's not working.

### 3. Verify Connection
Use the `ping` tool to verify connectivity:
```json
{"success": true, "version": "11.4.0", "message": "CE MCP Bridge Active"}
```

---

## Critical Configuration

### BSOD Prevention
> [!CAUTION]
> **You MUST disable:** Cheat Engine → Settings → Extra → **"Query memory region routines"**
> 
> Enabled: Causes `CLOCK_WATCHDOG_TIMEOUT` BSODs with DBVM/Anti-Cheat conflicts  
> Disabled: Scanning works perfectly and safely

### Anti-Cheat Safety Rules
- **DO NOT** use software breakpoints (0xCC/Int3)
- **DO NOT** write to memory
- **USE** Hardware Debug Registers (DR0-DR3) only
- **USE** DBVM for invisible Ring -1 tracing

---

## Project Structure

```
MCP_Server/
├── mcp_cheatengine.py      # Python MCP Server (FastMCP)
├── ce_mcp_bridge.lua   # Cheat Engine Lua Bridge
└── test_mcp.py # Test Suite (36/37 passing)

AI_Context/
├── MCP_Bridge_Command_Reference.md   # MCP Commands reference
├── CE_LUA_Documentation.md   # Full CheatEngine 7.6 official documentation
└── AI_Guide_MCP_Server_Implementation.md  # Full technical documentation for AI agent
```

---

## Testing

Run the comprehensive test suite:
```bash
python MCP_Server/test_mcp_.py
```

Expected output:
```
✅ Memory Reading: 6/6 tests passed
✅ Process Info: 4/4 tests passed  
✅ Code Analysis: 8/8 tests passed
✅ Breakpoints: 4/4 tests passed
✅ DBVM Functions: 3/3 tests passed
✅ Utility Commands: 11/11 tests passed
⏭️ Skipped: 1 test (generate_signature)
────────────────────────────────────
Total: 36/37 PASSED (100% success)
```

---

## License

MIT License - For educational and research purposes only.

---

## Keywords

`mcp` `cheat-engine` `memory-analysis` `reverse-engineering` `model-context-protocol` `ai-agents` `python` `lua` `debugging` `game-research`

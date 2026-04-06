# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Optimus Coding Tool** - a minimal Python implementation of Claude Code that supports multiple AI providers including local Ollama models. It provides an AI coding assistant with tool execution capabilities in a terminal interface.

## Running the Tool

To start the interactive coding tool:
```bash
python coding_tool.py
```

For non-interactive execution with a prompt:
```bash
python coding_tool.py "your prompt here"
```

For one-shot execution with specific options:
```bash
python coding_tool.py --print --model gpt-4o "your prompt"
python coding_tool.py --accept-all --verbose "your prompt"
```

## Configuration

The tool loads configuration from:
- `~/.optimus/config.json` - User configuration
- Environment variables for API keys (e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`)
- Command line arguments override config settings

Use slash commands to interactively configure:
- `/model [model]` - Show or set model
- `/config key=value` - Set configuration
- `/permissions [mode]` - Set permission mode (auto/accept-all/manual)

## Architecture

### Core Components

1. **coding_tool.py** - Main entry point and REPL
   - Handles user input, slash commands
   - Manages console rendering with Rich markdown support
   - Provides interactive prompts and permission requests

2. **agent.py** - Multi-turn agent loop
   - Manages conversation state
   - Handles tool execution flow
   - Yields streaming events (TextChunk, ThinkingChunk, ToolStart, etc.)

3. **providers.py** - Multi-provider abstraction
   - Supports: Anthropic, OpenAI, Gemini, Kimi, Qwen, Zhipu, DeepSeek, Ollama, LM Studio
   - Auto-detects provider from model string
   - Converts between neutral message formats and provider-specific formats

4. **tools.py** - Tool definitions and execution
   - Built-in tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch, EmailSend
   - Permission gating for write operations
   - Safe bash command filtering

5. **config.py** - Configuration management
   - Loads/saves config to `~/.optimus/config.json`
   - Handles API key resolution (config → env → hardcoded)

6. **context.py** - System prompt building
   - Injects git status and recent commits
   - Loads CLAUDE.md from project and global locations

### Data Flow

1. User input → agent.run()
2. Agent calls providers.stream() with neutral messages
3. Provider returns streaming events
4. Tool calls executed via execute_tool()
5. Tool results appended to conversation history

### Key Design Patterns

- **Neutral message format** throughout the system (provider-independent)
- **Streaming architecture** with event types
- **Permission modes** for safety (auto/accept-all/manual)
- **Session persistence** (save/load conversation history)

## Multi-Provider Support

Model string formats:
- `"claude-opus-4-6"` → auto-detected → Anthropic
- `"gpt-4o"` → auto-detected → OpenAI
- `"ollama/qwen2.5-coder"` → explicit Ollama provider
- `"custom/my-model"` → uses CUSTOM_BASE_URL from config

Use `/model` command to list available models by provider.

## Tool System

The tool supports 9 built-in tools:
- **Read** - Read file contents with line numbers
- **Write** - Write/create files (asks permission). **Default output path: `./test/` for ALL generated files � including code, scripts, Excel files, helper scripts, temporary files, etc. Never write generated files anywhere outside `./test/`.**
- **Edit** - Replace exact text in files (asks permission)
- **Bash** - Execute shell commands (filtered for safety)
- **Glob** - Find files by pattern
- **Grep** - Search file contents with regex
- **WebFetch** - Fetch and extract web content
- **WebSearch** - Search the web via DuckDuckGo
- **ExcelAutomate**: Automate Excel operations based on natural language requests
- **EmailSend** - Send email via SMTP2Go API (asks permission � only recipient required)

## Email Tool Integration

The system includes email sending capabilities through the `EmailSend` tool:

### EmailSend Tool
- **Description**: Send an email via the SMTP2Go API
- **Required Parameters**:
  - `to`: List of recipient email addresses � the **only mandatory field**
- **Optional Parameters** (auto-generated in a professional tone if not provided):
  - `subject`: Email subject line � if not specified, the model will generate a professional subject based on the context of the email or the user's request
  - `text_body`: Plain-text email body � if not specified, the model will draft a professional email body based on the user's description
  - `sender`: Sender email address � **defaults to `marva112@suchance.com`** unless user specifies another
  - `cc`: List of CC recipients
  - `bcc`: List of BCC recipients
  - `html_body`: HTML email body
- **API Configuration**:
  - `smtp_api_key` in `.optimus/config.json` (or `SMTP_API_KEY` env variable)
  - `smtp_base_url` in `.optimus/config.json` (default: `https://api.smtp2go.com/v3/`)
- **Examples**:
  ```bash
  # EmailSend to=["colleague@example.com"], subject="Meeting Reminder", text_body="Hi, this is a reminder about tomorrow's meeting at 10 AM."
  
  # Minimal � only recipient required; subject and body auto-generated professionally
  EmailSend to=["manager@example.com"]
  ```
- **Usage Notes**:
  - The model **MUST** ensure `to` is provided before calling the tool. If missing, ask the user.
  - If `subject` is not provided, the model will generate a professional, contextual subject line.
  - If `text_body` is not provided, the model will draft a professional email body based on the user's request.
  - `sender` defaults to `marva112@suchance.com`.
  - All auto-generated content uses a **professional tone**.
  - Email operations require user permission (same as Write/Edit).

Note: When using Write to save ANY user-requested output (code, scripts, data, helper files, etc.), always use the `./test/` directory. This includes temporary helper scripts � never place them in the main project directory.

## Development Notes

- Python 3.7+ required
- Dependencies: anthropic, openai, httpx, rich, pyreadline3
- Virtual environment recommended (.venv provided)
- No lint/test framework configured

## Excel Tool Integration

The system includes Excel automation capabilities through the `ExcelAutomate` tool:

### ExcelAutomate Tool
- **Description**: Automate Excel operations based on natural language requests
- **Capabilities**:
  - Read/write data to Excel files
  - Format headers and cells
  - Create charts and visualizations
  - Handle macros and complex operations
- **Parameters**:
  - `request`: Natural language description of Excel operation
  - `file_path`: Optional path to Excel file. If the file exists it is opened; if not, a new workbook is created and saved to this path. **Always saves data to the specified file_path.**
  - `sheet_name`: Optional sheet name. **Always defaults to "Sheet1"** unless user specifies otherwise.
  - `output_format`: Output format (text, json, file_path)
- **Examples**:
  ```bash
  # Create new workbook with oil price data in default Sheet1
  ExcelAutomate "Create new Excel file with 10 years of oil price data", file_path="./test/oil_prices.xlsx"
  
  # Write data to Sheet1 (default) and bold the header row
  ExcelAutomate "Write data: [['Year','Revenue'],[2020,100],[2021,150],[2022,200]], and format header bold", file_path="./test/pepsico_revenue.xlsx"
  
  # Create chart of oil prices in a specific sheet
  ExcelAutomate "Create a chart of oil prices over time", file_path="./test/oil_prices.xlsx", sheet_name="ChartSheet"
  ```
- **Dependencies**: Requires openpyxl (always). Excel COM support (Windows only) required for chart creation.
- **Usage Notes**:
  - For Windows systems only
  - openpyxl is always required; pywin32 is only needed for chart creation via COM
  - **Sheet name always defaults to "Sheet1"** � no longer "first sheet" or variable
  - **Data is always saved to file_path** when provided; if file does not exist, it is created
  - Parent directories are created automatically if missing
  - Use with caution as Excel automation can be resource-intensive

## Common Development Tasks

1. **Building the project**: 
   ```bash
   # No build system configured - check README for instructions
   ```

2. **Running tests**:
   ```bash
   # No test framework configured - check README for instructions
   ```

3. **Running a single test**:
   ```bash
   # No test framework configured - check README for instructions
   ```

4. **Setting up environment**:
   ```bash
   # Create virtual environment
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

5. **Running with specific model**:
   ```bash
   python coding_tool.py --model ollama/qwen2.5-coder
   ```

## Architecture Overview

The system follows a modular design with clear separation of concerns:
- **User Interface Layer**: Handles user input/output and slash commands
- **Agent Layer**: Manages conversation state and tool execution
- **Provider Layer**: Abstracts different AI services
- **Tool Layer**: Implements file and system operations

## Configuration Management

The configuration system supports:
- API key management
- Model selection
- Permission modes
- Verbose logging
- Thinking mode

## Security Considerations

- All file operations include permission gating
- Bash commands are filtered for safety
- No sensitive information (API keys) are stored in code
- Git status is injected into system prompts for context

## File Output / Code Generation Rules

- **ALL generated files of ANY type (code, scripts, Excel files, helper scripts, temporary files, etc.) MUST be saved in the `./test/` folder. This is a strict, non-negotiable rule.**
- **This includes:**
  - Generated source code files (e.g., `./test/script.py`)
  - Excel / data files (e.g., `./test/report.xlsx`)
  - Helper / utility scripts created for a one-time task (e.g., `./test/fix_excel.py`)
  - Temporary or intermediate files
- **NEVER** create generated files in the main project directory or any directory outside of `./test/`.
- **NEVER** mix user-requested generated code or data files with the main project code.
- The `./test/` folder should be created if it does not already exist.
- Examples:
  - If the user asks the AI to generate a Python script, save it as `./test/script_name.py`.
  - If the user asks the AI to generate an Excel file, save it as `./test/report.xlsx`.
  - If a temporary helper script is needed to process/format data, save it as `./test/helper_script.py` and delete it after use if appropriate.
- This rule applies even if the user does not explicitly mention a path. Always use `./test/` as the default output directory for ANY generated or created artifact.

## Version History

- 1.0.0 - Initial release
- 1.0.1 - Added multi-provider support
- 1.0.2 - Improved error handling

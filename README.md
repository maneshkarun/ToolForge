# ToolForge

ToolForge is a minimal local agent execution engine built with LangChain DeepAgents and LangGraph. It runs as an interactive command-line assistant that can work inside user-granted folders, execute a constrained set of shell commands, write files, fetch URLs, and search the web.

The project is intentionally small and easy to inspect:

- `main.py` starts the CLI, loads environment variables, creates the agent, prompts for folder access, and runs the task loop.
- `agent.py` configures the model provider and creates the DeepAgents agent.
- `tools.py` defines the sandboxed tools exposed to the agent.
- `requirements.txt` lists the Python dependencies.

## Features

- Interactive CLI task runner.
- Folder-based sandbox model.
- Shell command execution with an allowlist and extra blocked-pattern checks.
- Cross-platform translation for common Unix commands on Windows.
- File writing inside the granted workspace.
- URL fetching with HTML text extraction.
- DuckDuckGo web search through `ddgs`.
- Tool-call logging so executed actions are visible in the terminal.
- Environment variable loading from a local `.env` file.

## Project Structure

```text
ToolForge/
|-- agent.py
|-- main.py
|-- tools.py
|-- requirements.txt
|-- README.md
`-- .gitignore
```

## How It Works

1. `main.py` loads `.env` using `python-dotenv`.
2. It imports the agent factory and provider metadata from `agent.py`.
3. It wires a logger into `tools.py` so tool calls are printed in the terminal.
4. The CLI asks the user to grant access to a folder.
5. Granted folders are stored in `tools.ALLOWED_DIRECTORIES`.
6. User tasks are sent to the DeepAgents agent through `agent.ainvoke(...)`.
7. The agent can call the tools in `tools.py` to inspect files, write files, run allowed commands, fetch pages, and search the web.

## Agent Configuration

`agent.py` creates the model and passes the project tools into `create_deep_agent(...)`.

It reads these environment variables:

- `TOOLFORGE_PROVIDER`: model provider. Supported values are `anthropic`, `google`, `openai`, `ollama`, `openrouter`, and `custom`.
- `TOOLFORGE_MODEL`: provider-specific model name. Defaults to `claude-sonnet-4-6`.
- `TOOLFORGE_API_KEY`: API key for hosted providers.
- `TOOLFORGE_BASE_URL`: custom OpenAI-compatible endpoint URL.

If no provider is configured, ToolForge defaults to Anthropic with `claude-sonnet-4-6`.

Supported provider paths:

- Anthropic uses `langchain_anthropic.ChatAnthropic`.
- Google Gemini uses `langchain_google_genai.ChatGoogleGenerativeAI`.
- OpenAI uses `langchain_openai.ChatOpenAI`.
- Ollama uses `langchain_ollama.ChatOllama`, with a fallback to OpenAI-compatible `ChatOpenAI` at `http://localhost:11434/v1`.
- OpenRouter uses `langchain_openrouter.ChatOpenRouter`.
- Custom endpoints use OpenAI-compatible `ChatOpenAI` with `TOOLFORGE_BASE_URL`.

## Tools

### `run_shell`

Executes shell commands inside the first granted directory.

Allowed command families include:

- File operations: `ls`, `cat`, `cp`, `mv`, `rm`, `mkdir`, `rmdir`, `touch`
- Search: `find`, `grep`, `rg`, `fd`, `locate`
- Text processing: `sort`, `uniq`, `wc`, `cut`, `tr`, `sed`, `awk`, `xargs`
- Utilities: `echo`, `printf`, `date`, `stat`, `file`, `du`, `df`
- Scripting: `python`, `python3`, `bash`, `sh`
- Archives: `tar`, `zip`, `unzip`, `gzip`, `gunzip`, `bzip2`, `xz`
- Other: `jq`, `git`, `diff`, `patch`

The tool blocks patterns such as `sudo`, `su`, `curl`, `wget`, `ssh`, `chmod`, `chown`, process-kill commands, sensitive system paths, and simple path traversal attempts.

On Windows, common Unix-style commands are translated to Windows equivalents where possible.

### `write_file`

Writes text content to a relative path inside the granted workspace. It creates intermediate directories automatically and rejects absolute paths.

### `fetch_url`

Fetches an HTTP or HTTPS URL, removes noisy HTML elements, and returns extracted page text. This requires `beautifulsoup4`.

### `search_web`

Searches the web using DuckDuckGo through the `ddgs` package and returns the top results.

## Requirements

- Python 3.10 or newer is recommended.
- A supported model provider configured in `agent.py`.
- Network access if you want to use hosted model providers, URL fetching, or web search.

Python packages:

```text
deepagents>=0.6.1
httpx>=0.27.0
beautifulsoup4>=4.12.0
ddgs>=9.10.0
python-dotenv>=1.2.1
```

Optional provider packages used by `agent.py` include:

```text
langchain-anthropic
langchain-google-genai
langchain-openai>=1.0.0
langchain-ollama>=0.3.0
langchain-openrouter
```

Install the package that matches your configured provider.

## Getting Started

### 1. Clone the repository

```bash
git clone <repository-url>
cd ToolForge
```

If you already have the project locally, open a terminal in the project directory.

### 2. Create a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

macOS or Linux:

```bash
python -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

Install your provider integration if needed:

```bash
pip install langchain-anthropic
```

or:

```bash
pip install langchain-google-genai
```

or:

```bash
pip install langchain-openai
```

or:

```bash
pip install langchain-ollama
```

or:

```bash
pip install langchain-openrouter
```

### 4. Add environment variables

Create a `.env` file in the project root. The exact values depend on your `agent.py` implementation and chosen provider.

Example for Anthropic:

```env
TOOLFORGE_PROVIDER=anthropic
TOOLFORGE_MODEL=claude-sonnet-4-6
TOOLFORGE_API_KEY=sk-ant-your-key
```

Example for OpenAI:

```env
TOOLFORGE_PROVIDER=openai
TOOLFORGE_MODEL=gpt-4o
TOOLFORGE_API_KEY=sk-your-key
```

Example for Google Gemini:

```env
TOOLFORGE_PROVIDER=google
TOOLFORGE_MODEL=gemini-2.0-flash
TOOLFORGE_API_KEY=your-google-api-key
```

Example for local Ollama:

```env
TOOLFORGE_PROVIDER=ollama
TOOLFORGE_MODEL=qwen2.5:7b
TOOLFORGE_BASE_URL=http://localhost:11434
```

Example for OpenRouter:

```env
TOOLFORGE_PROVIDER=openrouter
TOOLFORGE_MODEL=openai/gpt-4o-mini
TOOLFORGE_API_KEY=your-openrouter-key
```

Example for any OpenAI-compatible endpoint:

```env
TOOLFORGE_PROVIDER=custom
TOOLFORGE_MODEL=Qwen/Qwen2.5-72B-Instruct
TOOLFORGE_BASE_URL=https://router.huggingface.co/v1
TOOLFORGE_API_KEY=hf_your_token
```

Do not commit `.env` files containing secrets.

### 5. Run the application

```bash
python main.py
```

When prompted, enter a folder path that the agent is allowed to access.

Example:

```text
Grant folder access (path): E:\Projects\MyWorkspace
```

After access is granted, type a task and press Enter.

## CLI Commands

Inside the running app:

- `quit` exits the program.
- `grant` grants access to another directory.
- `folders` lists all currently granted directories.

Any other input is treated as a task for the agent.

## Example Tasks

```text
List the files in this folder and summarize the project.
```

```text
Create a README draft for this package based on the source files.
```

```text
Search for TODO comments and suggest cleanup tasks.
```

```text
Fetch https://example.com and summarize the page.
```

## Sandbox Model

ToolForge uses a simple folder-based sandbox:

- The agent cannot use tools until at least one directory is granted.
- Shell commands run with the first granted directory as the current working directory.
- `write_file` only accepts relative paths and blocks writes outside the granted directory.
- Shell commands are checked against an allowlist before execution.
- Absolute Unix-style paths in shell commands are checked to ensure they stay inside granted directories.

This is a practical development sandbox, not a hardened security boundary. Review and improve the sandbox before using ToolForge with untrusted prompts, sensitive files, or production credentials.

## Development Notes

- Keep tool behavior narrow and explicit.
- Prefer adding new tools over widening the shell allowlist.
- Keep provider-specific code in `agent.py`.
- Keep secrets in `.env`, not in source files.
- Use `rg` for fast code search when available.

## Troubleshooting

### Provider authentication fails

Check that `.env` exists, the API key is correct, and the provider package is installed.

### Provider import fails

Install the matching LangChain provider package for the configured `TOOLFORGE_PROVIDER`.

Examples:

```bash
pip install langchain-anthropic
pip install langchain-google-genai
pip install langchain-openai
pip install langchain-ollama
pip install langchain-openrouter
```

### Web search fails

Verify that `ddgs` is installed and that the environment has network access.

### URL fetching fails

Verify that `beautifulsoup4` is installed and that the URL starts with `http://` or `https://`.

### Commands are blocked

Check the allowlist and blocked patterns in `tools.py`. Some commands are intentionally unavailable.

## License

No license file is currently included. Add a license before distributing or accepting external contributions.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MarkItDown is a lightweight Python utility (by Microsoft's AutoGen Team) for converting diverse document formats to Markdown for use with LLMs and text analysis pipelines. It emphasizes preserving document structure (headings, lists, tables, links) while remaining token-efficient for downstream LLM consumption.

The project is a monorepo with four packages:

- `packages/markitdown` — core library (main development target)
- `packages/markitdown-mcp` — MCP server wrapping the library for Claude Desktop / AI agent integration
- `packages/markitdown-ocr` — OCR plugin using PyMuPDF + LLM vision for image text extraction
- `packages/markitdown-sample-plugin` — reference plugin implementation (converts RTF files)

## Directory Structure

```
markitdown/
├── Dockerfile                        # Root image: Python 3.13, ffmpeg, exiftool; runs markitdown CLI
├── .devcontainer/devcontainer.json   # Dev Container config (uses root Dockerfile + hatch feature)
├── .dockerignore
├── .pre-commit-config.yaml           # Black formatter hook
├── .github/
│   ├── workflows/tests.yml           # CI: runs hatch test on Python 3.10/3.11/3.12
│   └── workflows/pre-commit.yml
├── CODE_OF_CONDUCT.md                # Microsoft Open Source Code of Conduct
├── SECURITY.md                       # Vulnerability reporting (MSRC); do NOT file public GitHub issues
├── SUPPORT.md
├── LICENSE                           # MIT
└── packages/
    ├── markitdown/                   # Core library
    │   ├── pyproject.toml
    │   ├── src/markitdown/
    │   │   ├── __init__.py           # Public API surface
    │   │   ├── __main__.py           # CLI entry point
    │   │   ├── _markitdown.py        # MarkItDown class + plugin loading
    │   │   ├── _base_converter.py    # DocumentConverter + DocumentConverterResult
    │   │   ├── _stream_info.py       # StreamInfo dataclass
    │   │   ├── _exceptions.py        # Exception hierarchy
    │   │   ├── _uri_utils.py         # data: and file: URI helpers
    │   │   ├── converters/           # All built-in converters (one file each)
    │   │   └── converter_utils/      # Internal helpers (DOCX math, pre-processing)
    │   └── tests/
    │       ├── test_files/           # Binary fixtures (pdf, docx, xlsx, jpg, …)
    │       ├── _test_vectors.py      # Shared test data definitions
    │       ├── test_module_vectors.py / test_cli_vectors.py   # Integration tests
    │       ├── test_module_misc.py / test_cli_misc.py         # Unit / edge-case tests
    │       ├── test_pdf_*.py         # PDF-specific tests (memory, tables, MasterFormat)
    │       ├── test_cu_converter.py  # Azure Content Understanding tests
    │       └── test_docintel_html.py # Azure Document Intelligence tests
    ├── markitdown-mcp/               # MCP server package
    │   ├── Dockerfile                # Separate image for running as MCP server
    │   ├── pyproject.toml
    │   └── src/markitdown_mcp/__main__.py   # FastMCP server; exposes convert_to_markdown tool
    ├── markitdown-ocr/               # OCR plugin package
    │   ├── pyproject.toml
    │   └── src/markitdown_ocr/       # OCR converters for PDF, DOCX, PPTX, XLSX
    └── markitdown-sample-plugin/     # Reference plugin (RTF converter)
        ├── pyproject.toml
        └── src/markitdown_sample_plugin/_plugin.py
```

## Key Files and Their Roles

| File | Role |
|------|------|
| `packages/markitdown/src/markitdown/_markitdown.py` | Core `MarkItDown` class: converter registration, plugin loading, all `convert_*` entry points |
| `packages/markitdown/src/markitdown/_base_converter.py` | Abstract `DocumentConverter` base class and `DocumentConverterResult` |
| `packages/markitdown/src/markitdown/_stream_info.py` | `StreamInfo` frozen dataclass (mimetype, extension, charset, filename, local_path, url) |
| `packages/markitdown/src/markitdown/_exceptions.py` | Exception hierarchy exported as public API |
| `packages/markitdown/src/markitdown/__init__.py` | Public exports — only import from here in application code |
| `packages/markitdown/src/markitdown/converters/` | One file per format; all named `_<format>_converter.py` |
| `packages/markitdown/src/markitdown/converter_utils/` | Internal helpers (not public API): OMML→LaTeX math rendering, DOCX DOM pre-processing |
| `Dockerfile` | Root Docker image — entry point is the `markitdown` CLI |
| `packages/markitdown-mcp/Dockerfile` | MCP server image — entry point is `markitdown-mcp` |
| `CODE_OF_CONDUCT.md` | Microsoft Open Source Code of Conduct (links to external policy) |
| `SECURITY.md` | Do NOT file security issues publicly; use MSRC at https://msrc.microsoft.com/create-report |

## Development Setup and Workflows

### Prerequisites

- Python 3.10+ (CI tests 3.10, 3.11, 3.12; supports up to 3.13)
- `hatch` for building and testing

### Local Setup

```bash
pip install hatch

# Enter dev shell with all optional deps installed
cd packages/markitdown
hatch shell

# OR install editable for direct use
pip install -e 'packages/markitdown[all]'
```

### Running Tests

All test commands run from `packages/markitdown/`:

```bash
# Run the full test suite (targets Python 3.10, 3.11, 3.12)
hatch test

# Run a single test file
hatch test tests/test_module_misc.py

# Run a single test by name
hatch test -- -k "test_name"

# Run with verbose output
hatch test -- -v
```

### Type Checking

```bash
# From packages/markitdown/
hatch run types:check
```

### Formatting and Pre-commit

The project uses Black for formatting (enforced via pre-commit):

```bash
# Run pre-commit checks before submitting a PR
pre-commit run --all-files

# Or install the hook locally
pre-commit install
```

### Docker

**Root image** — runs the `markitdown` CLI:

```bash
# Build
docker build -t markitdown:latest .

# Use (piped input)
docker run --rm -i markitdown:latest < your-file.pdf > output.md

# With git support (optional, larger image)
docker build --build-arg INSTALL_GIT=true -t markitdown:git .

# Custom user
docker build --build-arg USERID=$(id -u) --build-arg GROUPID=$(id -g) -t markitdown:latest .
```

The root Dockerfile:
- Base: `python:3.13-slim-bullseye`
- Installs `ffmpeg` and `exiftool` as runtime deps
- Sets `EXIFTOOL_PATH` and `FFMPEG_PATH` env vars
- Installs `markitdown[all]` and `markitdown-sample-plugin`
- Default user: `nobody:nogroup`

**MCP server image** — runs the MCP server:

```bash
# From packages/markitdown-mcp/
docker build -t markitdown-mcp:latest .
docker run --rm -i markitdown-mcp:latest
# Or with HTTP transport:
docker run --rm -p 3001:3001 markitdown-mcp:latest --http
```

### Dev Container

The `.devcontainer/devcontainer.json` builds the root Dockerfile with `INSTALL_GIT=true` and adds the `hatch` feature. Open in VS Code Dev Containers or GitHub Codespaces, then run `hatch test` directly.

## Architecture

### Converter Pipeline

The core flow in `_markitdown.py`:

1. `MarkItDown.convert(path_or_uri)` is the main entry point
2. File type is detected via **Magika** (ML-based) to populate a `StreamInfo` object (extension, mimetype, charset)
3. All registered `DocumentConverter` subclasses are iterated in reverse registration order (later-registered = higher priority)
4. The first converter whose `accepts(stream, stream_info, **kwargs) -> bool` returns `True` performs the conversion
5. Returns `DocumentConverterResult` with a `.markdown` string property (`.text_content` is a soft-deprecated alias)

Priority constants (lower = tried first):

```python
PRIORITY_SPECIFIC_FILE_FORMAT = 0.0   # format-specific converters (default)
PRIORITY_GENERIC_FILE_FORMAT  = 10.0  # catch-all converters (plain text, HTML, ZIP)
```

### Conversion Entry Points

| Method | Use case |
|--------|----------|
| `convert(path_or_uri)` | Auto-detects local path vs. URI — most permissive |
| `convert_local(path)` | Local files only |
| `convert_stream(stream, stream_info)` | Raw binary streams |
| `convert_response(response)` | `requests.Response` objects |
| `convert_uri(uri)` | Explicit URI handling (`data:`, `file:`, `http:`, `https:`) |

Prefer the narrowest method for your use case (security principle).

### Converter Contract

Every converter in `src/markitdown/converters/` implements:

```python
class MyConverter(DocumentConverter):
    def accepts(self, file_stream: BinaryIO, stream_info: StreamInfo, **kwargs) -> bool:
        # Check stream_info.mimetype, stream_info.extension, stream_info.url
        # If you must read from file_stream, save/restore position:
        #   cur_pos = file_stream.tell()
        #   data = file_stream.read(100)
        #   file_stream.seek(cur_pos)
        ...

    def convert(self, file_stream: BinaryIO, stream_info: StreamInfo, **kwargs) -> DocumentConverterResult:
        # Always seek(0) before reading in convert()
        file_stream.seek(0)
        ...
        return DocumentConverterResult(markdown="...", title="optional")
```

Optional dependencies: guard imports at module level, return `False` from `accepts()` when the dep is missing.

### Public API and Exceptions

Exported from `markitdown.__init__`:

```python
from markitdown import (
    MarkItDown,
    DocumentConverter,
    DocumentConverterResult,
    StreamInfo,
    PRIORITY_SPECIFIC_FILE_FORMAT,
    PRIORITY_GENERIC_FILE_FORMAT,
    MarkItDownException,        # base exception
    MissingDependencyException, # optional dep absent
    FailedConversionAttempt,    # converter tried but failed
    FileConversionException,    # I/O / format-level failure
    UnsupportedFormatException, # no converter accepted the input
)
```

### Plugin System

Plugins are discovered lazily via Python entry points (`markitdown.plugin` group) on first conversion call. A plugin package must:

1. Declare an entry point in `pyproject.toml`:
   ```toml
   [project.entry-points."markitdown.plugin"]
   my_plugin = "my_package_module"
   ```
2. Expose `register_converters(markitdown: MarkItDown, **kwargs)` at module level
3. Call `markitdown.register_converter(MyConverter())` inside it

See `packages/markitdown-sample-plugin` for the complete, working pattern.

Plugins are disabled by default. Enable via:
- Python: `MarkItDown(enable_plugins=True)`
- CLI: `markitdown --use-plugins file.pdf`
- MCP server: set `MARKITDOWN_ENABLE_PLUGINS=true` env var

### MCP Server (`markitdown-mcp`)

Exposes a single tool `convert_to_markdown(uri: str) -> str`. Supports:
- STDIO transport (default, for Claude Desktop)
- Streamable HTTP + SSE transport (`--http`, default port 3001)

### Optional Dependencies

Core installs minimal deps: `beautifulsoup4`, `requests`, `markdownify`, `magika~=0.6.1`, `charset-normalizer`, `defusedxml`.

Format-specific optional groups:

| Extra | Dependencies |
|-------|-------------|
| `[pptx]` | python-pptx |
| `[docx]` | mammoth, lxml |
| `[xlsx]` | pandas, openpyxl |
| `[xls]` | pandas, xlrd |
| `[pdf]` | pdfminer.six, pdfplumber |
| `[outlook]` | olefile |
| `[audio-transcription]` | pydub, SpeechRecognition |
| `[youtube-transcription]` | youtube-transcript-api |
| `[az-doc-intel]` | azure-ai-documentintelligence, azure-identity |
| `[az-content-understanding]` | azure-ai-contentunderstanding>=1.2.0b1, azure-identity |
| `[all]` | All of the above |

## Tests

Test files and their scope:

| File | Scope |
|------|-------|
| `test_module_vectors.py` | Integration tests using binary fixtures in `tests/test_files/` |
| `test_cli_vectors.py` | Same tests but via the CLI |
| `test_module_misc.py` | Unit/edge-case tests for the Python API |
| `test_cli_misc.py` | Unit/edge-case tests for the CLI |
| `test_pdf_memory.py` | PDF converter memory usage |
| `test_pdf_tables.py` | PDF table extraction |
| `test_pdf_masterformat.py` | MasterFormat spec PDF |
| `test_cu_converter.py` | Azure Content Understanding converter |
| `test_docintel_html.py` | Azure Document Intelligence converter |
| `_test_vectors.py` | Shared test vector definitions (imported by vector test files) |

CI (`.github/workflows/tests.yml`) runs `hatch test` on Python 3.10, 3.11, 3.12 for every PR.

## Conventions and Patterns

- **File naming**: converter files use `_snake_case_converter.py` with a leading underscore (private modules)
- **Import guards**: optional-dep imports are wrapped in `try/except ImportError`; `accepts()` returns `False` when the dep is absent
- **Stream handling**: `accepts()` must restore stream position if it reads; `convert()` should `seek(0)` before reading
- **No shell commands**: converters must not pass unsanitized input to subprocesses
- **defusedxml**: use for all XML parsing (never the stdlib `xml` module directly)
- **`DocumentConverterResult`**: always pass `markdown=` as the first positional arg; `title=` is optional keyword-only
- **`text_content` property**: soft-deprecated alias for `.markdown` — use `.markdown` in new code
- **Requests session**: `MarkItDown` creates a session with `Accept: text/markdown, text/html;q=0.9, ...` header for agent-friendly servers
- **`__plugin_interface_version__`**: set to `1` in plugin `_plugin.py` to declare compatibility

## Security Considerations

- Do NOT report vulnerabilities via public GitHub issues — use MSRC: https://msrc.microsoft.com/create-report
- Converters run with the privileges of the calling process; sanitize inputs in server/hosted environments
- Use `defusedxml` for all XML parsing
- Prefer the narrowest `convert_*` method (e.g., `convert_stream()` over `convert()`)
- The MCP server has no authentication — binding to non-localhost prints a prominent warning
- The `markitdown-ocr` plugin requires an `llm_client`; without one it silently falls back to the built-in converter

## Notes for AI Assistants

- The main development target is `packages/markitdown/`. Other packages depend on it but have their own release cycles.
- When adding a new converter: create `packages/markitdown/src/markitdown/converters/_<format>_converter.py`, add it to `converters/__init__.py`, and register it in `_markitdown.py`'s `enable_builtins()`.
- Converter registration order in `enable_builtins()` matters: later-registered converters are tried first. Specific format converters should be registered after generic ones (PlainTextConverter, HtmlConverter, ZipConverter are registered first/lowest priority).
- `DocumentConverterResult` previously required a `title` positional arg — it is now keyword-only and optional. Old code may still pass it positionally; be careful when refactoring.
- The `[all]` extra in `pyproject.toml` pins specific versions (e.g., `mammoth~=1.11.0`, `magika~=0.6.1`, `pdfminer.six>=20251230`, `youtube-transcript-api~=1.0.0`) — respect these when updating deps.
- Python 3.13 is supported (listed in classifiers and used in Dockerfiles) but CI only runs 3.10–3.12.
- The `markitdown-mcp` package pins `mcp~=1.8.0` and `markitdown>=0.1.1,<0.2.0` — both are version-sensitive.

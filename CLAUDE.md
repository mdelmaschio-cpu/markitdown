# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MarkItDown is a lightweight Python utility (built by Microsoft's AutoGen Team) that converts diverse document and file formats into Markdown text, primarily for consumption by LLMs and text-analysis pipelines. The goal is to preserve meaningful document structure (headings, lists, tables, links) while producing token-efficient output that mainstream LLMs understand natively.

MarkItDown is **not** a high-fidelity document renderer for human consumption — it is optimized for machine ingestion. It operates with the privileges of the current process (like `open()` or `requests.get()`), so callers must sanitize inputs and use the narrowest conversion API that fits their use case.

## Supported File Formats

Built-in converters (in `packages/markitdown/src/markitdown/converters/`) handle:

| Format | Converter file | Optional dep group |
|--------|---------------|--------------------|
| PDF | `_pdf_converter.py` | `[pdf]` — pdfminer.six, pdfplumber |
| PowerPoint (.pptx) | `_pptx_converter.py` | `[pptx]` — python-pptx |
| Word (.docx) | `_docx_converter.py` | `[docx]` — mammoth, lxml |
| Excel (.xlsx) | `_xlsx_converter.py` | `[xlsx]` — pandas, openpyxl |
| Older Excel (.xls) | `_xlsx_converter.py` | `[xls]` — pandas, xlrd |
| Images (EXIF + LLM caption) | `_image_converter.py` | none (LLM caption optional) |
| Audio (EXIF + transcription) | `_audio_converter.py` | `[audio-transcription]` — pydub, SpeechRecognition |
| HTML | `_html_converter.py` | none |
| CSV | `_csv_converter.py` | none |
| JSON | `_plain_text_converter.py` | none |
| XML | `_plain_text_converter.py` | defusedxml (core dep) |
| ZIP archives | `_zip_converter.py` | none (iterates contents) |
| EPub | `_epub_converter.py` | none |
| Jupyter Notebooks (.ipynb) | `_ipynb_converter.py` | none |
| Outlook messages (.msg) | `_outlook_msg_converter.py` | `[outlook]` — olefile |
| RSS/Atom feeds | `_rss_converter.py` | none |
| YouTube URLs | `_youtube_converter.py` | `[youtube-transcription]` — youtube-transcript-api |
| Wikipedia URLs | `_wikipedia_converter.py` | none |
| Bing SERP pages | `_bing_serp_converter.py` | none |
| Plain text fallback | `_plain_text_converter.py` | none |
| Azure Document Intelligence | `_doc_intel_converter.py` | `[az-doc-intel]` — azure-ai-documentintelligence |
| Azure Content Understanding | `_cu_converter.py` | `[az-content-understanding]` — azure-ai-contentunderstanding |

Azure Content Understanding is the only option for **video** files and provides structured field extraction (YAML front matter) for documents, audio, and images via prebuilt or custom analyzers.

## Repository Structure

This is a Python monorepo managed with [Hatch](https://hatch.pypa.io/). There is no workspace-level build; each package is developed and tested independently.

```
markitdown/
├── .devcontainer/          # VS Code Dev Container configuration
├── .github/
│   └── workflows/          # CI (tests.yml runs on Python 3.10–3.13)
├── Dockerfile              # Docker image for CLI usage (python:3.13-slim-bullseye)
├── packages/
│   ├── markitdown/                  # Core library — main development target
│   │   ├── pyproject.toml           # Hatch build config, optional dep groups
│   │   └── src/markitdown/
│   │       ├── __init__.py          # Public API exports
│   │       ├── __main__.py          # CLI entry point (markitdown command)
│   │       ├── __about__.py         # Version string
│   │       ├── _markitdown.py       # MarkItDown class — core orchestration
│   │       ├── _base_converter.py   # DocumentConverter base class + priority constants
│   │       ├── _exceptions.py       # Exception hierarchy
│   │       ├── _stream_info.py      # StreamInfo dataclass
│   │       ├── _uri_utils.py        # URI handling (data:, file: URIs)
│   │       ├── converters/          # All built-in converter implementations
│   │       └── converter_utils/     # Shared helpers (OMML→LaTeX, DOCX pre-processing)
│   │           └── docx/
│   │               ├── math/        # OMML equation→LaTeX rendering
│   │               └── pre_process.py
│   ├── markitdown-mcp/              # MCP server wrapping the library
│   │   ├── pyproject.toml
│   │   └── src/                    # Exposes convert tool for Claude Desktop
│   ├── markitdown-ocr/              # OCR plugin (PyMuPDF + LLM Vision)
│   │   └── README.md               # Full OCR plugin documentation
│   └── markitdown-sample-plugin/    # Reference plugin implementation
│       ├── pyproject.toml           # Shows entry-point registration pattern
│       └── src/markitdown_sample_plugin/
└── tests/ (per-package, under packages/<name>/tests/)
    ├── _test_vectors.py             # Shared test data definitions
    ├── test_module_vectors.py       # Module integration tests (file fixtures)
    ├── test_cli_vectors.py          # CLI integration tests
    ├── test_module_misc.py          # Unit / edge-case tests
    ├── test_cli_misc.py             # CLI unit tests
    ├── test_pdf_*.py                # PDF-specific tests (memory, tables, MasterFormat)
    ├── test_cu_converter.py         # Content Understanding converter tests
    └── test_files/                  # Test fixture files for all supported formats
```

## Development Setup

### Prerequisites

- Python 3.10 or higher (3.12 recommended for development)
- [Hatch](https://hatch.pypa.io/) for build and test management
- Optional system binaries (auto-detected via env vars):
  - `exiftool` — EXIF metadata extraction from images/audio
  - `ffmpeg` — audio format conversion

### Local Setup

```bash
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Or with uv:
uv venv --python=3.12 .venv
source .venv/bin/activate

# Install the core library with all optional dependencies (for development)
pip install -e 'packages/markitdown[all]'

# Install hatch for running tests and type checks
pip install hatch
```

### Dev Container

A fully configured VS Code Dev Container is available in `.devcontainer/`. Opening the repo in the container provides all runtime and dev dependencies pre-installed. Inside the container, just run `hatch test`.

## Build and Test Commands

All commands below are run from `packages/markitdown/` unless noted.

```bash
# Install hatch (one-time)
pip install hatch

# Run the full test suite (targets Python 3.10–3.12)
hatch test

# Run a specific test file
hatch test tests/test_module_misc.py

# Run a specific test by name
hatch test -- -k "test_name"

# Run PDF-specific tests
hatch test tests/test_pdf_tables.py
hatch test tests/test_pdf_memory.py
hatch test tests/test_pdf_masterformat.py

# Type checking (mypy)
hatch run types:check

# Open a development shell with all optional deps available
hatch shell

# Run pre-commit hooks (Black formatting + other checks) before submitting a PR
pre-commit run --all-files
```

CI runs on Python 3.10, 3.11, and 3.12 via GitHub Actions (`.github/workflows/tests.yml`).

## Architecture

### Converter Pipeline

The core orchestration is in `_markitdown.py`. The flow for a `convert()` call:

1. **Input normalization** — `convert(path_or_uri)` detects whether the input is a local path or URI and dispatches to `convert_local()`, `convert_uri()`, or `convert_response()`.
2. **File-type detection** — [Magika](https://github.com/google/magika) (ML-based) is used to populate a `StreamInfo` object containing `extension`, `mimetype`, and `charset`.
3. **Converter selection** — All registered `DocumentConverter` subclasses are iterated in **priority order** (ascending float). The first converter whose `accepts(stream, stream_info, **kwargs)` returns `True` performs the conversion.
4. **Result** — Returns `DocumentConverterResult` with a `.markdown` string property (also accessible as `.text_content` for backwards compatibility).

### Priority Constants (`_base_converter.py`)

```python
PRIORITY_SPECIFIC_FILE_FORMAT = 0.0   # format-specific converters (tried first)
PRIORITY_GENERIC_FILE_FORMAT  = 10.0  # fallback converters (plain text, etc.)
```

Cloud converters (Document Intelligence, Content Understanding) register at a priority that places them before the built-in format-specific converters when their endpoint is configured.

### Conversion Entry Points

`MarkItDown` exposes several methods for different input types — prefer the narrowest one for your use case:

| Method | Use case |
|--------|----------|
| `convert(path_or_uri)` | Auto-detect local path vs. URI |
| `convert_local(path)` | Local files only — no network access |
| `convert_stream(stream, stream_info)` | Raw seekable byte streams |
| `convert_response(response)` | `requests.Response` objects |
| `convert_uri(uri)` | Explicit URI handling (`data:` and `file:` URIs via `_uri_utils.py`) |

### Converter Contract

Every converter in `src/markitdown/converters/` subclasses `DocumentConverter` and must implement:

```python
class MyConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs) -> bool:
        # Return True if this converter can handle the input.
        # file_stream is seekable; do not consume it here.
        ...

    def convert(self, file_stream, stream_info, **kwargs) -> DocumentConverterResult:
        # Always seek(0) before reading the stream.
        file_stream.seek(0)
        ...
        return DocumentConverterResult(markdown="...")
```

Key rule: `accepts()` receives a seekable stream. Always call `file_stream.seek(0)` at the start of `convert()` before reading.

### Optional Dependency Guard Pattern

Converters with optional dependencies guard their imports at the top of the file and return `False` from `accepts()` when the dependency is absent:

```python
try:
    import pdfminer
    IS_AVAILABLE = True
except ImportError:
    IS_AVAILABLE = False

class PdfConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs) -> bool:
        return IS_AVAILABLE and stream_info.mimetype == "application/pdf"
```

This means installing `markitdown` without extras still works — unsupported formats are silently skipped rather than raising import errors.

### Converter Utilities

`src/markitdown/converter_utils/` contains shared helpers used by converter implementations (not part of the public API):

- `converter_utils/docx/math/` — renders OMML (Office Math Markup Language) equations to LaTeX for DOCX and PPTX converters
- `converter_utils/docx/pre_process.py` — DOM pre-processing for DOCX before markdownification

### Public API and Exception Hierarchy

The public surface exported from `markitdown.__init__`:

```python
from markitdown import (
    MarkItDown,
    DocumentConverterResult,
    StreamInfo,
    # Exceptions:
    MarkItDownException,          # base exception
    MissingDependencyException,   # optional dep absent
    FailedConversionAttempt,      # converter tried but could not complete
    FileConversionException,      # I/O or format-level failure
    UnsupportedFormatException,   # no converter accepted the input
)
```

## Plugin System

Plugins extend MarkItDown with additional converters without modifying the core package.

### How Plugins Work

Plugins are discovered lazily via Python [entry points](https://packaging.python.org/en/latest/specifications/entry-points/) in the `markitdown.plugin` group. They are loaded once on the first conversion call (via `_load_plugins()`) when `enable_plugins=True` is passed to `MarkItDown()`.

A plugin package must:
1. Declare an entry point in its `pyproject.toml`:
   ```toml
   [project.entry-points."markitdown.plugin"]
   my_plugin = "my_plugin_module"
   ```
2. Expose a `register_converters(markitdown_instance, **kwargs)` function in the named module.

### Using Plugins

```bash
# List installed plugins
markitdown --list-plugins

# Enable plugins at the CLI
markitdown --use-plugins path-to-file.pdf
```

```python
# Enable plugins in Python
md = MarkItDown(enable_plugins=True)
result = md.convert("file.pdf")
```

Find community plugins by searching GitHub for `#markitdown-plugin`.

### markitdown-ocr Plugin

The `packages/markitdown-ocr/` package is an official plugin that adds OCR support to PDF, DOCX, PPTX, and XLSX converters, extracting text from embedded images using LLM Vision (the same `llm_client`/`llm_model` pattern as image descriptions).

```bash
pip install markitdown-ocr
pip install openai  # or any OpenAI-compatible client
```

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(
    enable_plugins=True,
    llm_client=OpenAI(),
    llm_model="gpt-4o",
)
result = md.convert("document_with_images.pdf")
print(result.text_content)
```

If no `llm_client` is provided the plugin still loads but OCR is silently skipped.

### markitdown-mcp Package

`packages/markitdown-mcp/` wraps the core library as an [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) server. This allows Claude Desktop (and other MCP-compatible clients) to call MarkItDown as a tool, passing a file path or URI to get back Markdown text.

### Writing a New Plugin

See `packages/markitdown-sample-plugin/` for the complete reference implementation. The pattern:

1. Create a Python package with `hatchling` build backend.
2. Implement your `DocumentConverter` subclass.
3. Expose `register_converters(markitdown_instance, **kwargs)` in your package's `__init__.py`.
4. Declare the entry point in `pyproject.toml` under `[project.entry-points."markitdown.plugin"]`.
5. Install the package (`pip install .`) and test with `markitdown --use-plugins`.

The sample plugin uses `striprtf` to add RTF support as an example of a complete, installable plugin.

## Python API Usage

### Basic Conversion

```python
from markitdown import MarkItDown

md = MarkItDown(enable_plugins=False)  # True to enable installed plugins
result = md.convert("test.xlsx")
print(result.text_content)  # or result.markdown
```

### LLM-Assisted Image/PPTX Descriptions

```python
from markitdown import MarkItDown
from openai import OpenAI

client = OpenAI()
md = MarkItDown(
    llm_client=client,
    llm_model="gpt-4o",
    llm_prompt="optional custom prompt",  # overrides default description prompt
)
result = md.convert("example.jpg")
print(result.text_content)
```

### Azure Document Intelligence

```python
from markitdown import MarkItDown

md = MarkItDown(docintel_endpoint="<document_intelligence_endpoint>")
result = md.convert("test.pdf")
print(result.text_content)
```

### Azure Content Understanding

```python
from markitdown import MarkItDown

# Zero-config — auto-selects prebuilt analyzer per file type
md = MarkItDown(cu_endpoint="<content_understanding_endpoint>")
result = md.convert("report.pdf")    # documents → prebuilt-documentSearch
result = md.convert("meeting.mp4")  # video → prebuilt-videoSearch
result = md.convert("call.wav")     # audio → prebuilt-audioSearch
print(result.markdown)
```

```python
# With a custom analyzer (domain-specific field extraction)
from markitdown import MarkItDown

md = MarkItDown(
    cu_endpoint="<content_understanding_endpoint>",
    cu_analyzer_id="my-invoice-analyzer",
)
result = md.convert("invoice.pdf")
# Output includes YAML front matter with extracted fields
print(result.markdown)
```

```python
# Restrict which file types route to Content Understanding (to control cost)
from markitdown import MarkItDown
from markitdown.converters import ContentUnderstandingFileType

md = MarkItDown(
    cu_endpoint="<content_understanding_endpoint>",
    cu_file_types=[ContentUnderstandingFileType.PDF],  # only PDFs use CU
)
```

## CLI Usage

```bash
# Convert a file, print to stdout
markitdown path-to-file.pdf

# Convert a file, write to output file
markitdown path-to-file.pdf -o document.md

# Convert via stdin pipe
cat path-to-file.pdf | markitdown

# Use Azure Document Intelligence
markitdown path-to-file.pdf -o document.md -d -e "<document_intelligence_endpoint>"

# Use Azure Content Understanding
markitdown path-to-file.pdf --use-cu --cu-endpoint "<content_understanding_endpoint>"

# List installed plugins
markitdown --list-plugins

# Enable plugins
markitdown --use-plugins path-to-file.pdf
```

## Docker Usage

The root `Dockerfile` builds a `python:3.13-slim-bullseye` image with:
- `ffmpeg` and `exiftool` pre-installed as runtime dependencies
- `markitdown[all]` and `markitdown-sample-plugin` installed from the local packages
- Optional `git` installation via build arg `INSTALL_GIT=true`
- Runs as a non-root user (`nobody:nogroup` by default)

```bash
# Build the image
docker build -t markitdown:latest .

# Convert a file via stdin/stdout
docker run --rm -i markitdown:latest < ~/your-file.pdf > output.md

# Build with git support
docker build --build-arg INSTALL_GIT=true -t markitdown:git .

# Customize user
docker build --build-arg USERID=1000 --build-arg GROUPID=1000 -t markitdown:latest .
```

The `packages/markitdown-mcp/Dockerfile` provides a separate image for running the MCP server.

## Installation Options

```bash
# Install from PyPI with all optional dependencies
pip install 'markitdown[all]'

# Install specific format support only
pip install 'markitdown[pdf,docx,pptx]'

# Available optional groups:
# [pptx]                  — PowerPoint
# [docx]                  — Word documents
# [xlsx]                  — Excel (xlsx)
# [xls]                   — Older Excel (xls)
# [pdf]                   — PDF files
# [outlook]               — Outlook .msg files
# [audio-transcription]   — WAV/MP3 speech transcription
# [youtube-transcription] — YouTube transcript fetching
# [az-doc-intel]          — Azure Document Intelligence
# [az-content-understanding] — Azure Content Understanding (video, audio, docs)

# Install from source for development
git clone https://github.com/microsoft/markitdown.git
cd markitdown
pip install -e 'packages/markitdown[all]'
```

## Tests

Test files in `packages/markitdown/tests/` mirror the feature surface:

| File | Description |
|------|-------------|
| `_test_vectors.py` | Shared test data definitions (expected output strings) |
| `test_module_vectors.py` | Integration tests using file fixtures in `tests/test_files/` |
| `test_cli_vectors.py` | CLI integration tests using the same fixtures |
| `test_module_misc.py` | Unit and edge-case tests for module behavior |
| `test_cli_misc.py` | Unit tests for CLI flags and edge cases |
| `test_pdf_tables.py` | PDF table extraction correctness |
| `test_pdf_memory.py` | PDF memory handling and stream tests |
| `test_pdf_masterformat.py` | PDF parsing against MasterFormat spec document |
| `test_cu_converter.py` | Azure Content Understanding converter tests |
| `test_docintel_html.py` | Document Intelligence HTML output tests |

Test fixture files (sample documents for all supported formats) live in `packages/markitdown/tests/test_files/`.

## Security Considerations

From `SECURITY.md`:

- **MarkItDown operates with process privileges.** It accesses any resource the running process can access, just like `open()` or `requests.get()`. Do not pass untrusted user input directly to MarkItDown in server-side or hosted applications.
- **Use the narrowest API.** Prefer `convert_local()` for local files, `convert_response()` when you control the HTTP fetch, or `convert_stream()` for maximum control.
- **Sanitize inputs.** In untrusted environments, restrict file paths, limit URI schemes and network destinations, and block access to private/loopback/metadata-service addresses.
- **XML parsing** uses `defusedxml` to prevent XXE and related attacks.
- **No shell commands** — do not pass unsanitized user input to shell commands or external processes.

## Contributing

### Workflow

1. Fork the repository and create a branch from `main`.
2. Navigate to `packages/markitdown/` for core library changes.
3. Install dev dependencies: `pip install hatch && hatch shell`.
4. Make your changes, add tests under `packages/markitdown/tests/`.
5. Run tests: `hatch test`.
6. Run type checks: `hatch run types:check`.
7. Run pre-commit checks: `pre-commit run --all-files` (enforces Black formatting).
8. Submit a pull request. A CLA bot will request a Contributor License Agreement on first submission.

### CLA and Code of Conduct

Contributions require agreement to the [Microsoft CLA](https://cla.opensource.microsoft.com). This project follows the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

### Finding Work

- Issues labeled [`open for contribution`](https://github.com/microsoft/markitdown/issues?q=is%3Aissue+is%3Aopen+label%3A%22open+for+contribution%22)
- PRs needing review: [`open for reviewing`](https://github.com/microsoft/markitdown/pulls?q=is%3Apr+is%3Aopen+label%3A%22open+for+reviewing%22)

### Adding a New Built-in Converter

1. Create `packages/markitdown/src/markitdown/converters/_my_format_converter.py`.
2. Subclass `DocumentConverter` from `._base_converter`.
3. Guard optional imports and return `False` from `accepts()` when deps are absent.
4. Register the converter in `packages/markitdown/src/markitdown/converters/__init__.py`.
5. Add it to `_markitdown.py` where converters are instantiated and registered.
6. Add optional dep group to `packages/markitdown/pyproject.toml` if needed.
7. Add test fixtures and test vectors.

### Adding a 3rd-Party Plugin

See `packages/markitdown-sample-plugin/` for the reference implementation and publish your package to PyPI with the `#markitdown-plugin` topic tag so the community can find it.

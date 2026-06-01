# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MarkItDown is a Python library (by Microsoft's AutoGen Team) that converts diverse document formats to Markdown for LLM consumption. It is organized as a monorepo with four packages:

- `packages/markitdown` — core library (main development target)
- `packages/markitdown-mcp` — MCP server wrapping the library for Claude Desktop integration
- `packages/markitdown-ocr` — OCR plugin using PyMuPDF + LLM vision
- `packages/markitdown-sample-plugin` — reference plugin implementation

## Commands

All commands below are run from `packages/markitdown/` unless noted.

```bash
# Install build/test tool
pip install hatch

# Run all tests (targets Python 3.10–3.12)
hatch test

# Run a single test file
hatch test tests/test_module_misc.py

# Run a single test by name
hatch test -- -k "test_name"

# Type checking
hatch run types:check

# Enter development shell
hatch shell

# Formatting (pre-commit hook uses Black)
pre-commit run --all-files
```

## Architecture

### Converter Pipeline

The core flow in `_markitdown.py`:

1. `MarkItDown.convert(path_or_uri)` is the main entry point
2. File type is detected via **Magika** (ML-based) to populate a `StreamInfo` object (extension, mimetype, charset)
3. All registered `DocumentConverter` subclasses are iterated in priority order
4. The first converter whose `accepts(stream, stream_info, **kwargs) -> bool` returns `True` performs the conversion
5. Returns `DocumentConverterResult` with a `.markdown` string property

Priority constants in `_base_converter.py`:
- `PRIORITY_SPECIFIC_FILE_FORMAT = 0.0` — tried first (format-specific converters)
- `PRIORITY_GENERIC_FILE_FORMAT = 10.0` — fallback (plain text, etc.)

### Converter Contract

Every converter in `src/markitdown/converters/` must implement:

```python
class MyConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs) -> bool: ...
    def convert(self, file_stream, stream_info, **kwargs) -> DocumentConverterResult: ...
```

`accepts()` is called with a seekable stream; always reset the stream position (`file_stream.seek(0)`) before reading in `convert()`.

### Converter Utilities

`src/markitdown/converter_utils/` provides shared helpers used by converter implementations (not part of the public API):
- `converter_utils/docx/math/` — renders OMML (Office Math Markup Language) equations to LaTeX for DOCX/PPTX converters
- `converter_utils/docx/pre_process.py` — DOM pre-processing for DOCX before markdownification

### Public API / Exceptions

The public surface exported from `markitdown.__init__` includes the exception hierarchy, useful when writing plugins or handling errors:
- `MarkItDownException` — base exception
- `MissingDependencyException` — raised when an optional dep is absent
- `FailedConversionAttempt` — a converter tried but could not complete
- `FileConversionException` — I/O or format-level conversion failure
- `UnsupportedFormatException` — no converter accepted the input

### Plugin System

Plugins are discovered lazily via Python entry points (`markitdown.plugin` group). A plugin package must expose a `register_converters(markitdown_instance, **kwargs)` function. See `packages/markitdown-sample-plugin` for the complete pattern. Plugins are loaded once on first conversion call via `_load_plugins()`.

### Conversion Entry Points

`MarkItDown` exposes several conversion methods for different input types:
- `convert(path_or_uri)` — auto-detects local path vs. URI
- `convert_local(path)` — local files only
- `convert_stream(stream, stream_info)` — raw streams
- `convert_response(response)` — `requests.Response` objects
- `convert_uri(uri)` — explicit URI handling (supports `data:` and `file:` URIs via `_uri_utils.py`)

### Optional Dependencies

Core installs minimal deps (beautifulsoup4, requests, markdownify, magika, charset-normalizer, defusedxml). Format-specific converters guard their imports and silently decline (`accepts()` returns `False`) when optional deps are missing. Install everything with `pip install markitdown[all]`.

Key optional groups: document formats (pptx, mammoth, pandas, openpyxl), PDF (pdfminer.six, pdfplumber), audio (SpeechRecognition, pydub), Azure (azure-ai-documentintelligence, azure-ai-contentunderstanding).

## Tests

Test files mirror the feature surface:
- `test_module_vectors.py` / `test_cli_vectors.py` — integration tests using fixtures in `tests/test_files/`
- `test_module_misc.py` / `test_cli_misc.py` — unit/edge-case tests
- `test_pdf_*.py` — PDF-specific (memory, tables, MasterFormat spec)
- `_test_vectors.py` — shared test data definitions

CI runs on Python 3.10, 3.11, and 3.12 via GitHub Actions (`.github/workflows/tests.yml`).

## Writing a New Converter

Quick reference for adding support for a new file format:

1. Create `src/markitdown/converters/my_format.py`
2. Subclass `DocumentConverter` and implement `accepts()` and `convert()`
3. Guard optional dep imports at the top of the file with try/except
4. Register in `_markitdown.py` via `self.register_converter(MyConverter())`
5. Set `PRIORITY_SPECIFIC_FILE_FORMAT` (0.0) for format-specific converters
6. Add a test file to `tests/test_files/` and a test case in `test_module_vectors.py`
7. Document the optional dependency in `pyproject.toml` under the appropriate extras group

Pattern for optional dependency guard:
```python
try:
    import my_optional_dep
    MY_DEP_AVAILABLE = True
except ImportError:
    MY_DEP_AVAILABLE = False
```

In `accepts()`: return `False` immediately if `not MY_DEP_AVAILABLE`.

## markitdown-mcp

The `packages/markitdown-mcp/` package wraps the core library as an MCP server for Claude Desktop. Run it from `packages/markitdown-mcp/`:

```bash
pip install hatch
hatch run serve        # start MCP server
hatch test             # run MCP-specific tests
```

The MCP server exposes `convert_to_markdown(uri)` as a tool. Its test suite is separate from the core library tests.

## Security Considerations

From `SECURITY.md`: converters perform I/O operations on provided files. Use `defusedxml` for XML parsing. Prefer narrow API surface over broad file system access. Do not pass unsanitized user input directly to shell commands or external processes.

When writing converters that invoke external processes (e.g., for audio transcription), use `subprocess.run()` with a fixed argument list — never pass user-controlled strings to `shell=True`.

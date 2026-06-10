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

# Type checking (mypy)
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
3. All registered `DocumentConverter` subclasses are iterated in priority order (stable sort)
4. The first converter whose `accepts(stream, stream_info, **kwargs) -> bool` returns `True` performs the conversion
5. Returns `DocumentConverterResult` with a `.markdown` string property

Priority constants exported from `markitdown.__init__`:
- `PRIORITY_SPECIFIC_FILE_FORMAT = 0.0` — tried first (format-specific converters)
- `PRIORITY_GENERIC_FILE_FORMAT = 10.0` — fallback (plain text, etc.)

Plugins can register converters with priority `-1.0` to run before all built-ins and effectively replace them.

### Converter Contract

Every converter in `src/markitdown/converters/` must implement:

```python
class MyConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs) -> bool: ...
    def convert(self, file_stream, stream_info, **kwargs) -> DocumentConverterResult: ...
```

**Critical invariants:**
- `accepts()` must not consume or reposition the stream — the core always resets to position 0 before calling `convert()`, but `accepts()` should not rely on this
- `convert()` must call `file_stream.seek(0)` before reading, because the core resets before calling it but multiple converters may have read it
- When optional dependencies are absent, `accepts()` should return `False` silently; raise `MissingDependencyException` only in `convert()`
- The standard guard pattern: try the import at module level, store the exception, raise `MissingDependencyException` in `convert()` if the import failed

### Optional Dependency Guard Pattern

```python
try:
    import some_optional_lib
    _OPTIONAL_LIB_AVAILABLE = True
except ImportError as e:
    _OPTIONAL_LIB_AVAILABLE = False
    _OPTIONAL_LIB_IMPORT_ERROR = e

class MyConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs) -> bool:
        return _OPTIONAL_LIB_AVAILABLE and stream_info.extension == ".xyz"

    def convert(self, file_stream, stream_info, **kwargs) -> DocumentConverterResult:
        if not _OPTIONAL_LIB_AVAILABLE:
            raise MissingDependencyException(...) from _OPTIONAL_LIB_IMPORT_ERROR
        ...
```

### StreamInfo

`StreamInfo` (from `_stream_info.py`) is a dataclass populated by Magika and passed to every converter:

| Field | Type | Description |
|-------|------|-------------|
| `mimetype` | `str \| None` | MIME type detected by Magika |
| `extension` | `str \| None` | File extension (e.g. `.pdf`) |
| `charset` | `str \| None` | Character encoding |
| `filename` | `str \| None` | From path, URL, or Content-Disposition header |
| `local_path` | `str \| None` | Absolute path if source is a local file |
| `url` | `str \| None` | Original URL if source is HTTP/HTTPS |

Use `stream_info.copy_and_update(mimetype=..., extension=...)` to create a modified copy.

### Converter Implementations

All 19 converters live in `src/markitdown/converters/`:

| File | Converter(s) | Key dependency |
|------|-------------|----------------|
| `_plain_text_converter.py` | `PlainTextConverter` | none (PRIORITY_GENERIC_FILE_FORMAT) |
| `_html_converter.py` | `HtmlConverter` | beautifulsoup4, markdownify |
| `_rss_converter.py` | `RssConverter` | beautifulsoup4 |
| `_wikipedia_converter.py` | `WikipediaConverter` | requests |
| `_youtube_converter.py` | `YouTubeConverter` | youtube-transcript-api |
| `_ipynb_converter.py` | `IpynbConverter` | none |
| `_bing_serp_converter.py` | `BingSerpConverter` | beautifulsoup4 |
| `_pdf_converter.py` | `PdfConverter` | pdfminer.six, pdfplumber |
| `_docx_converter.py` | `DocxConverter` | mammoth, lxml |
| `_xlsx_converter.py` | `XlsxConverter`, `XlsConverter` | pandas, openpyxl / xlrd |
| `_pptx_converter.py` | `PptxConverter` | python-pptx |
| `_image_converter.py` | `ImageConverter` | Pillow; LLM vision optional |
| `_audio_converter.py` | `AudioConverter` | SpeechRecognition, pydub |
| `_outlook_msg_converter.py` | `OutlookMsgConverter` | olefile |
| `_zip_converter.py` | `ZipConverter` | none (nested extraction) |
| `_epub_converter.py` | `EpubConverter` | beautifulsoup4 |
| `_csv_converter.py` | `CsvConverter` | none |
| `_doc_intel_converter.py` | `DocumentIntelligenceConverter` | azure-ai-documentintelligence |
| `_cu_converter.py` | `ContentUnderstandingConverter` | azure-ai-contentunderstanding |

Helper modules used by converters:
- `_llm_caption.py` — LLM vision captioning for images (requires `llm_client` + `llm_model` kwargs)
- `_transcribe_audio.py` — Audio transcription helper

### Converter Utilities

`src/markitdown/converter_utils/` provides shared helpers (not part of the public API):

- `converter_utils/docx/math/omml.py` — Converts OMML (Office Math Markup Language) to LaTeX
- `converter_utils/docx/math/latex_dict.py` — Symbol mapping tables used by the OMML converter
- `converter_utils/docx/pre_process.py` — DOM pre-processing of DOCX markup before markdownification (calls the OMML converter, manipulates BeautifulSoup tree)

### Conversion Entry Points

`MarkItDown` exposes several conversion methods:

| Method | Input | Notes |
|--------|-------|-------|
| `convert(source)` | path string or URI | auto-detects local path vs URI |
| `convert_local(path)` | file path | local files only |
| `convert_stream(stream, stream_info)` | binary stream | raw streams with explicit metadata |
| `convert_response(response)` | `requests.Response` | parses headers for StreamInfo |
| `convert_uri(uri)` | URI string | handles `data:`, `file:`, `http:`, `https:` |

URI utilities in `_uri_utils.py`: `parse_data_uri()` and `file_uri_to_path()`.

### LLM Integration

`ImageConverter` and Azure converters accept optional kwargs passed through from `MarkItDown.convert()`:
- `llm_client` — an OpenAI-compatible client instance
- `llm_model` — model name string (e.g. `"gpt-4o"`)

When provided, `ImageConverter` calls the LLM to generate an image description.

### Public API / Exceptions

Everything exported from `markitdown.__init__`:

**Classes:**
- `MarkItDown` — main class
- `DocumentConverter` — base class for converters and plugins
- `DocumentConverterResult` — holds `.markdown` (`.text_content` is a soft-deprecated alias)
- `StreamInfo` — file metadata dataclass

**Exceptions:**
- `MarkItDownException` — base exception
- `MissingDependencyException` — raised when an optional dep is absent
- `FailedConversionAttempt` — a converter tried but could not complete
- `FileConversionException` — I/O or format-level conversion failure
- `UnsupportedFormatException` — no converter accepted the input

**Constants:**
- `PRIORITY_SPECIFIC_FILE_FORMAT = 0.0`
- `PRIORITY_GENERIC_FILE_FORMAT = 10.0`

### Plugin System

Plugins are discovered lazily via Python entry points (`markitdown.plugin` group). The plugin module must expose:
- `__plugin_interface_version__ = 1`
- `register_converters(markitdown_instance: MarkItDown, **kwargs) -> None`

`pyproject.toml` entry point declaration:
```toml
[project.entry-points."markitdown.plugin"]
my_plugin = "my_package"
```

Registration flow: `MarkItDown()` constructor calls `enable_plugins(**kwargs)` if enabled → `_load_plugins()` (runs once) → calls `plugin.register_converters(self, **kwargs)` for each plugin → plugin calls `markitdown.register_converter(converter_instance, priority=...)`.

See `packages/markitdown-sample-plugin` for the minimal complete pattern (RTF converter via `striprtf`).

### Optional Dependencies

Core installs minimal deps: `beautifulsoup4`, `requests`, `markdownify`, `magika~=0.6.1`, `charset-normalizer`, `defusedxml`. Install everything with `pip install markitdown[all]`.

Key optional groups defined in `pyproject.toml`:

| Group | Packages |
|-------|----------|
| `pptx` | python-pptx |
| `docx` | mammoth~=1.11.0, lxml |
| `xlsx` | pandas, openpyxl |
| `xls` | pandas, xlrd |
| `pdf` | pdfminer.six>=20251230, pdfplumber>=0.11.9 |
| `outlook` | olefile |
| `audio-transcription` | pydub, SpeechRecognition |
| `youtube-transcription` | youtube-transcript-api~=1.0.0 |
| `az-doc-intel` | azure-ai-documentintelligence, azure-identity |
| `az-content-understanding` | azure-ai-contentunderstanding>=1.2.0b1, azure-identity |

## Secondary Packages

### markitdown-mcp

MCP (Model Context Protocol) server for Claude Desktop integration.

- **Entry point:** `markitdown-mcp` CLI
- **Dependency:** `mcp~=1.8.0`, `markitdown[all]>=0.1.1,<0.2.0`
- **Single tool exposed:** `convert_to_markdown(uri: str) -> str`
- **Transports:** stdio, HTTP + SSE, streamable HTTP
- **Env var:** `MARKITDOWN_ENABLE_PLUGINS=true` to enable plugin discovery

### markitdown-ocr

OCR plugin that uses LLM vision to extract text from images embedded in PDF, DOCX, PPTX, and XLSX files.

- **Entry point:** `markitdown.plugin` (auto-discovered when installed)
- **Priority:** `-1.0` — runs before all built-in converters, effectively replacing them
- **Converters:** `PdfConverterWithOCR`, `DocxConverterWithOCR`, `PptxConverterWithOCR`, `XlsxConverterWithOCR`
- **Service:** `LLMVisionOCRService` wraps any OpenAI-compatible client
- **Key deps:** PyMuPDF, pdfminer.six, pdfplumber, mammoth, python-docx, python-pptx, pandas, openpyxl, Pillow; `openai>=1.0.0` optional

### markitdown-sample-plugin

Minimal reference plugin showing the complete plugin contract: entry point declaration + `register_converters()` + a single `RtfConverter` class.

## Tests

Test files in `packages/markitdown/tests/`:

| File | Coverage |
|------|----------|
| `test_module_vectors.py` | File format integration (uses `_test_vectors.py` fixtures) |
| `test_cli_vectors.py` | CLI argument parsing with the same vectors |
| `test_module_misc.py` | Unit/edge-case tests, URI utilities, mock tests |
| `test_cli_misc.py` | CLI misc behavior |
| `test_pdf_memory.py` | PDF memory profiling |
| `test_pdf_tables.py` | PDF table extraction |
| `test_pdf_masterformat.py` | PDF MasterFormat partial-numbering edge case |
| `test_cu_converter.py` | Azure Content Understanding converter |
| `test_docintel_html.py` | Document Intelligence HTML output |

Test fixtures live in `tests/test_files/`; expected outputs in `tests/test_files/expected_outputs/`.

`_test_vectors.py` defines `FileTestVector` (filename, mimetype, charset, url, must_include, must_not_include) and the `GENERAL_TEST_VECTORS` list used by both module and CLI vector tests.

CI runs on Python 3.10, 3.11, and 3.12 via GitHub Actions (`.github/workflows/tests.yml`).

## Security Considerations

From `SECURITY.md`: converters perform I/O on provided files. Use `defusedxml` for XML parsing (never the stdlib `xml` module directly). Prefer narrow API surface over broad filesystem access. Do not pass unsanitized user input directly to shell commands or external processes.

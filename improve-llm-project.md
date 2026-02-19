# dbt-osmosis LLM Enhancement Project Plan

## Executive Summary

This project extends dbt-osmosis's LLM-powered documentation generation capabilities with two major enhancements:

1. **CLI Agent Integration** - Support for CLI-based LLM tools (claude-code, copilot-cli, etc.) as an alternative to API calls
2. **Custom Prompt Templates** - Allow users to define their own Jinja2 prompt templates within their dbt project

These changes maintain full backwards compatibility while opening new possibilities for local LLM usage and organization-specific documentation standards.

---

## Project Goals

### Goal 1: CLI Agent Integration

**Problem**: Current implementation only supports OpenAI-compatible APIs, requiring API keys and internet connectivity.

**Solution**: Abstract the LLM interaction layer to support both API calls and CLI-based tools.

**Benefits**:
- Use local LLMs via CLI tools without API costs
- Integrate with enterprise-approved CLI tools (e.g., corporate Copilot deployments)
- Work offline with locally-hosted models
- Leverage CLI tools that handle authentication/context automatically

**Supported CLI Tools** (initial targets):
- `claude` (Claude Code CLI)
- `gh copilot` (GitHub Copilot CLI)
- `llm` (Simon Willison's LLM CLI)
- `aider` (AI pair programming CLI)

### Goal 2: Custom Prompt Templates

**Problem**: Hardcoded prompts don't accommodate organization-specific documentation standards, terminology, or formatting requirements.

**Solution**: Allow users to define Jinja2 templates in their dbt project that override default prompts.

**Benefits**:
- Customize documentation tone, style, and format
- Include company-specific terminology or business context
- Add custom rules (e.g., "always mention data privacy classification")
- Share templates across teams via version control
- Iterate on prompts without modifying dbt-osmosis source

---

## Architectural Changes

### Current Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      CLI Layer                          │
│                   (cli/main.py)                         │
│               --synthesize flag                         │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  Transform Layer                        │
│               (core/transforms.py)                      │
│    synthesize_missing_documentation_with_openai()       │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    LLM Layer                            │
│                 (core/llm.py)                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  get_llm_client() → OpenAI/Azure/Ollama client  │   │
│  │  _create_llm_prompt_*() → Hardcoded prompts     │   │
│  │  generate_*() → API call + parse response       │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Proposed Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      CLI Layer                          │
│                   (cli/main.py)                         │
│  --synthesize, --llm-backend, --llm-cli, --prompts-dir  │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  Transform Layer                        │
│               (core/transforms.py)                      │
│      synthesize_missing_documentation() [renamed]       │
│              passes project_dir for templates           │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    LLM Layer                            │
│                 (core/llm.py)                           │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │           Backend Abstraction                      │ │
│  │  ┌─────────────┐    ┌─────────────┐              │ │
│  │  │ APIBackend  │    │ CLIBackend  │              │ │
│  │  │ (existing)  │    │   (new)     │              │ │
│  │  └─────────────┘    └─────────────┘              │ │
│  │         │                  │                      │ │
│  │         └────────┬─────────┘                      │ │
│  │                  ▼                                │ │
│  │         get_llm_backend()                         │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │           Template System                          │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │  PromptTemplateLoader                       │  │ │
│  │  │  - Loads from .dbt-osmosis/prompts/         │  │ │
│  │  │  - Falls back to built-in defaults          │  │ │
│  │  │  - Renders with Jinja2                      │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │           Generate Functions                       │ │
│  │  generate_model_spec_as_json()                    │ │
│  │  generate_column_doc()                            │ │
│  │  generate_table_doc()                             │ │
│  │  - Use backend abstraction                        │ │
│  │  - Use template system                            │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### New Project Structure for Users

```
my-dbt-project/
├── dbt_project.yml
├── .dbt-osmosis/
│   └── prompts/
│       ├── model_docs.md      # Custom template for model + columns
│       ├── column_doc.md      # Custom template for single column
│       └── table_doc.md       # Custom template for table description
├── models/
│   └── ...
```

---

## Implementation Phases

### Phase 1: Backend Abstraction Layer

**Objective**: Refactor existing LLM code into an abstraction that supports multiple backends without changing external behavior.

**Design Principles**:
- **Composition over inheritance**: Use Protocol for type hints, not ABC for enforcement
- **No factory function**: Backend instantiation happens directly in `_call_llm()`
- **Data objects**: Separate configuration data (`ProviderConfig`) from behavior (`APIBackend`)
- **Dependency injection**: `APIBackend` accepts optional `ProviderConfig` for testing

#### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      llm.py (refactored)                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Data Layer                                             │ │
│  │  ┌──────────────────┐  ┌─────────────────────────────┐ │ │
│  │  │  LLMResponse     │  │  ProviderConfig             │ │ │
│  │  │  - content       │  │  - provider: str            │ │ │
│  │  │  - raw_response  │  │  - client: Any              │ │ │
│  │  │  - model         │  │  - model: str               │ │ │
│  │  │  - provider      │  │                             │ │ │
│  │  └──────────────────┘  └─────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Type Layer                                            │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │  LLMBackend (Protocol)                          │   │ │
│  │  │  - generate(messages, temperature) -> LLMResponse   │ │
│  │  │  - is_available() -> bool                       │   │ │
│  │  │  - name: str (property)                         │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Implementation Layer                                  │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │  APIBackend                                     │   │ │
│  │  │  - __init__(config: ProviderConfig | None)      │   │ │
│  │  │  - _ensure_config() -> ProviderConfig           │   │ │
│  │  │  - generate(messages, temperature) -> Response  │   │ │
│  │  │  - is_available() -> bool                       │   │ │
│  │  │  - name: str (property)                         │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                        │ │
│  │  Phase 2 adds: CLIBackend (same interface)             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Helper Layer                                          │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │  _call_llm(messages, temperature, backend=None) │   │ │
│  │  │  - Instantiates APIBackend (or uses injected)   │   │ │
│  │  │  - Calls backend.generate()                     │   │ │
│  │  │  - Extracts content, raises on empty            │   │ │
│  │  │  - Returns stripped string                      │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Backwards Compatibility                                │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │  get_llm_client() -> tuple[Any, str]            │   │ │
│  │  │  - Wrapper that uses APIBackend internally      │   │ │
│  │  │  - Returns (client, model_engine) as before     │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### Components

| Component | Type | Purpose |
|-----------|------|---------|
| `LLMResponse` | dataclass | Unified response container (content, raw_response, model, provider) |
| `ProviderConfig` | dataclass | Configuration data (provider name, client instance, model string) |
| `LLMBackend` | Protocol | Type hint interface for static analysis (not runtime enforced) |
| `APIBackend` | class | OpenAI-compatible API backend, composes `ProviderConfig` |
| `_call_llm()` | function | Helper that instantiates backend and extracts content |
| `get_llm_client()` | function | Backwards-compat wrapper returning (client, model) tuple |

#### Key Design Decisions

1. **Protocol instead of ABC**: `LLMBackend` is a `typing.Protocol` - no inheritance required. Any class with matching methods satisfies the type.

2. **No factory function**: `_call_llm()` directly instantiates `APIBackend()`. Phase 2 will add simple if/else for backend selection. No need for a separate `get_llm_backend()`.

3. **Composition via ProviderConfig**: `APIBackend` HAS a `ProviderConfig` (injected or created). This enables easy testing via dependency injection.

4. **Config creation inside APIBackend**: The `_ensure_config()` method creates `ProviderConfig` from environment variables. Logic stays encapsulated in the class.

5. **get_llm_client() as wrapper**: Existing function delegates to `APIBackend` for backwards compatibility with any external code using it.

#### Refactoring Impact

Each `generate_*()` function simplifies from ~15 lines to ~2 lines:

```python
# Before (repeated in every generate function)
def generate_column_doc(...) -> str:
    messages = _create_llm_prompt_for_column(...)
    client, model_engine = get_llm_client()
    provider = os.getenv("LLM_PROVIDER", "openai").lower()
    if provider == "azure-openai":
        response = client.ChatCompletion.create(...)
    else:
        response = client.chat.completions.create(...)
    content = response.choices[0].message.content
    if not content:
        raise LLMResponseError("...")
    return content.strip()

# After
def generate_column_doc(...) -> str:
    messages = _create_llm_prompt_for_column(...)
    return _call_llm(messages, temperature)
```

#### Tasks

**Setup Architecture:**
- [x] Create `LLMResponse` dataclass
- [x] Create `ProviderConfig` dataclass
- [x] Create `LLMBackend` Protocol
- [x] Create `APIBackend` class with `_ensure_config()` containing current `get_llm_client()` logic
- [x] Create `_call_llm()` helper function
- [x] Refactor `get_llm_client()` to use `APIBackend` internally

**Refactoring generate_*() functions to use _call_llm():**
- [x] `generate_column_doc()`
- [x] `generate_table_doc()`
- [x] `generate_style_aware_column_doc()`
- [x] `generate_style_aware_table_doc()`
- [x] `generate_semantic_description()`
- [x] `generate_model_spec_as_json()`
- [x] `analyze_column_semantics()`
- [x] `generate_sql_from_nl()`
- [x] `generate_dbt_model_from_nl()`
- [x] `generate_staging_model_spec()`

**Finalization:**
- [x] Run existing tests to verify backwards compatibility (16 tests passed)

**Acceptance Criteria**:
- All existing tests pass without modification
- `--synthesize` flag works identically to current behavior
- No new dependencies required
- `get_llm_client()` returns same values as before

---

### Phase 2: CLI Backend Implementation

**Objective**: Implement CLI-based backend for executing LLM commands via Claude Code.

**Implementation Note**: Instead of custom subprocess handling for multiple CLIs, we used the official `claude-agent-sdk` Python package which provides a clean async API for Claude Code integration.

**Tasks**:
- [x] Add `claude-agent-sdk` as optional dependency (`pip install dbt-osmosis[claude-code]`)
- [x] Implement `_messages_to_prompt()` to flatten chat format to single string
- [x] Implement `ClaudeCodeBackend` class using `claude-agent-sdk`
- [x] Add `_get_backend()` for backend selection via `LLM_BACKEND` env var
- [x] Update `_call_llm()` to use `_get_backend()`
- [x] Add exports to `__all__`: `LLMBackend`, `LLMResponse`, `ProviderConfig`, `APIBackend`, `ClaudeCodeBackend`
- [x] Run tests to verify backwards compatibility (436 passed)

**Environment Variables**:
- `LLM_BACKEND`: Backend type (`api` or `claude-code`, default: `api`)

**Usage**:
```bash
# Install with Claude Code support
pip install dbt-osmosis[claude-code]

# Use Claude Code backend
export LLM_BACKEND=claude-code
dbt-osmosis yaml document --project-dir . --profiles-dir .
```

**Acceptance Criteria**:
- [x] Can generate documentation using Claude Code CLI
- [x] Graceful error handling when `claude-agent-sdk` not installed
- [x] Clear error messages for configuration issues
- [x] All existing tests pass (backwards compatibility)

**Estimated Effort**: 4-5 hours | **Actual**: ~2 hours

---

### Phase 3: Prompt Template System

**Objective**: Allow users to customize prompts via Jinja2 templates in their dbt project.

**Tasks**:
1. Create `PromptTemplateLoader` class using Jinja2's `Environment` and loaders
2. Define template file format (markdown with system/user prompt sections)
3. Convert existing hardcoded prompts to default templates
4. Implement template discovery (project → defaults fallback)
5. Integrate template rendering into `generate_*()` functions
6. Pass project directory through the call chain
7. Add optional custom Jinja2 filters for dbt-specific formatting

**Acceptance Criteria**:
- Default templates produce identical output to current hardcoded prompts
- Custom templates in `.dbt-osmosis/prompts/` are discovered and used
- Template syntax errors produce clear error messages
- All template variables are documented

**Estimated Effort**: 4-5 hours

---

### Phase 4: CLI Integration & User Experience

**Objective**: Expose new features via CLI options and provide tooling for users.

**Tasks**:
1. Add `--llm-backend` option (api/cli)
2. Add `--llm-cli` option (claude/copilot/llm/aider)
3. Add `--prompt-template-dir` option for custom template location
4. Create `dbt-osmosis init-prompts` command to scaffold templates
5. Create `dbt-osmosis list-templates` command to show active templates
6. Update help text and documentation
7. Add examples to README

**Acceptance Criteria**:
- All new options are documented in `--help`
- `init-prompts` creates working template files
- Users can override settings via CLI or environment variables

**Estimated Effort**: 2-3 hours

---

### Phase 5: Documentation & Release

**Objective**: Document features and prepare for release.

**Tasks**:
1. Update README with new features
2. Add documentation page for CLI backends
3. Add documentation page for custom templates
4. Create migration guide (if any breaking changes)
5. Add CHANGELOG entries
6. Create example templates for common use cases

**Estimated Effort**: 2-3 hours

---

## Risks & Considerations

### Technical Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| CLI output format varies across tools | Medium | High | Implement tool-specific parsers; add output normalization layer |
| CLI tools may require interactive authentication | High | Medium | Document requirements; detect and warn users; support auth token env vars |
| Subprocess execution security concerns | Medium | Low | Sanitize inputs; don't pass user content directly to shell; use list-based subprocess calls |
| Jinja2 template errors crash the process | Medium | Medium | Wrap template rendering in try/catch; provide helpful error messages with line numbers |
| Template injection vulnerabilities | Low | Low | Templates are user-controlled files in their own project; document that templates execute Jinja2 code |

### Compatibility Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Breaking existing `--synthesize` behavior | High | Low | Extensive backwards compatibility testing; default to current API behavior |
| Environment variable conflicts | Medium | Low | Use prefixed env vars; document all variables |
| Different behavior across LLM providers | Medium | Medium | Document expected outputs; add response normalization |

### Operational Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| CLI backend is slower than API | Low | High | Document performance expectations; CLI tools handle their own optimization |
| Users create templates with syntax errors | Low | High | Validate templates on load; provide `validate-templates` command |
| Rate limiting from CLI tools | Medium | Medium | Document; add configurable delays between calls |

### Scope Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Supporting too many CLI tools | Medium | Medium | Start with 1-2 tools; add plugin system for others |
| Template system feature creep | Medium | Medium | Keep initial scope minimal; use standard Jinja2 features only |

---

## Testing Strategy

### Unit Tests

**Backend Abstraction**:
- `APIBackend` correctly initializes for each provider (openai, azure, ollama, etc.)
- `CLIBackend` correctly builds commands for each CLI tool
- `get_llm_backend()` returns correct backend based on environment
- Backend `is_available()` correctly detects configuration

**Template System**:
- Template loader discovers project templates
- Template loader falls back to defaults when project templates don't exist
- Jinja2 rendering works with all expected variables
- Template parsing correctly splits system/user prompts
- Invalid templates produce meaningful errors

### Integration Tests

**API Backend** (existing, ensure no regression):
- Mock OpenAI API responses
- Verify request format matches expected
- Verify response parsing handles edge cases

**CLI Backend**:
- Mock subprocess calls
- Verify command construction for each CLI tool
- Verify output parsing handles various formats
- Verify error handling for failed commands

**Template Integration**:
- End-to-end test with custom template directory
- Verify variables are passed correctly through the stack
- Test with missing/malformed templates

### Backwards Compatibility Tests

**Critical**: These tests ensure existing users are not affected.

1. **Default Behavior Test**
   - Run `dbt-osmosis yaml refactor --synthesize` without any new env vars
   - Verify behavior is identical to current version
   - Compare output against known baseline

2. **Environment Variable Test**
   - Verify all existing env vars still work:
     - `LLM_PROVIDER`
     - `OPENAI_API_KEY`
     - `OPENAI_MODEL`
     - `AZURE_OPENAI_*`
     - `OLLAMA_*`
     - `OSMOSIS_LLM_MAX_SQL_CHARS`
   
3. **Output Format Test**
   - Generate documentation for sample models
   - Verify JSON output structure unchanged
   - Verify column documentation format unchanged

4. **Error Handling Test**
   - Missing API key produces same error message
   - Invalid provider produces same error message

### Manual Testing Checklist

- [ ] Fresh install of dbt-osmosis works with `--synthesize`
- [ ] Existing dbt project with no `.dbt-osmosis/` folder works unchanged
- [ ] `dbt-osmosis init-prompts` creates valid templates
- [ ] Custom templates are used when present
- [ ] CLI backend works with `claude` installed
- [ ] CLI backend produces helpful error when `claude` not installed
- [ ] Invalid template produces clear error message
- [ ] Documentation examples work as written

### Performance Testing

- Compare execution time: API backend vs baseline
- Compare execution time: CLI backend vs API backend
- Measure overhead of template loading (should be negligible)

---

## Success Metrics

1. **Backwards Compatibility**: 100% of existing tests pass without modification
2. **CLI Backend Adoption**: Users can successfully use at least 2 CLI tools
3. **Template Customization**: Users can create and use custom templates without errors
4. **Documentation Quality**: All new features documented with examples
5. **Error Handling**: All error paths produce actionable messages

---

## Dependencies & Prerequisites

### No New Python Dependencies Required

- **Jinja2**: Already installed (dbt-core dependency)
- **subprocess**: Python standard library
- **abc**: Python standard library

### External Tool Dependencies (for CLI backend)

Users must install their chosen CLI tool separately:
- `claude`: Install via Anthropic's instructions
- `gh copilot`: Install via GitHub CLI
- `llm`: `pip install llm`
- `aider`: `pip install aider-chat`

---

## Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Backend Abstraction | 3-4 hours | None |
| Phase 2: CLI Backend | 4-5 hours | Phase 1 |
| Phase 3: Template System | 4-5 hours | Phase 1 |
| Phase 4: CLI Integration | 2-3 hours | Phases 2, 3 |
| Phase 5: Documentation | 2-3 hours | Phase 4 |
| **Total** | **15-20 hours** | |

Phases 2 and 3 can be developed in parallel after Phase 1 is complete.

---

## Future Considerations

Items explicitly **out of scope** for this project but worth considering later:

1. **Plugin System for CLI Tools**: Allow third-party CLI tool support via plugins
2. **Template Inheritance**: Base templates that project templates can extend
3. **Template Validation Command**: `dbt-osmosis validate-templates`
4. **Interactive Mode**: Preview generated docs before committing
5. **Caching**: Cache LLM responses to avoid regenerating unchanged models
6. **Async CLI Execution**: Parallel subprocess calls for better performance
7. **Template Variables from dbt_project.yml**: Allow users to define custom variables

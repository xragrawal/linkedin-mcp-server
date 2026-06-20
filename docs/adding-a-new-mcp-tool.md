# Adding a New MCP Tool

Use this guide when adding a new tool to the LinkedIn MCP server, such as
`search_posts`, `get_company_jobs`, or another LinkedIn workflow exposed through
FastMCP.

Before implementing, read `CONTRIBUTING.md` and open or find a GitHub issue for
the feature. New tools should follow the repository workflow: branch from
`main`, implement and test, update docs and metadata, then open a draft PR.

## 1. Define the Tool Contract

Decide the user-facing behavior before changing code.

- Tool name, using snake_case, for example `search_posts`
- Arguments and validation, for example `keywords: str` or `max_results: int`
- LinkedIn URL or flow the tool will use
- Whether the tool is read-only or performs an action
- Return shape

Prefer the standard return format:

```python
{
    "url": "...",
    "sections": {
        "search_results": "raw LinkedIn text..."
    },
    "references": {
        "search_results": [...]
    }
}
```

All scraping tools should keep `sections` as the main readable payload. Use
optional keys such as `references`, `section_errors`, `unknown_sections`, or
tool-specific fields only when they add clear value.

## 2. Choose Where the Tool Belongs

Add the tool wrapper to the closest existing module:

- Person tools: `linkedin_mcp_server/tools/person.py`
- Company tools: `linkedin_mcp_server/tools/company.py`
- Job tools: `linkedin_mcp_server/tools/job.py`
- Feed tools: `linkedin_mcp_server/tools/feed.py`
- Messaging tools: `linkedin_mcp_server/tools/messaging.py`

If the tool needs a new category, create a new `tools/*.py` file with a
`register_*_tools()` function and call it from
`linkedin_mcp_server/server.py`.

## 3. Implement the Extractor Logic

Put browser and scraping behavior in
`linkedin_mcp_server/scraping/extractor.py` unless an existing helper already
fits.

Follow the scraping rules:

- One section equals one navigation.
- Prefer URL navigation and `innerText` extraction.
- Avoid LinkedIn layout class names.
- Use only minimal generic selectors when DOM access is unavoidable.
- Detection logic must be locale-independent.

For read-only tools, use annotations like:

```python
annotations={"readOnlyHint": True, "openWorldHint": True}
```

For tools that send messages, connect with users, or otherwise perform actions,
use `destructiveHint` so MCP clients can ask for confirmation.

## 4. Register the Tool

Tool wrappers should follow the existing FastMCP pattern:

```python
@mcp.tool(
    timeout=tool_timeout,
    title="Search Posts",
    annotations={"readOnlyHint": True, "openWorldHint": True},
    tags={"feed", "search"},
    exclude_args=["extractor"],
)
async def search_posts(
    keywords: str,
    ctx: Context,
    extractor: Any | None = None,
) -> dict[str, Any]:
    try:
        extractor = extractor or await get_ready_extractor(
            ctx, tool_name="search_posts"
        )
        await ctx.report_progress(
            progress=0, total=100, message="Starting post search"
        )
        result = await extractor.search_posts(keywords)
        await ctx.report_progress(progress=100, total=100, message="Complete")
        return result
    except AuthenticationError as e:
        try:
            await handle_auth_error(e, ctx)
        except Exception as relogin_exc:
            raise_tool_error(relogin_exc, "search_posts")
    except Exception as e:
        raise_tool_error(e, "search_posts")
```

If the tool lives in an existing module, it is registered automatically by that
module's `register_*_tools()` function. If it lives in a new module, import and
call the registration function in `create_mcp_server()`.

## 5. Add Tests

Update the mock extractor in `tests/test_tools.py` with the new async method.
Then add tool-level tests that verify:

- The tool is registered.
- Arguments are forwarded to the extractor.
- The returned payload keeps the standard shape.
- Auth and tool errors are handled consistently.

Add extractor or scraping tests in `tests/test_scraping.py` when the tool adds a
new LinkedIn URL pattern, navigation flow, reference extraction, or section
handling.

## 6. Update Docs and Metadata

Update all user-facing places that list tools:

- `README.md`
- `docs/docker-hub.md`
- `manifest.json`

Keep tool descriptions concise and aligned with the actual return format.

## 7. Verify Locally

Run the standard checks:

```bash
uv run pytest --cov
uv run ruff check . --fix
uv run ruff format .
uv run ty check
```

For scraping behavior, verify end-to-end against live LinkedIn using the local
workspace server, not the packaged `uvx` command:

```bash
uv run -m linkedin_mcp_server --transport streamable-http --log-level DEBUG
```

Then initialize an MCP session and call the new tool over HTTP. Keep any live
verification notes in the PR so reviewers can see the exact behavior tested.

## 8. Prepare the PR

Before opening the PR, include a synthetic prompt in the PR description:

```markdown
## Synthetic prompt

> Add a new `search_posts` MCP tool that searches LinkedIn posts by keyword,
> returns raw search result text and post references, and updates tests, docs,
> and manifest metadata.

Generated with <model name and version>
```

Open the PR as a draft first. Run AI review before requesting manual review, and
do not squash commits when merging.

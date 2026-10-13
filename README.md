# Knowledge Base Manager

Open WebUI tool for managing knowledge bases, directories, and files directly from chat. Built for keeping technical documentation up to date: home-server setups, service configs, VPN notes, install logs, anything you want the model to consult later and update as things change.

The model can list knowledge bases, browse the directory tree, read files, create and update Markdown documents at nested paths, create or reuse directory structures, rename, move, and delete files and directories, and search for files across every knowledge base the user can access.

## Features

- 24 functions covering full CRUD operations
- Nested directory management with automatic path creation
- Race-safe directory creation (concurrent calls cannot create duplicate folders)
- Search with content preview
- Safety features (`confirm=true` for deletions, nonempty checks)
- Upsert operations with duplicate prevention
- Recursive tree navigation and duplicate-directory diagnostics
- Synchronous indexing for reliable content availability

## Installation

### Main method (recommended)

1. Open the [Knowledge Base Manager tool page](https://openwebui.com/posts/knowledge_base_manager_e1b2a255) on Open WebUI Community
2. Click the **Get** button
3. The tool will be automatically installed to your Open WebUI instance

### Alternative method (manual)

If the Get button doesn't work or you prefer manual installation:

1. Download or copy the code from `knowledge_base_manager.py`
2. In Open WebUI, go to **Workspace → Tools → +**
3. Paste the code and save

## Requirements

- No API key needed. The tool reuses the Authorization header and cookies of the current chat request, so it acts as the signed-in user with that user's permissions.
- Open WebUI must be reachable at the address set in `self.base_url` (default `http://127.0.0.1:8080/api/v1`). Change it if your instance runs on a different host or port.
- The target knowledge base needs write access for the current user.

## How it works

File creation and updates run synchronously: the tool uploads with `process_in_background=false` and waits until Open WebUI finishes extraction and indexing before reporting success. Large documents take longer, but a success message means the file is ready.

Directory listings read all pages, so nested and paginated content is not missed. Path-based functions resolve directories level by level from the root, and an ambiguous match stops the operation instead of picking a file at random.

Directory creation is serialized with a process-wide lock and is idempotent: when a create call fails because the level appeared in the meantime, the tool re-reads the level and reuses the existing directory. Two concurrent calls that share a not-yet-existing path therefore cannot leave two same-name folders behind.

## Search

`search_knowledge_files` matches the query against filenames. With `include_content=true` it also matches each file's stored extracted text and returns a preview, fetching one extra request per hit, capped by `max_content_items` (default 10, max 50).

Notes:

- the match is a case-insensitive substring match, not semantic or relevance-ranked;
- content matching only covers the first `RAG_FILE_CONTENT_SEARCH_MAX_CHARS` characters of each file (64 MiB by default);
- a file placed in several knowledge bases is returned once per knowledge base;
- for semantic retrieval over document text use Open WebUI's normal knowledge retrieval, or read a known file directly.

## Safety

- Deletions require `confirm=true`.
- Deleting a knowledge base is refused when it still contains files anywhere in the tree (checked recursively across all nested directories), unless `allow_nonempty=true` is passed.
- Upsert never falls back to creating a file when the existence check fails due to an API error, timeout, or ambiguous match, which avoids duplicates.
- Ambiguous matches are never guessed: same-name directories report their `directory_id`s, same-name files report their `file_id`s.

## Duplicate directories

Open WebUI describes a directory as `UNIQUE(knowledge_id, parent_id, name)`, but at the root level `parent_id` is `NULL`, and `NULL` never equals another `NULL` in that unique index, so the database cannot reject a duplicate root folder. Older versions of this tool could produce such duplicates under concurrent calls; the current version serializes directory creation to prevent that.

`find_duplicate_directories` is a read-only helper that reports same-name directories at the same level, their `directory_id`s and how many files each copy holds. Duplicates are not merged automatically — move the files you want to keep to one directory and delete the leftover empty one. Until then, any path-based function will refuse the ambiguous path instead of guessing.

## Tools

| Function | Purpose |
| --- | --- |
| `list_knowledge_bases` | List accessible knowledge bases with IDs and write access |
| `list_knowledge_files` | List files and subdirectories of one directory |
| `get_knowledge_tree` | Recursive file/directory tree of a knowledge base |
| `create_knowledge_base` | Create a knowledge base |
| `update_knowledge_base` | Rename / re-describe a knowledge base |
| `delete_knowledge_base` | Delete a knowledge base (`confirm`, nonempty check) |
| `search_knowledge_files` | Search across all accessible knowledge bases |
| `find_and_read_knowledge_file` | Find a file and return its full text |
| `read_knowledge_file` / `read_knowledge_file_at_path` | Read a file by ID or path |
| `create_knowledge_markdown` / `create_knowledge_markdown_at_path` | Create a Markdown file |
| `update_knowledge_file` / `update_knowledge_file_at_path` | Replace a file's text and reindex |
| `upsert_knowledge_markdown_at_path` | Create or update a Markdown file at a path |
| `create_knowledge_directory` | Create a directory |
| `ensure_knowledge_directory_path` | Create or reuse a nested directory path |
| `rename_knowledge_directory` | Rename a directory |
| `move_knowledge_directory` | Move a directory (empty target = root) |
| `delete_knowledge_directory` | Delete a directory (files move to parent) |
| `find_duplicate_directories` | Report same-name directories at the same level |
| `rename_knowledge_file` | Rename a file |
| `delete_knowledge_file` / `delete_knowledge_file_at_path` | Delete a file |

## Tested

Tested against a live Open WebUI 0.12.0 instance across multiple rounds, with all functions exercised end to end, including:

- nested paths and pagination across large directory trees
- ambiguous matches (refused instead of guessed, with the conflicting IDs listed)
- `confirm=false` correctly refusing deletion for files, path-based deletion, and knowledge bases
- `delete_knowledge_base` correctly detecting files nested several levels deep before refusing deletion
- `upsert` correctly avoiding duplicate creation on API errors (e.g. invalid `knowledge_id`)
- `search_knowledge_files` matching by filename and, with `include_content=true`, by document text
- `max_content_items` correctly limiting the number of previews fetched
- concurrent creation of a shared path: the current version leaves exactly one directory where the previous version left two
- full cleanup after each test run, with no impact on pre-existing knowledge bases

## Links

- Community post: https://openwebui.com/posts/knowledge_base_manager_e1b2a255

## License

MIT License

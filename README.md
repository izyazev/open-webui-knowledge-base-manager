# Knowledge Base Manager

Open WebUI tool for managing knowledge bases, directories, and files directly from chat. Built for keeping technical documentation up to date: home-server setups, service configs, VPN notes, install logs, anything you want the model to consult later and update as things change.

The model can list knowledge bases, browse the directory tree, read files, create and update Markdown documents at nested paths, create or reuse directory structures, rename, move, and delete files and directories, and search for files across every knowledge base the user can access.

## Features

- 23 functions covering full CRUD operations
- Nested directory management with automatic path creation
- Search with content preview
- Safety features (`confirm=true` for deletions, nonempty checks)
- Upsert operations with duplicate prevention
- Recursive tree navigation
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

## Safety

- Deletions require `confirm=true`.
- Deleting a knowledge base is refused when it still contains files anywhere in the tree (checked recursively across all nested directories), unless `allow_nonempty=true` is passed.
- Upsert never falls back to creating a file when the existence check fails due to an API error, timeout, or ambiguous match, which avoids duplicates.

## Known limitation

The Open WebUI search endpoint matches filenames and metadata, not document text, so `search_knowledge_files` will not find a file by a phrase inside it. With `include_content=true` the tool fetches a content preview from each result in a separate request; `max_content_items` (default 10, max 50) caps how many previews are fetched. For content-based retrieval, use Open WebUI's normal knowledge retrieval or read a known file directly.

## Tested

Tested against a live Open WebUI instance across multiple rounds, with all 23 functions exercised end to end, including:

- nested paths and pagination across large directory trees
- ambiguous matches (refused instead of guessed)
- `confirm=false` correctly refusing deletion for files, path-based deletion, and knowledge bases
- `delete_knowledge_base` correctly detecting files nested several levels deep before refusing deletion
- `upsert` correctly avoiding duplicate creation on API errors (e.g. invalid `knowledge_id`)
- `search_knowledge_files` with `include_content=true` returning real content previews
- `max_content_items` correctly limiting the number of previews fetched
- full cleanup after each test run, with no impact on pre-existing knowledge bases

## Links

- Community post: https://openwebui.com/posts/knowledge_base_manager_e1b2a255

## License

MIT License

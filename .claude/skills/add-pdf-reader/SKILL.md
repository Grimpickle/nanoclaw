---
name: add-pdf-reader
description: Add PDF reading to NanoClaw agents. Extracts text from PDFs via pdftotext CLI. Handles Telegram document attachments, URLs, and local files.
---

# Add PDF Reader

Adds PDF reading capability to all container agents using poppler-utils (pdftotext/pdfinfo). PDFs sent as Telegram document messages are auto-downloaded to the group workspace.

## Phase 1: Pre-flight

1. Check if `container/skills/pdf-reader/pdf-reader` exists — skip to Phase 3 if already applied
2. Confirm Telegram is installed (`src/channels/telegram.ts` exists)

## Phase 2: Apply Code Changes

### 2a. Add poppler-utils to the container Dockerfile

Read `container/Dockerfile` and add `poppler-utils \` to the `apt-get install` list (alongside `git`), and after the TypeScript build step add:

```dockerfile
# Install pdf-reader CLI
COPY skills/pdf-reader/pdf-reader /usr/local/bin/pdf-reader
RUN chmod +x /usr/local/bin/pdf-reader
```

### 2b. Create container/skills/pdf-reader/SKILL.md

Create `container/skills/pdf-reader/SKILL.md`:

```markdown
---
name: pdf-reader
description: Read and extract text from PDF files — documents, reports, contracts, spreadsheets. Use whenever you need to read PDF content, not just when explicitly asked. Handles local files, URLs, and Telegram attachments.
allowed-tools: Bash(pdf-reader:*)
---

# PDF Reader

## Quick start

\`\`\`bash
pdf-reader extract report.pdf              # Extract all text
pdf-reader extract report.pdf --layout     # Preserve tables/columns
pdf-reader fetch https://example.com/doc.pdf  # Download and extract
pdf-reader info report.pdf                 # Show metadata + size
pdf-reader list                            # List all PDFs in directory tree
\`\`\`

## Commands

### extract — Extract text from PDF

\`\`\`bash
pdf-reader extract <file>                        # Full text to stdout
pdf-reader extract <file> --layout               # Preserve layout (tables, columns)
pdf-reader extract <file> --pages 1-5            # Pages 1 through 5
pdf-reader extract <file> --pages 3-3            # Single page (page 3)
pdf-reader extract <file> --layout --pages 2-10  # Layout + page range
\`\`\`

Options:
- `--layout` — Maintains spatial positioning. Essential for tables, spreadsheets, multi-column docs.
- `--pages N-M` — Extract only pages N through M (1-based, inclusive).

### fetch — Download and extract PDF from URL

\`\`\`bash
pdf-reader fetch <url>                    # Download, verify, extract with layout
pdf-reader fetch <url> report.pdf         # Also save a local copy
\`\`\`

Downloads the PDF, verifies it has a valid `%PDF` header, then extracts text with layout preservation. Temporary files are cleaned up automatically.

### info — PDF metadata and file size

\`\`\`bash
pdf-reader info <file>
\`\`\`

Shows title, author, page count, page size, PDF version, and file size on disk.

### list — Find all PDFs in directory tree

\`\`\`bash
pdf-reader list
\`\`\`

Recursively lists all `.pdf` files with page count and file size.

## Telegram PDF attachments

When a user sends a PDF on Telegram, it is automatically saved to the `attachments/` directory. The message will include a path hint like:

> [PDF: attachments/document.pdf (42KB)]
> Use: pdf-reader extract attachments/document.pdf

To read the attached PDF:

\`\`\`bash
pdf-reader extract attachments/document.pdf --layout
\`\`\`

## Example workflows

### Read a contract and summarize key terms

\`\`\`bash
pdf-reader info attachments/contract.pdf
pdf-reader extract attachments/contract.pdf --layout
\`\`\`

### Extract specific pages from a long report

\`\`\`bash
pdf-reader info report.pdf                    # Check total pages
pdf-reader extract report.pdf --pages 1-3     # Executive summary
pdf-reader extract report.pdf --pages 15-20   # Financial tables
\`\`\`

### Fetch and analyze a public document

\`\`\`bash
pdf-reader fetch https://example.com/annual-report.pdf report.pdf
pdf-reader info report.pdf
\`\`\`
```

### 2c. Create container/skills/pdf-reader/pdf-reader

Create `container/skills/pdf-reader/pdf-reader` (executable bash script):

```bash
#!/bin/bash
set -euo pipefail

# pdf-reader — CLI wrapper around poppler-utils (pdftotext, pdfinfo)
# Provides extract, fetch, info, list commands for PDF processing.

VERSION="1.0.0"

usage() {
  cat <<'USAGE'
pdf-reader — Extract text and metadata from PDF files

Usage:
  pdf-reader extract <file> [--layout] [--pages N-M]
  pdf-reader fetch <url> [filename]
  pdf-reader info <file>
  pdf-reader list
  pdf-reader help

Commands:
  extract   Extract text from a PDF file to stdout
  fetch     Download a PDF from a URL and extract text
  info      Show PDF metadata and file size
  list      List all PDFs in current directory tree
  help      Show this help message

Extract options:
  --layout  Preserve original layout (tables, columns)
  --pages   Page range to extract (e.g. 1-5, 3-3 for single page)
USAGE
}

cmd_extract() {
  local file=""
  local layout=false
  local first_page=""
  local last_page=""

  while [[ $# -gt 0 ]]; do
    case "$1" in
      --layout)
        layout=true
        shift
        ;;
      --pages)
        if [[ -z "${2:-}" ]]; then
          echo "Error: --pages requires a range argument (e.g. 1-5)" >&2
          exit 1
        fi
        local range="$2"
        first_page="${range%-*}"
        last_page="${range#*-}"
        shift 2
        ;;
      -*)
        echo "Error: Unknown option: $1" >&2
        exit 1
        ;;
      *)
        if [[ -z "$file" ]]; then
          file="$1"
        else
          echo "Error: Unexpected argument: $1" >&2
          exit 1
        fi
        shift
        ;;
    esac
  done

  if [[ -z "$file" ]]; then
    echo "Error: No file specified" >&2
    echo "Usage: pdf-reader extract <file> [--layout] [--pages N-M]" >&2
    exit 1
  fi

  if [[ ! -f "$file" ]]; then
    echo "Error: File not found: $file" >&2
    exit 1
  fi

  local args=()
  if [[ "$layout" == true ]]; then
    args+=(-layout)
  fi
  if [[ -n "$first_page" ]]; then
    args+=(-f "$first_page")
  fi
  if [[ -n "$last_page" ]]; then
    args+=(-l "$last_page")
  fi

  pdftotext ${args[@]+"${args[@]}"} "$file" -
}

cmd_fetch() {
  local url="${1:-}"
  local filename="${2:-}"

  if [[ -z "$url" ]]; then
    echo "Error: No URL specified" >&2
    echo "Usage: pdf-reader fetch <url> [filename]" >&2
    exit 1
  fi

  local tmpfile
  tmpfile="$(mktemp /tmp/pdf-reader-XXXXXX.pdf)"
  trap 'rm -f "$tmpfile"' EXIT

  echo "Downloading: $url" >&2
  if ! curl -sL -o "$tmpfile" "$url"; then
    echo "Error: Failed to download: $url" >&2
    exit 1
  fi

  local header
  header="$(head -c 4 "$tmpfile")"
  if [[ "$header" != "%PDF" ]]; then
    echo "Error: Downloaded file is not a valid PDF (header: $header)" >&2
    exit 1
  fi

  if [[ -n "$filename" ]]; then
    cp "$tmpfile" "$filename"
    echo "Saved to: $filename" >&2
  fi

  pdftotext -layout "$tmpfile" -
}

cmd_info() {
  local file="${1:-}"

  if [[ -z "$file" ]]; then
    echo "Error: No file specified" >&2
    echo "Usage: pdf-reader info <file>" >&2
    exit 1
  fi

  if [[ ! -f "$file" ]]; then
    echo "Error: File not found: $file" >&2
    exit 1
  fi

  pdfinfo "$file"
  echo ""
  echo "File size:    $(du -h "$file" | cut -f1)"
}

cmd_list() {
  local found=false

  shopt -s nullglob globstar

  declare -A seen
  for pdf in *.pdf **/*.pdf; do
    [[ -v seen["$pdf"] ]] && continue
    seen["$pdf"]=1
    found=true

    local pages="?"
    local size
    size="$(du -h "$pdf" | cut -f1)"

    if page_line="$(pdfinfo "$pdf" 2>/dev/null | grep '^Pages:')"; then
      pages="$(echo "$page_line" | awk '{print $2}')"
    fi

    printf "%-60s %5s pages  %8s\n" "$pdf" "$pages" "$size"
  done

  if [[ "$found" == false ]]; then
    echo "No PDF files found in current directory tree." >&2
  fi
}

command="${1:-help}"
shift || true

case "$command" in
  extract) cmd_extract "$@" ;;
  fetch)   cmd_fetch "$@" ;;
  info)    cmd_info "$@" ;;
  list)    cmd_list ;;
  help|--help|-h) usage ;;
  version|--version|-v) echo "pdf-reader $VERSION" ;;
  *)
    echo "Error: Unknown command: $command" >&2
    echo "Run 'pdf-reader help' for usage." >&2
    exit 1
    ;;
esac
```

After creating the file, make it executable:

```bash
chmod +x container/skills/pdf-reader/pdf-reader
```

### 2d. Patch src/channels/telegram.ts

Add `import fs from 'fs';` to the imports at the top of the file (alongside the existing `import path from 'path';`).

Replace the existing `message:document` handler:

```typescript
this.bot.on('message:document', (ctx) => {
  const name = ctx.message.document?.file_name || 'file';
  storeNonText(ctx, `[Document: ${name}]`);
});
```

With this expanded handler:

```typescript
this.bot.on('message:document', async (ctx) => {
  const doc = ctx.message.document;
  const chatJid = `tg:${ctx.chat.id}`;
  const group = this.opts.registeredGroups()[chatJid];
  if (!group) return;

  const timestamp = new Date(ctx.message.date * 1000).toISOString();
  const senderName =
    ctx.from?.first_name ||
    ctx.from?.username ||
    ctx.from?.id?.toString() ||
    'Unknown';
  const caption = ctx.message.caption || '';
  const isGroup =
    ctx.chat.type === 'group' || ctx.chat.type === 'supergroup';
  this.opts.onChatMetadata(
    chatJid,
    timestamp,
    undefined,
    'telegram',
    isGroup,
  );

  let content: string;

  if (doc.mime_type === 'application/pdf') {
    try {
      const file = await ctx.api.getFile(doc.file_id);
      if (!file.file_path) throw new Error('No file_path returned');
      const url = `https://api.telegram.org/file/bot${this.botToken}/${file.file_path}`;
      const buffer = await downloadBuffer(url);
      const groupDir = path.join(GROUPS_DIR, group.folder);
      const attachDir = path.join(groupDir, 'attachments');
      fs.mkdirSync(attachDir, { recursive: true });
      const filename = doc.file_name || `doc-${Date.now()}.pdf`;
      const filePath = path.join(attachDir, filename);
      fs.writeFileSync(filePath, buffer);
      const sizeKB = Math.round(buffer.length / 1024);
      const pdfRef = `[PDF: attachments/${filename} (${sizeKB}KB)]\nUse: pdf-reader extract attachments/${filename}`;
      content = caption ? `${caption}\n\n${pdfRef}` : pdfRef;
      logger.info(
        { chatJid, filename },
        'Downloaded Telegram PDF attachment',
      );
    } catch (err) {
      logger.warn(
        { chatJid, err },
        'Failed to download Telegram PDF attachment',
      );
      const name = doc.file_name || 'document.pdf';
      content = caption ? `[PDF: ${name}] ${caption}` : `[PDF: ${name}]`;
    }
  } else {
    const name = doc.file_name || 'file';
    content = caption ? `[Document: ${name}] ${caption}` : `[Document: ${name}]`;
  }

  this.opts.onMessage(chatJid, {
    id: ctx.message.message_id.toString(),
    chat_jid: chatJid,
    sender: ctx.from?.id?.toString() || '',
    sender_name: senderName,
    content,
    timestamp,
    is_from_me: false,
  });
});
```

Note: `GROUPS_DIR` is already imported in telegram.ts from `'../config.js'`.

### 2e. Validate

```bash
npm run build
npm test
```

### 2f. Rebuild container

```bash
./container/build.sh
```

### 2g. Restart service

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 3: Verify

### Test PDF extraction

Send a PDF file in any registered Telegram chat. The agent should:
1. Download the PDF to `attachments/`
2. Show a message like `[PDF: attachments/document.pdf (42KB)]`
3. Be able to extract text when asked

### Test URL fetching

Ask the agent to read a PDF from a URL. It should use `pdf-reader fetch <url>`.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log | grep -i pdf
```

Look for:
- `Downloaded Telegram PDF attachment` — successful download
- `Failed to download Telegram PDF attachment` — media download issue

## Troubleshooting

### Agent says pdf-reader command not found

Container needs rebuilding. Run `./container/build.sh` and restart the service.

### PDF text extraction is empty

The PDF may be scanned (image-based). pdftotext only handles text-based PDFs. Consider using the agent-browser to open the PDF visually instead.

### Telegram PDF not detected

Verify the file was sent as a document (not compressed). Telegram sometimes compresses files — sending as "File" (not "Photo") ensures it arrives as a document with the correct mime type.

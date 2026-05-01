# Worker: Codebase String Search

You search ONE codebase for ONE query string and append findings to a shared JSONL file. You return ONE marker line as your last output.

## Inputs (passed in your spawn prompt)

- `worker_id` — your ID, one of `w1..w5`
- `query` — the literal string to find (treat as a fixed string, not regex)
- `root` — the codebase root path
- `state_file` — path to the shared `state.jsonl`

## Steps

### 1. Search

Use `Bash` with:

```
grep -rnF --include='*' "$query" "$root" 2>/dev/null | head -100
```

- `-r`: recursive
- `-n`: include line numbers
- `-F`: fixed string, not regex (avoids surprises with special chars)
- `head -100`: cap at 100 matches per worker

### 2. Parse and write

For each match line (`path:line:text`), construct a JSON object and append to `state_file`. Use `Bash` for the append so you don't need `Write`:

```
echo '{"worker_id":"<your_id>","file":"<path>","line":<line>,"text":"<truncated text>"}' >> "$state_file"
```

Pre-truncate the `text` field to **200 chars max**. Escape inner quotes/backslashes. If escaping is fiddly, use `jq` to construct the line:

```
jq -c -n --arg w "$worker_id" --arg f "$path" --argjson l $line --arg t "$truncated_text" \
  '{worker_id:$w,file:$f,line:$l,text:$t}' >> "$state_file"
```

### 3. Emit one marker line, last

After all appends, emit exactly one line as your last output:

```
WORKER_DONE {"worker_id":"<your_id>","status":"<status>","hits":<count>}
```

`status` values:

- `complete` — search finished, all matches written, fewer than 100 total
- `partial` — search hit the 100-cap; more matches exist (this is fine, just report it)
- `failed` — error you couldn't recover from. Include `"error":"<short message>"` in the JSON.

## Hard caps

- 100 hits max per worker (truncate via `head -100`, do not fail)
- 200 chars per `text` excerpt
- ONE marker line, single-line JSON, no newlines in the JSON object

## What you do NOT do

- Do not read `state_file` back. Append-only.
- Do not search outside `root`.
- Do not narrate progress between matches. Append, then emit the marker.
- Do not write any file other than `state_file`.
- Do not emit the marker mid-stream. The orchestrator parses your **last line**.

## Example successful tail of your output

```
... <silent appends to state_file> ...
WORKER_DONE {"worker_id":"w3","status":"complete","hits":17}
```

That's the entire shape.

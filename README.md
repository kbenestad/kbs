# kbs

A simple filename and content search tool for the command line. Built as a
faster-to-use alternative to typing raw `grep`/`find` invocations for
everyday searches — supports OR terms, wildcards, file-type filters, tree
output, and surfacing surrounding context around a content match.

## Install

One-line install:

```
curl -fsSL https://raw.githubusercontent.com/kbenestad/kbs/refs/heads/main/kbs -o ~/.local/bin/kbs
chmod +x ~/.local/bin/kbs

```

```sh
mkdir -p ~/.local/bin
curl -fsSL {CANONICALPATHHERE} -o ~/.local/bin/kbs
chmod +x ~/.local/bin/kbs
```

Make sure `~/.local/bin` is on your `PATH`:

```sh
echo $PATH | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

(Use `~/.zshrc` instead if your shell is zsh.)

Requires Python 3. No other dependencies.

## Usage

```
kbs SEARCHTERM [flags]
```

By default, `kbs` searches **filenames** (not contents), starting in the
current directory and recursing into all subfolders.

### Basic search

```sh
kbs report
```

Finds any filename containing "report", anywhere below the current directory.

### Multiple terms (OR)

Separate terms with `|`. **Quote the whole argument** — an unquoted `|` is a
shell pipe operator, not a search separator, and bash will misinterpret it.

```sh
kbs "invoice | receipt"
```

### Wildcards

`*` matches any sequence of characters. **Quote the argument** — an unquoted
`*` gets expanded by bash itself before `kbs` ever sees it.

```sh
kbs "draft*"
```

### Search file contents instead of filenames

```sh
kbs -c "budget"
```

Prints each matching file, followed by which of your search term(s) were
found in it:

```
./finance/q1.md
budget

./notes/meeting.txt
budget | forecast
```

### Restrict to the current folder only (no recursion)

```sh
kbs report -r
```

### Filter by file type

```sh
kbs report -p office   # office/doc formats: .doc .docx .odt .rtf .md .txt .csv .pdf
kbs report -p html     # .html .htm
kbs report -p dev      # common code files: .py .js .ts .go .rs .java .php .rb ...
```

Only one `-p` value at a time. New types can be added easily — see
"Adding a new file type" below.

### Tree view

```sh
kbs report -t
```

Prints matches as an indented tree instead of a flat list.

### Show context around a content match

Requires `-c`. Shows N words before and after the match (default 8):

```sh
kbs "budget" -c -s
kbs "budget" -c -s 15
```

```
./finance/q1.md
we came in under the projected budget for the quarter overall and
```

### Combining flags

Flags combine freely:

```sh
kbs "invoice*" -c -p office -s 10
```

## Flag reference

| Flag | Long form | Meaning |
|---|---|---|
| `-c` | `--content` | Search file contents instead of filenames |
| `-r` | `--root` | Current folder only, don't recurse |
| `-p TYPE` | `--type TYPE` | Restrict to a file type: `office`, `html`, `dev` |
| `-t` | `--tree` | Print results as a tree |
| `-s [N]` | `--surface [N]` | Show N words of context around a content match (default 8); requires `-c` |

## Adding a new file type

Open `kbs` and find the `TYPES` dictionary near the top of the file:

```python
TYPES = {
    'office': {'.doc', '.docx', '.odt', '.rtf', '.md', '.markdown',
               '.pdf', '.txt', '.csv'},
    'html': {'.html', '.htm'},
    'dev': {'.py', '.js', '.jsx', '.ts', '.tsx', '.sh', '.bash', '.c', '.h',
            '.cpp', '.hpp', '.java', '.go', '.rs', '.php', '.rb', '.json',
            '.yaml', '.yml', '.css', '.sql'},
}
```

Add a new entry — for example, images:

```python
    'images': {'.jpg', '.jpeg', '.png', '.gif', '.svg'},
```

That's the only change needed. `kbs report -p images` works immediately,
and an invalid `-p` value (e.g. a typo) now gives a clear error rather than
silently matching nothing.

## Notes and caveats

- Matching is case-insensitive throughout.
- Content search skips files it can't decode as text (binaries, etc.) rather
  than erroring.
- `-s` (surface) shows only the **first** match per file, not every
  occurrence — keeps output readable for files with many hits.
- Always quote search terms containing `|` or `*`, or your shell will
  interpret them before `kbs` gets a chance to.

## License

Apache License, Version 2.0. See [LICENSE](LICENSE).

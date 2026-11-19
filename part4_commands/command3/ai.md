`wc -l access.log` counts the number of lines in `access.log`. Since a web server writes one line per request, that's effectively the **total number of requests** logged.

## Breaking it down

- `wc` stands for "word count," but it can count lines, words, and bytes.
- `-l` restricts the output to the **line count** only. Without any option, `wc` prints three numbers (lines, words, bytes).
- `access.log` is the input file.

## Example output

```
48213 access.log
```

That means the file has 48,213 lines. The filename is printed after the number because you passed it as an argument.

## Getting just the number

To print only the number (handy in scripts), have `wc` read from stdin so it has no filename to print:

```bash
wc -l < access.log
```

Output: `48213`

On some systems, notably macOS, the number may be padded with leading spaces.

## Caveats

- **It counts newline characters.** If the last line has no trailing newline, `wc -l` reports one fewer than the actual number of lines. Log files normally end with a newline, so this rarely matters.
- **Lines are not always requests.** Multi-line entries or malformed lines would inflate the count, but for a standard access log, one line equals one request.
- **Rotated logs.** This only covers the current file. For compressed rotated logs, use `zcat access.log.2.gz | wc -l`.
- **Multiple files.** `wc -l access.log access.log.1` prints a count per file plus a `total` line.
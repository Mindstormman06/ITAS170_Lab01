This pipeline finds the three busiest hours of the day in a web server access log. It chains five commands together, each feeding the next through a pipe (`|`).

## The assumed log format

It assumes Apache/Nginx "common log format," where a line looks like this:

```
192.168.1.5 - - [18/Nov/2026:14:32:07 -0800] "GET /index.html HTTP/1.1" 200 5120
```

Splitting that on spaces gives: field 1 is the IP, 2 and 3 are `-`, **field 4 is `[18/Nov/2026:14:32:07`**, field 5 is `-0800]`, and so on. This is the key to why the first command works.

## Stage 1: `cut -d' ' -f4 access.log`

- `cut` extracts sections from each line of input.
- `-d' '` sets the **delimiter** to a single space. The default is a tab, so you have to override it for space-separated logs.
- `-f4` selects **field 4** (the fourth chunk when split on that delimiter).
- `access.log` is the input file. Since a file is given, `cut` reads it directly, which is why there's no `cat` in front.

Output for each line: `[18/Nov/2026:14:32:07`

## Stage 2: `cut -d: -f2`

- `-d:` sets the delimiter to a colon. Note there's no space or quoting needed here, since `:` has no special shell meaning.
- `-f2` selects field 2.
- There's no filename, so `cut` reads from the pipe (stdin).

Splitting `[18/Nov/2026:14:32:07` on colons gives `[18/Nov/2026`, `14`, `32`, `07`. Field 2 is **`14`**, the hour. After this stage, the stream is just one hour value per request, like:

```
14
14
09
14
```

## Stage 3: `sort`

Sorts the lines alphabetically, which here also groups identical hours together. This step is required because `uniq` only collapses *adjacent* duplicates. Without `sort`, the same hour appearing in separate places would be counted as separate groups.

## Stage 4: `uniq -c`

- `uniq` collapses consecutive identical lines into one.
- `-c` **prefixes each line with its count** (how many times it occurred).

Output looks like:

```
   312 09
   845 14
   590 15
```

## Stage 5: `sort -nr`

- `-n` sorts **numerically** rather than alphabetically. Without it, `9` would sort above `845` since `"9" > "8"` as text.
- `-r` **reverses** the order, so the largest counts come first.

Because the count is the first thing on each line, sorting numerically ranks the hours by request volume, highest first.

## Stage 6: `head -3`

Prints only the **first 3 lines** (`-3` is shorthand for `-n 3`), giving the top three busiest hours.

## Final result

```
    845 14
    590 15
    312 09
```

Meaning: 845 requests during the 14:00 hour, 590 during 15:00, and 312 during 09:00. Note that it aggregates by hour of day across *all dates* in the file, not by individual date-hours.

## Caveats

- **It's format-dependent.** If your log has a different layout (custom format, extra spaces in earlier fields), field 4 may not be the timestamp.
- **Multiple spaces break `cut`.** Unlike `awk`, `cut` treats each single space as a separator, so runs of spaces create empty fields.
- **Multi-day logs get merged.** If you want hours per date, extract the date too, e.g. `cut -d' ' -f4 access.log | cut -c2-15 | sort | uniq -c | sort -nr | head -3`.

An `awk` equivalent that's a bit more robust: `awk '{print substr($4,14,2)}' access.log | sort | uniq -c | sort -nr | head -3`.
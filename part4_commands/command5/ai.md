This is the same 404 pipeline from earlier, so this walkthrough keeps the other stages brief and spends its time on `uniq`, which is the stage that does the counting.

## The pipeline in one line

`grep` keeps only 404 lines, `cut` pulls out the IP, `sort` groups identical IPs, **`uniq -c` counts each group**, `sort -nr` ranks by count, and `head -5` keeps the top five.

## Stages 1-3, briefly

- `grep '" 404 ' access.log` keeps only lines where the status code is 404.
- `cut -d' ' -f1` extracts field 1, the client IP.
- `sort` puts identical IPs next to each other. This is a prerequisite for `uniq`, explained below.

At this point the stream looks like this:

```
10.0.0.8
10.0.0.8
172.16.4.2
192.168.1.5
192.168.1.5
192.168.1.5
```

## Stage 4: `uniq -c`

### What `uniq` does

`uniq` reads lines and **collapses consecutive duplicates into a single line**. It has no memory beyond the previous line. For each new line, it asks one question: "Is this identical to the line right before it?" If yes, it's absorbed into the current group. If no, a new group starts.

Without any options, the output of the stream above would be:

```
10.0.0.8
172.16.4.2
192.168.1.5
```

That removes duplicates but tells you nothing about how many there were.

### What `-c` adds

`-c` (count) **prefixes each output line with the number of lines in its group**:

```
      2 10.0.0.8
      1 172.16.4.2
      3 192.168.1.5
```

The count is right-aligned in a fixed-width column, then a space, then the original line. In this pipeline, that turns a list of IPs into a frequency table: each IP and how many 404s it caused.

### Why `uniq` needs sorted input

Because `uniq` only compares neighbors, unsorted input produces wrong counts. If the stream were in log order:

```
192.168.1.5
10.0.0.8
192.168.1.5
10.0.0.8
192.168.1.5
```

then `uniq -c` would output:

```
      1 192.168.1.5
      1 10.0.0.8
      1 192.168.1.5
      1 10.0.0.8
      1 192.168.1.5
```

Five groups of one, instead of two groups with counts 3 and 2. This is why `sort` always comes immediately before `uniq`.

### Why this is better than `sort | uniq` thinking of it as "dedupe"

People often use `sort | uniq` to remove duplicates. Here the duplicates are the data: the number of repeats *is* the answer. `-c` preserves that information instead of throwing it away.

### Other useful `uniq` options

- **`-d`** prints only lines that were duplicated (appeared more than once), one copy each. For example, IPs that caused at least two 404s.
- **`-u`** prints only lines that appeared exactly once, such as IPs that triggered a single 404.
- **`-i`** ignores case when comparing. Irrelevant for IPs, useful for text like usernames or URLs.
- **`-f N`** skips the first N whitespace-separated fields when comparing, and **`-s N`** skips the first N characters.
- **`-w N`** compares only the first N characters.

### Related: `sort -u`

`sort -u` sorts and removes duplicates in one step, equivalent to `sort | uniq`. It can't count, though, so it's no substitute here. It's useful when you only need the *distinct* values:

```bash
grep '" 404 ' access.log | cut -d' ' -f1 | sort -u
```

## Stages 5-6, briefly

- `sort -nr` sorts the `uniq -c` output **numerically** (`-n`) and in **reverse** (`-r`), so the highest counts come first. It works because `uniq -c` puts the count at the start of each line.
- `head -5` keeps the first five lines.

## Final result

```
    212 192.168.1.5
     97 172.16.4.2
     41 203.0.113.9
     12 198.51.100.7
      3 10.0.0.8
```

## Caveats

- **Format-dependent.** It assumes the IP is field 1 and that the status code follows the closing quote of the request.
- **Reverse proxies.** Behind a load balancer, field 1 may be the proxy's address.
- **Output format varies slightly.** The width of the count column differs between GNU and BSD/macOS `uniq`, so avoid parsing it by character position. Splitting on whitespace is safer.
- **`uniq` compares whole lines.** If you ever feed it lines with extra columns, two lines that differ in any column are counted as different groups.
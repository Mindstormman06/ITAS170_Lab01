This pipeline finds the five IP addresses that sent the most requests overall. It's the same shape as the 404 pipeline, minus the `grep` filter, so this walkthrough spends most of its time on the two `sort` stages and why they're both needed.

## Stage 1: `cut -d' ' -f1 access.log`

- `-d' '` sets the delimiter to a single space.
- `-f1` selects field 1, the client IP in common log format.
- `access.log` is read directly as the input file.

The output is one IP per request, in the order the requests were logged:

```
192.168.1.5
10.0.0.8
192.168.1.5
172.16.4.2
10.0.0.8
192.168.1.5
```

## Stage 2: `sort` (the first sort)

With no options, `sort` orders lines **alphabetically** (lexicographically, character by character). Applied to the stream above, it gives:

```
10.0.0.8
10.0.0.8
172.16.4.2
192.168.1.5
192.168.1.5
192.168.1.5
```

The ordering itself doesn't matter here. What matters is the side effect: **identical IPs end up adjacent**. That's the entire purpose of this stage.

Why it's mandatory: `uniq` only compares each line to the one immediately before it. On the unsorted stream, `192.168.1.5` would be counted as three separate groups of one, instead of one group of three.

Note that this is a text sort, so `172.16.4.2` lands between `10.x` and `192.x` by character comparison, not by numeric IP value. That's fine for grouping, but it means this stage's output isn't in true IP order. If you ever want that, `sort -t. -k1,1n -k2,2n -k3,3n -k4,4n` sorts by octet, or `sort -V` does a version-style natural sort on GNU systems.

## Stage 3: `uniq -c`

`uniq` collapses runs of identical adjacent lines into one, and `-c` prefixes each with the run length:

```
      2 10.0.0.8
      1 172.16.4.2
      3 192.168.1.5
```

The count is right-aligned in a fixed-width column, followed by a space, then the IP. That layout (count first) is what makes the next stage work.

## Stage 4: `sort -nr` (the second sort)

This is the ranking step, and each option matters.

**`-n` (numeric sort).** Without it, `sort` compares text. Compare the two orderings of counts 3, 25, and 100:

```
Text sort, descending:      Numeric sort, descending:
      3 ...                       100 ...
     25 ...                        25 ...
    100 ...                         3 ...
```

As text, `"3"` beats `"25"` and `"100"` because `3` is greater than `2` and `1` as a first character. `-n` makes `sort` read the leading number on each line and compare by value. It also skips the leading spaces `uniq -c` adds, so the padding doesn't interfere.

**`-r` (reverse).** `sort` defaults to ascending order, smallest first. `-r` flips it, so the biggest counts come first. Without `-r`, you'd get the *least* active IPs at the top.

**Combined as `-nr`.** Short options can be bundled, so this is the same as `-n -r`.

**Which part of the line is compared?** With no `-k` option, `sort` uses the whole line as the key. With `-n`, only the leading numeric portion is parsed, which is the count. Everything after it (the IP) is ignored for numeric comparison.

## Tie-breaking

When two IPs have the same count, `-n` sees them as equal. By default, GNU `sort` then falls back to a last-resort comparison of the **entire line** as text (also reversed under `-r`), so ties come out in a deterministic order. If you want to disable that and keep tied lines in their original input order, add `-s` (stable sort):

```bash
sort -nr -s
```

## Stage 5: `head -5`

Prints the first 5 lines, giving the five busiest IPs:

```
    212 192.168.1.5
     97 172.16.4.2
     41 203.0.113.9
     12 198.51.100.7
      3 10.0.0.8
```

## Alternative that skips the first sort

If you'd rather not sort the whole stream first (it can be slow on huge logs), `awk` can count with an associative array and only sort the much smaller summary:

```bash
awk '{c[$1]++} END {for (ip in c) print c[ip], ip}' access.log | sort -nr | head -5
```

Here only one sort is needed, because `awk` has already aggregated the counts and the sort is purely for ranking.

## Caveats

- **Format-dependent.** It assumes the IP is field 1.
- **Reverse proxies.** Behind a load balancer, field 1 may be the proxy's address rather than the client's.
- **Locale affects `sort`.** In some locales, text sorting ignores punctuation or case. For the grouping stage this is harmless, but for predictable behavior in scripts, `LC_ALL=C sort` forces plain byte-order sorting and is also noticeably faster.
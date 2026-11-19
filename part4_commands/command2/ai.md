This pipeline finds the five IP addresses that triggered the most 404 (Not Found) errors. It's a close cousin of the last one, with `grep` doing the filtering up front and the IP address (field 1) being counted instead of the hour.

## Stage 1: `grep '" 404 ' access.log`

- `grep` prints only the lines matching a pattern.
- The pattern `'" 404 '` is **a double quote, a space, `404`, and another space**. It's wrapped in single quotes so the shell passes the `"` through literally.
- `access.log` is the input file.

The pattern is chosen to match the spot in a common-format log line where the status code sits:

```
192.168.1.5 - - [18/Nov/2026:14:32:07 -0800] "GET /missing HTTP/1.1" 404 512
                                                                   ^^^^^^^^
```

The request section ends with a closing `"`, followed by a space, the status code, and a space before the byte count. Anchoring on `" 404 ` rather than just `404` avoids false positives like a byte count of `404`, an IP such as `10.0.4.40`, or a URL containing `404` (e.g. `/page404.html`).

Output: only the full log lines with a 404 status.

## Stage 2: `cut -d' ' -f1`

- `-d' '` sets the delimiter to a single space.
- `-f1` selects **field 1**, which is the client IP address in this format.

The stream is now just one IP per 404 request:

```
192.168.1.5
10.0.0.8
192.168.1.5
```

## Stage 3: `sort`

Sorts the IPs so identical addresses sit next to each other. Again, this is required because `uniq` only merges *adjacent* duplicates.

## Stage 4: `uniq -c`

Collapses each run of identical IPs into one line, prefixed with its count (`-c`):

```
    212 192.168.1.5
      3 10.0.0.8
     97 172.16.4.2
```

## Stage 5: `sort -nr`

- `-n` sorts **numerically**, so `97` ranks below `212` instead of above it as text.
- `-r` **reverses** the order, putting the biggest counts first.

## Stage 6: `head -5`

Keeps only the **first 5 lines**, so you see the five worst offenders.

## Final result

```
    212 192.168.1.5
     97 172.16.4.2
     41 203.0.113.9
     12 198.51.100.7
      3 10.0.0.8
```

Meaning: the top five client IPs by number of 404 responses. This is a handy way to spot vulnerability scanners, broken crawlers, or a misconfigured client hammering missing URLs.

## Caveats

- **Format-dependent.** It assumes the IP is field 1 and that the status code follows the closing quote of the request. A custom log format will break this.
- **Reverse proxies.** If your server sits behind a load balancer, field 1 may be the proxy's IP rather than the real client's. Check whether your log uses `X-Forwarded-For`.
- **Quotes inside requests.** An odd request containing a `" 404 ` sequence in its own text could produce a rare false match, though it's uncommon.
- **An `awk` version** that checks the status field explicitly (more precise, no pattern guesswork):

```bash
awk '$9 == 404 {print $1}' access.log | sort | uniq -c | sort -nr | head -5
```

Here `$9` is the status code in common log format, so the match is exact.
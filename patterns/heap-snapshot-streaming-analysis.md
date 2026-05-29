# Streaming analysis of large Chrome heap snapshots

DevTools' "Memory" tab can capture heap snapshots, but the snapshotter itself runs in another Chrome renderer and needs roughly as much memory as the target heap. **When the leak is >2-3 GB you can't open the snapshot in DevTools at all** — the diagnostic tool OOMs before it can show you anything. This pattern handles that case.

## The .heapsnapshot format

It's JSON, despite the extension. Structure:

```
{
  "snapshot": {
    "meta": {
      "node_fields": ["type", "name", "id", "self_size", "edge_count", "detachedness"],
      "node_types": [...enum names...],
      "edge_fields": [...],
      "edge_types": [...]
    },
    "node_count": N,
    "edge_count": M
  },
  "nodes":   [flat int array, N * len(node_fields) entries],
  "edges":   [flat int array, M * len(edge_fields) entries],
  "strings": ["str0", "str1", ...]
}
```

Each node occupies `len(node_fields)` consecutive ints. To get a node's constructor name: take its `name` field, that's an index into the `strings` array.

Sizes scale roughly: a 1.4 GB snapshot on disk has ~18 M nodes and ~75 M edges, but only ~600 K strings. **Strings barely grow during a memory leak** — leaks add object instances, not new label data.

## Why streaming matters

`json.load()` on a 1.4 GB snapshot uses 6-10 GB of Python heap (Python objects carry overhead). On a 16 GB Mac you'll OOM. Streaming with `ijson` + pre-allocated `array.array('i')` keeps total memory under ~300 MB even on multi-GB inputs.

## Approach

Two passes through the file (sequential disk reads are fast even on multi-GB):

1. **Pass 1** — stream `nodes.item` events. For each node, store only `(type_id, name_id, self_size)` as int32. ~12 bytes per node × 18 M = ~216 MB.
2. **Pass 2** — stream `strings.item` events into a list. ~50 MB.
3. **Resolve** — walk the buffered nodes, look up names from the strings table, aggregate by constructor.

The trick is `strings` come **after** `nodes` in the JSON. You can't resolve names during the first pass; you have to buffer the integer IDs and resolve at the end.

```python
import ijson, array

# Pass 1: buffer (type_id, name_id, self_size) per node
types, names, sizes = array.array("i"), array.array("i"), array.array("i")
with open(path, "rb") as f:
    field_pos = 0
    cur_type = cur_name = cur_size = 0
    for value in ijson.items(f, "nodes.item"):
        if field_pos == TYPE_IDX:  cur_type = value
        elif field_pos == NAME_IDX: cur_name = value
        elif field_pos == SIZE_IDX: cur_size = value
        field_pos += 1
        if field_pos == FIELD_COUNT:
            types.append(cur_type); names.append(cur_name); sizes.append(cur_size)
            field_pos = 0

# Pass 2: strings
with open(path, "rb") as f:
    strings = list(ijson.items(f, "strings.item"))

# Aggregate by constructor
```

Pre-sizing `array.array` (extend with zeros to `node_count` before the loop) speeds it up significantly vs appending.

## Reference tool

Published as a public gist: https://gist.github.com/jblenman/926498490d4736f28cf2e1850167792e — three scripts (`heap_stream_count.py` for streaming aggregation, `heap_diff_report.py` for N-snapshot diffs, `heap_summary.py` for small one-shot summaries) plus a usage README. Together: ~12 seconds per snapshot on a 300 MB input, ~65 s on 1.4 GB. Suitable for linking from upstream bug reports when the maintainer might want to run the analyzer on their own captures.

## Reading the output

What to look for:

- **Strings count flat across snapshots** → leak is object instances, not cached data. (Cached data leak: large unique strings would grow.)
- **One node-type category exploding 30-50×** while others stay 1-2× → the leak has a *kind*, not a scattered nature.
- **Multiple constructors growing in identical lockstep** → automated framework code (event listener loops, subscription factories, observer hooks).
- **Class names with library-specific prefixes** (`var(--darkreader-...)`, `RxJS.Subscription`, etc.) → identifies the responsible code.

A diff between snapshot 1 and snapshot 4 ranked by count delta usually surfaces the culprit in the top 10 rows.

## Capturing snapshots when the leak is fast

If the heap grows so fast that even DevTools can't snapshot it without crashing:

1. Open the page **fresh** and immediately hit snapshot. Catch it small.
2. Wait 10-30 seconds, snapshot again.
3. Repeat 2-4 times before the heap blows past your free RAM.
4. Save each one (right-click → Save). Files compress poorly, expect 100s of MB to 1+ GB each.

Launching Chrome with `--js-flags="--max-old-space-size=8192"` gives DevTools more room to do its work before OOM.

## Sanitizing before sharing

Heap snapshots are JSON and will contain string fragments from the page — emails from auth UI, project IDs, OAuth client IDs, account display names. To share publicly:

```bash
sed -f scrub.sed input.heapsnapshot > sanitized.heapsnapshot
```

Heap snapshots index strings by *array position*, not byte offset, so length-varying replacements inside string values don't corrupt the format. Validate by re-parsing the sanitized file through `heap_stream_count.py`; if it produces sensible output, the JSON is intact.

Useful identifier patterns to scrub for typical GCP/Drive sessions:
- Email addresses (regex `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-zA-Z]{2,4}`)
- GCP project ID strings (`your-project-id-123456`)
- GCP project numbers (12-digit standalone numerics from OAuth client ID prefixes)
- OAuth client IDs (`<number>-<random>.apps.googleusercontent.com`)
- Display names (your surname, given name)

## Related

- [chrome-extension-spa-leaks.md](chrome-extension-spa-leaks.md) — Dark Reader signature, the case study this tooling was built for
- [oss-maintainer-bug-reports.md](oss-maintainer-bug-reports.md) — how to write the report that goes with these snapshots

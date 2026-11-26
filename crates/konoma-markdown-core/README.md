# konoma-markdown-core

The Markdown core of [konoma](https://github.com/LESIM-Co-Ltd/konoma), built on
[`pulldown-cmark`](https://crates.io/crates/pulldown-cmark). Nothing here draws.

| Module | What it offers |
|---|---|
| `model` | `Doc::parse`: a tree of blocks from one pass over the parser's events, with byte ranges into the source |
| `diff` | `block_ops` and `preview_marks`: which blocks differ between two versions |
| `syntax` | Source-text analysis and pre-passes: fences and code masks, front matter, footnotes, inline HTML, `<details>`, alert headers |

It has no HTML output and no renderer. The tree describes blocks only: inline content stays as a
range of parser events (headings, paragraphs) or of source bytes (table cells).

```rust
use konoma_markdown_core::diff::{block_ops, BlockOp};
use konoma_markdown_core::model::{BlockKind, Doc};
use konoma_markdown_core::syntax::strip_front_matter;

fn main() {
    let file = "---\ntitle: Demo\n---\n# Title\n\n- [ ] write it\n";
    let (front_matter, body) = strip_front_matter(file);
    assert_eq!(front_matter.as_deref(), Some("title: Demo"));

    // Block ranges and task positions are byte offsets into the string given to `parse`.
    let doc = Doc::parse(&body);
    assert!(matches!(doc.blocks[0].kind, BlockKind::Heading { level: 1, .. }));

    let item = &doc.blocks[1].children[0];
    let BlockKind::ListItem { task: Some(task) } = &item.kind else {
        panic!("expected a task item");
    };
    assert_eq!(&body[task.state_at - 1..task.state_at + 2], "[ ]");

    // Compare two versions block by block.
    let edited = body.replace("# Title", "# New title");
    let new_doc = Doc::parse(&edited);
    let ops = block_ops(&doc, &new_doc, &body, &edited);
    assert_eq!(ops[0], BlockOp::Replace { old: 0, new: 0 });
    assert_eq!(ops[1], BlockOp::Equal { new: 1 });
    println!("{ops:?}");
}
```

## Behaviour to know about

- `Doc::parse` does not run the pre-passes. It does not strip front matter, number footnotes or
  convert inline HTML; call the `syntax` functions first if you need that. Offsets then refer to
  the rewritten text.
- The diff compares top-level blocks. A change to one item of a list marks the whole list, and a
  moved block appears as a deletion plus an insertion.
- There is no limit on input size or nesting depth, so callers should bound untrusted input before
  parsing it.

The one feature, `test-hooks`, exists for konoma's own tests. Rust 1.92 or newer; tested in CI on
macOS and Linux. MIT licensed.

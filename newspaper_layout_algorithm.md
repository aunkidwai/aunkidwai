# Automated Newspaper Layout Algorithm

This document proposes a deterministic-yet-adaptable algorithm to generate newspaper-like grids for a variable number of articles. It balances hierarchy (headlines, images, quotes) with spatial efficiency and responds fluidly to page resizing.

## Content model
Each article is normalized into a `Block` structure before layout:

```json
{
  "id": "article-1",
  "headline": { "words": 8 },
  "subhead": { "words": 12, "optional": true },
  "body": { "words": 350 },
  "quote": { "words": 24, "optional": true },
  "image": { "aspect_ratio": 1.6, "optional": true },
  "priority": "A|B|C" // A = front-page feature, B = standard, C = filler
}
```

Derive a **unit height** estimate using word counts and typographic constants (e.g., `lines = ceil(words / words_per_line)`, `px = lines * line_height`). Maintain canonical **column width** across the page (e.g., 4–7 columns).

### Module recipes per article
- **Feature (priority A)**: full-width headline, subhead, image (optional), multi-column body, pull-quote if present.
- **Standard (B)**: headline spanning 2–3 columns, optional subhead, body in 1–2 columns, inline image or quote.
- **Filler (C)**: 1-column headline + body, no subhead; quote or image only if space remains.

Convert each article into one or more **modules** (rectangles) sized in column units and unit heights:
- `headline`: fixed small height; spans same columns as body.
- `subhead`: small height; same span as headline.
- `body`: dominant height; flexible column span.
- `image/quote`: reserve height from aspect ratio or word count; allow float left/right or full-span depending on priority.

## Layout pipeline
1. **Pre-sort modules** by priority, then by height descending (guillotine/skyline packing works best with tallest-first).
2. **Initialize grid** with `N` columns and page height budget `H`. Each cell stores occupancy (`row, col, height, width`).
3. **Place feature articles first**:
   - Try 3–4 column spans; allow top-of-page bias (row penalty increases with depth).
   - Keep image directly above or below body; keep headline at module top.
4. **Place standard articles**:
   - Use 2–3 column spans; prefer contiguous vertical space to reduce fragmentation.
   - Allow image/quote to float in first third of body height when width ≥ 2 columns.
5. **Place filler articles** in remaining gaps using skyline or binary-split packing.
6. **Resolve orphans/widows** by nudging heights within ±5% and redistributing empty rows across column tracks.
7. **Balance columns**: compute column fill ratio; if any column deviates >10% from mean, widen/narrow neighboring spans (subject to minimum column width) and reflow affected modules.

### Placement heuristic (skyline variant)
- Maintain a list of **skyline heights per column**.
- For each module candidate width `w` and height `h`:
  1. Slide across columns to find the position with minimum resulting max-height (`best_col`).
  2. If multiple slots tie, choose the one closest to page top-left (visual hierarchy).
  3. Place module, update skyline heights for columns `[best_col, best_col + w)`.
- When modules belong to the same article, lock their relative ordering (headline → subhead → image/quote → body) and constrain them to adjacent columns.

### Handling dynamic resize / edge dragging
When the page width changes or edges move:
1. Recompute `N` columns (`min_col_width` and `gutter` fixed). Common rule: `N = clamp(floor((page_width + gutter) / (min_col_width + gutter)), 3, 8)`.
2. Recalculate each module’s width in columns (keep approximate physical width) and recompute heights from new column width.
3. Re-run the placement pipeline with the preserved article order and priority weights.
4. Animate transitions using old → new positions to achieve “dynamic” feel.

## Pseudocode
```python
def layout_page(blocks, page_width, page_height, min_col_width=120, gutter=12):
    N = clamp(int((page_width + gutter) / (min_col_width + gutter)), 3, 8)
    modules = expand_blocks(blocks, column_width=compute_col_width(page_width, N, gutter))

    skyline = [0] * N
    placements = []

    for module in sort_modules(modules):
        w = module.columns
        h = module.estimated_height
        col, y = find_slot(skyline, w)
        placements.append((module.id, col, y, w, h))
        for c in range(col, col + w):
            skyline[c] = y + h

    return placements, skyline
```
`expand_blocks` performs word-count → height estimation and enforces module recipes. `find_slot` implements the skyline sweep (pick the column index that yields the lowest max height after placement; break ties toward smaller column indexes).

## Validation hooks
- **Aspect ratio tolerance** for images: allow ±10% stretch; otherwise shift to next row.
- **Read-order validation**: ensure vertical order inside each article is preserved by comparing `y` coordinates in `placements`.
- **Space utilization**: `coverage = total_area(placements) / (N * page_height)`; trigger column rebalance if coverage < 0.8 for high-priority pages.

## Extensions
- Add constraint solver (ILP or simulated annealing) for special layouts (e.g., center-spread images).
- Support advertisements by reserving fixed modules before article placement.
- Allow editors to pin modules; the algorithm locks pinned coordinates and reflows the rest.

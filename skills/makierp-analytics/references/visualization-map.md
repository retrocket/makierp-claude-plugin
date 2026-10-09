# Visualization: Map

A map answers "where". It plots one point per row from a latitude and a longitude
column, so any query that returns coordinates can use it — customer locations are
the first case, route stops and delivery points the obvious next ones.

Reach for it when the geography is the finding: which towns a channel has taken
and which it has not, where the biggest customers sit relative to the depot,
whether a representative's route matches where the volume actually is. A bar by
district gives you the same totals with the shape thrown away.

```json
{
  "selected_viz": "map",
  "visualization": {
    "_columns": {
      "cari": {"label": "Cari"},
      "kanal": {"label": "Kanal"},
      "net_karton": {"label": "Net Karton"}
    },
    "map": {
      "latitude_column": "enlem",
      "longitude_column": "boylam",
      "label_column": "cari",
      "value_column": "net_karton",
      "color_mode": "category",
      "color_column": "kanal",
      "colors": {"Bakkal": "#509EE3"},
      "size_range": [8, 34],
      "cluster": true,
      "bounds_mode": "kktc",
      "show_legend": true,
      "link_target": "customer",
      "link_column": "cari_id",
      "tile_style": "auto"
    }
  }
}
```

Coordinates come from the address relation (`addresses.latitude` /
`addresses.longitude`). Preview the query first and use the exact output keys.

## Keys

- `latitude_column`, `longitude_column` (required): numeric. Anything else is
  rejected before the question saves.
- `label_column`: what the point is called on hover.
- `value_column`: sizes the bubbles, by **area** — a point with four times the
  measure is drawn twice as wide, because that is how a reader compares circles.
  Omit it and every point is drawn the same size.
- `color_mode`: `measure` (default) ramps between the two `color_scale` ends;
  `category` gives each value of `color_column` its own colour and legend entry.
- `colors`: pins individual category colours (`{"Bakkal": "#509EE3"}`); the rest
  come from the shared palette, the same one the pie uses, so a question keeps its
  colours when it is switched between the two.
- `color_scale`: the two ends of the measure ramp.
- `size_range`: `[min, max]` bubble diameter in pixels.
- `cluster`: merges points once there are more of them than the picture can draw
  separately. A cluster sits at the mean position of its members and carries the
  **sum** of their measure.
- `heatmap`: adds a density layer under the points.
- `bounds_mode`: `kktc` (fixed extent) or `data` (fit the rows).
- `show_legend`, `tooltip_columns`, `zoom`, `center`.
- `link_target`: `customer` makes a point open the customer card, `none` disables it.
- `link_column`: the column holding the customer id the link opens. Select the id
  itself, not the name — a map keyed on names cannot open anything.
- `tile_style`: `auto` (default, follows the viewer's theme), `light` or `dark`.
  Only the browser's basemap; an export has no basemap to style.

## Three traps, all of them quiet

**Do not group the query coarser than the coordinate.** A `group_by` on district
with an averaged latitude puts one point in an empty field between two towns, and
nothing about the picture says so. One row per located thing.

**A map's point count is not the query's row count.** A missing fix arrives as
`null` *or as `0`*, and 0/0 is open water off Africa, not an address — both are
dropped. Roughly half of this tenant's piyasa customers have no coordinate at
all, so a map of "all customers" is really a map of the located ones. Put the
uncovered count on the card; otherwise the picture quietly claims full coverage.

**Prefer `bounds_mode: "kktc"`.** Fitting the extent to the data lets a single
foreign address — the tenant has one in İstanbul — open the map up to two
countries and shrink the island to a smudge. The fixed extent leaves such rows
out of the picture, which is worth saying on the card when it matters.

## Exports

Mail and PDF attachments render the map server-side **without a basemap**: the
points, their sizes and their colours survive; map tiles do not. The distribution
still reads; street-level recognition does not. Keep that in mind before making a
map the only attachment on a scheduled mail.

## Not a map

- Totals per district you only want to compare → `bar` or `table`.
- A relationship between two measures → `scatter`.
- Share of a whole across regions → `pie`.

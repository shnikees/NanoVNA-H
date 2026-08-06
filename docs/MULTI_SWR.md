# MULTI SWR (Multé) — customisation guide

Everything the MULTI SWR mode does is driven by a handful of values in
[`measure.c`](../measure.c). This document says which ones, and what changing
them costs. After any edit, rebuild and reflash:

```
make TARGET=F072 -j4                                    # NanoVNA-H  (rev 3.x)
make TARGET=F303 -j4                                    # NanoVNA-H4 (rev 4.x)
dfu-util -d 0483:df11 -a 0 -s 0x08000000:leave -D build/H.bin
```

---

## Changing the bands (IARU region, extra bands)

**The shipped band plan is US / IARU Region 2.** Band edges differ elsewhere —
Region 1 (Europe/Africa) 80m is 3.500–3.800MHz rather than 3.5–4.0, 40m is
7.000–7.200, and 902–928MHz is not an ISM allocation at all. Edit the table in
`measure.c`:

```c
static const struct {
  const char *name;
  freq_t lo;
  freq_t hi;
  uint16_t points;
} multi_swr_bands[] = {
  {"160m", 1800000,   2000000,   11},  //  20kHz step
  ...
  {"33cm", 902000000, 928000000, 27},
};
```

Each row is `{label, start Hz, stop Hz, points}`.

- **`label`** — up to **10 characters**. The stars column starts at 60px and the
  font is 6px wide, so anything longer runs into it.
- **`points`** — how finely that band is swept. `step = (hi - lo) / (points - 1)`.
  Keep the step comfortably smaller than the width of the SWR dip you expect;
  a few tens of kHz is plenty on HF. More points cost scan time directly.
- **Order** — keep it ascending by frequency. Rows are drawn in table order.

Adding or removing a band needs no other change; everything sizes off
`MULTI_SWR_BAND_COUNT`. Two limits apply:

- **Memory.** Results live in a fixed 128-byte scratch buffer at 8 bytes per
  band, so **16 bands maximum**. 14 are currently used.
- **Screen space.** Rows are 12px from `MULTI_SWR_TOP` (20px). The layout is
  header, then one row per band, then the footer, then the legend, and the
  status line sits at y=233. With 14 bands the legend lands at y=212 and ends at
  223. **Two more bands will collide with the status line** — at that point
  raise `MULTI_SWR_TOP`, or drop the legend.

Some regional examples:

```c
  {"80m",  3500000,  3800000,  13},  // Region 1
  {"40m",  7000000,  7200000,  11},  // Region 1
  {"868MHz", 863000000, 870000000, 15},  // EU ISM
```

Bands above **300MHz** are measured in si5351 harmonic mode and are noisier;
above roughly 900MHz accuracy falls off further. The firmware caps at 2GHz
(`FREQUENCY_MAX`), so 2.4GHz cannot be added.

---

## Changing the star thresholds

The five SWR boundaries, best first:

```c
static const float multi_swr_thr[5] = {1.10f, 1.15f, 1.30f, 1.70f, 3.00f};
```

`multi_swr_thr[4]` (3.00) is also the pass/fail line: a band worse than this is
drawn red as failing rather than earning a star. Raise it if you want a more
forgiving display, lower it to be stricter.

Nearby, `MULTI_SWR_HYST` (0.02) is how far the SWR must move past a boundary
before the displayed rating changes. It stops a band sitting exactly on a
threshold from flickering between two ratings. Increase it if ratings still
twitch; decrease it if the display feels slow to react.

---

## Changing the colours

```c
static pixel_t multi_swr_color(int stars) {
  if (stars >= 4) return RGB565(  0, 255,   0);  // green:  great
  if (stars >= 2) return RGB565(255, 255,   0);  // yellow: ok
  if (stars == 1) return RGB565(255, 160,   0);  // orange: marginal
  return                 RGB565(255,   0,   0);  // red:    failing
}
```

`RGB565(r, g, b)` takes ordinary 0–255 values. This one function colours the
band rows and the matching legend entries, so they cannot fall out of step.

---

## Changing the scan speed

Two things set how long a full pass takes.

**Points per band** — the `points` column above. This is the honest lever: fewer
points is faster and coarser.

**IF bandwidth** — in [`main.c`](../main.c), on entry to the mode:

```c
config._bandwidth = BANDWIDTH_4000;
```

Dwell time is `bandwidth + 2` samples per point, so `BANDWIDTH_4000` takes 3
samples where the 1000Hz default takes 9. Available values are
`BANDWIDTH_8000`, `_4000`, `_2000`, `_1000`, `_333`, `_100`, `_30`, `_10`.
Narrower is quieter
but slower; `BANDWIDTH_2000` or `_1000` is the fallback if ratings look jumpy.
The user's own setting is saved on entry and restored on exit — assign this
field directly rather than calling `set_bandwidth()`, which triggers a settings
backup write.

---

## Display internals worth knowing

If you move things around, the repaint region has to keep up. It is set in
`multi_swr_refresh()`:

```c
invalidate_rect(STR_MEASURE_X, MULTI_SWR_TOP,
                STR_MEASURE_X + MULTI_SWR_INVAL_W,
                MULTI_SWR_TOP + (MULTI_SWR_BAND_COUNT + 3) * STR_MEASURE_HEIGHT);
```

**Anything drawn outside this rectangle is never erased** and will leave stale
text on screen — both display bugs found during development were exactly this,
once horizontally (text ran past the width) and once vertically (the footer sat
below the height). If you widen a column or add a row, widen or extend this too.
`MULTI_SWR_INVAL_W` is 240px, which stops just short of the menu at 241px.

The table only repaints when something it displays actually changes: a star
rating, a resonance frequency that has moved more than two sweep steps, or a
failing band's SWR moving more than 0.1. A steady antenna produces no redraw at
all, which is what keeps the menu responsive.

Bands are scanned **one per main-loop iteration** (`multi_swr_run()` in
`main.c`), not all in one burst, so the UI stays responsive and a partial pass
never blanks the table. Results are written through per band and only cleared
when the mode is entered.

# Favorites Implementation Plan for 4cade

## Overview

Add the ability to mark games as favorites, browse only favorites, and persist the list across power cycles.

---

## 1. Storage Design

### On-disk: `FAVORITES.CONF`

A new ProDOS file on disk alongside `PREFS.CONF`. Contains favorite game filenames as a simple OKVS (using the existing `okvs_append` format from init code):

- Header: 2-byte record count + 2-byte free pointer
- Records: length-prefixed game filenames (keys only, no values needed)
- Max size: 1 block (512 bytes) — fits ~25-30 favorites at ~15 bytes per game filename
- Read/written using the same `SaveSmallFile` / `LoadFile` infrastructure as `PREFS.CONF`

Why filenames and not indices: game indices change when the search index is rebuilt (new games added to the collection), so storing the actual filename string guarantees favorites survive a rebuild.

### In-memory: 64-byte bitfield

At runtime, the favorites file is translated into a **bitfield** indexed by the game's position in `gSearchStore`. With a max of 511 games, that's `ceil(511/8)` = **64 bytes**.

- Bit N set = game #N is a favorite
- Recalculated on load by looking up each filename from `FAVORITES.CONF` in `gSearchStore`
- Fast O(1) check: `LDA bitfield,X / AND bitmask,Y`

**Memory location options** (in order of preference):

| Location | Address | Pros | Cons |
|----------|---------|------|------|
| LC RAM bank 1 gap | $D070-$D0AF | Out of the way, persistent | Requires bank switching to access |
| Main memory | $1F40-$1F7F | Direct access, in existing "temp" area | Overlaps gValLen/gVal parse buffers (only used during init) |
| Aux memory page | Any free aux page | Completely isolated | Requires aux mem read switching |

**Recommended:** `$1F40-$1F7F` in main memory. The `gValLen`/`gVal` parse buffers at `$1F80` are only used during OKVS parsing in init. After init completes, this area is free. The 64-byte bitfield fits at `$1F40-$1F7F` without overlapping `UILine1` ($1FB0) or `gKeyLen`/`gKey` ($1F00) which are used elsewhere.

---

## 2. Key Bindings

| Action | Key | Dispatch index | Context |
|--------|-----|----------------|---------|
| Toggle favorite on current game | `Ctrl-F` ($86) | `kBrowseFavorite` / `kInputFavorite` | Browse & Search mode |
| Toggle favorites-only browsing | `Ctrl-V` ($96) | `kBrowseFavView` / `kInputFavView` | Browse & Search mode |

Both modes need new entries in their key tables (`BrowseKeys`/`BrowseKeyDispatch` and `InputKeys`/`InputKeyDispatch`).

Optional joystick mapping: a long-press or double-tap of button 0 could toggle favorite, but this adds complexity. Better to keep it keyboard-only initially.

---

## 3. Runtime State

Add two new global bytes (can live alongside `gPreloadStatus` and other globals near end of `4cade.a`):

```
gFavoritesMode    !byte 0    ; $00 = normal browse, $FF = favorites-only
gFavoritesCount   !byte 0    ; number of favorites (0-255)
```

Plus the 64-byte bitfield:

```
gFavoriteBits = $1F40        ; 64 bytes ($1F40-$1F7F)
```

And a bitmask lookup table (8 bytes, can be in the new source file):

```
FavBitmask
    !byte $01, $02, $04, $08, $10, $20, $40, $80
```

---

## 4. New Routines (new file: `src/ui.favorites.a`)

### `LoadFavorites`

Called once during init (after search index is loaded):

1. Load `FAVORITES.CONF` into a temporary buffer (e.g., `$0900` like help/credits text)
2. Clear `gFavoriteBits` (64 bytes to $00)
3. Set `gFavoritesCount` to 0
4. For each filename in the OKVS:
   - Call `okvs_find` on `gSearchStore` to get the game's index
   - If found: set the corresponding bit in `gFavoriteBits`, increment `gFavoritesCount`
   - If not found: game was removed from collection, skip it
5. If file doesn't exist, just clear the bitfield (no favorites yet)

### `ToggleFavorite`

Called when user presses Ctrl-F:

1. Get `gGameToLaunch` — bail if $FFFF (no game selected)
2. Calculate byte offset (`gGameToLaunch / 8`) and bit mask (`gGameToLaunch AND 7`)
3. EOR the bit in `gFavoriteBits`
4. Update `gFavoritesCount` (increment or decrement)
5. Rebuild and save `FAVORITES.CONF`:
   - Iterate `gSearchStore` with `okvs_iter`
   - For each game whose bit is set, append filename to buffer
   - Write buffer to disk with `SaveSmallFile`
6. Update UI to show favorite indicator

### `IsFavorite`

Quick check used by browse navigation:

```asm
; in:  X/Y = game index (word, but Y is almost always 0 for <256 games)
; out: Z=0 if favorite, Z=1 if not
IsFavorite
    txa
    lsr            ; divide by 8 to get byte offset
    lsr
    lsr
    tax
    lda gGameToLaunch  ; use low 3 bits for bit position
    and #$07
    tay
    lda gFavoriteBits,x
    and FavBitmask,y
    rts
```

### `BrowseNextFavorite` / `BrowsePrevFavorite`

Wrappers around `OnBrowseNext`/`OnBrowsePrevious` that skip non-favorites:

```
BrowseNextFavorite:
    if gFavoritesMode == 0, just call OnBrowseNext (normal behavior)
    if gFavoritesMode != 0:
        save starting position
        loop:
            call OnBrowseNext
            check IsFavorite
            if favorite, done
            if wrapped back to starting position, beep (no favorites)
```

---

## 5. UI Changes

### Favorite indicator in the overlay

When the current game is a favorite, show a star or heart character in the UI bar. The HGR font already has special characters (e.g., $16 = padlock). Pick an unused character slot for a star/heart, or reuse an existing one.

In `DrawUI` (`ui.overlay.a`), after building UILine2, check `IsFavorite` for the current game and prepend a marker character if set.

### Favorites mode indicator

When `gFavoritesMode` is active, modify the instructions line (UILine2) to show something like:

```
[FAVORITES - Ctrl-V to exit]  5 games
```

instead of the normal:

```
[Type to search, ? for help ]  511 games
```

The `VisibleGameCount` string in `ui.overlay.a` would need to be updated dynamically when entering/exiting favorites mode.

---

## 6. Browse Mode Changes (`ui.browse.mode.a`)

### New dispatch entries

Add to `BrowseKeys` / `BrowseKeyDispatch`:

```
!byte $86        ; Ctrl-F = toggle favorite
!byte $96        ; Ctrl-V = toggle favorites view
```

Increment `kNumBrowseKeys` by 2.

Add handlers to dispatch table:

```
kBrowseFavorite = 15   ; (next available index)
kBrowseFavView  = 16

BrowseDispatchTableLo:
    ...existing entries...
    !byte <OnToggleFavorite
    !byte <OnToggleFavView

BrowseDispatchTableHi:
    ...existing entries...
    !byte >OnToggleFavorite
    !byte >OnToggleFavView
```

### `OnToggleFavView`

```
OnToggleFavView:
    lda gFavoritesCount
    beq SoftBell           ; no favorites, beep
    lda gFavoritesMode
    eor #$FF
    sta gFavoritesMode
    beq @exitFavMode

@enterFavMode:
    ; find first favorite game
    ldx #0 / ldy #0
    loop: check IsFavorite, if yes set gGameToLaunch and break
    jmp ForceBrowseChanged

@exitFavMode:
    jmp ForceBrowseChanged  ; redraw with normal game count
```

### Modified navigation

In `OnBrowseNext` and `OnBrowsePrevious`: check `gFavoritesMode`. If active, after advancing, loop until a favorite is found (or we've wrapped around).

---

## 7. Search Mode Changes (`ui.search.mode.a`)

Mirror the same two key bindings. When entering favorites mode from search mode, jump to `BrowseMode` (favorites mode doesn't make sense with text search).

---

## 8. Init Changes (`4cade.init.a`)

After the search index and cache are loaded, call `LoadFavorites` to populate the bitfield. This needs to happen after `gSearchStore` is populated so we can look up filenames.

---

## 9. Files to Create/Modify

| File | Change |
|------|--------|
| **NEW** `src/ui.favorites.a` | All favorites logic: LoadFavorites, ToggleFavorite, IsFavorite, BrowseNextFavorite, BrowsePrevFavorite, OnToggleFavView, SaveFavorites |
| `src/4cade.a` | `!source "src/ui.favorites.a"`, add gFavoritesMode/gFavoritesCount globals |
| `src/4cade.init.a` | Call LoadFavorites after search index load |
| `src/ui.browse.mode.a` | New key bindings, modify OnBrowseNext/OnBrowsePrevious for favorites mode |
| `src/ui.search.mode.a` | New key bindings |
| `src/ui.overlay.a` | Favorite indicator, favorites mode label |
| `src/constants.a` | gFavoriteBits address constant |
| `src/prodos.path.a` | kFavoritesFilename string |
| `Makefile` | Ensure FAVORITES.CONF is created on first run (or handle missing file gracefully) |

---

## 10. Risks and Considerations

- **Memory pressure**: 64 bytes for the bitfield is minimal. The favorites OKVS buffer ($0900) is temporary (same slot used by help/credits). No permanent memory cost beyond the 64-byte bitfield and ~2 bytes of state.

- **Disk writes**: Saving favorites uses `SaveSmallFile` (1 block / 512 bytes). This is the same mechanism as prefs and is proven reliable. But frequent saves during a session add wear. Consider batching: only save on exit from favorites toggle UI, not on every add/remove.

- **Code size**: The LC RAM bank 1 main program currently ends around $FF91 and has some room before $FFFA. The new code (~200-300 bytes estimated) needs to fit. Monitor the assembler's RELBASE warning.

- **Game count > 255**: The current codebase uses 16-bit game indices (`gGameToLaunch` is a word). The bitfield approach handles up to 511 games (64 bytes * 8 bits). If the collection ever exceeds 511 games, increase the bitfield to 128 bytes.

- **No FAVORITES.CONF on first run**: `LoadFavorites` must handle the case where the file doesn't exist (just initialize everything to zero, no favorites).

- **Search index rebuild**: When games are added/removed and the disk image is rebuilt, `FAVORITES.CONF` stores filenames so favorites survive. But the bitfield must be recalculated each boot (which `LoadFavorites` does).

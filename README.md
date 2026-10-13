# cwm (theseus-chimaira fork)

This is [Leah Neukirchen's portable cwm](https://github.com/leahneukirchen/cwm)
(`linux` branch, tracking OpenBSD's cwm) with a few local patches on top.
See [`README`](README) for the upstream description, build requirements and
changelog.

Build and test as upstream:

```sh
make
./cwm -n -c ~/.cwmrc    # parse the config only, then exit
```

Each patch is a separate commit on `linux`, so it can be reviewed, reverted
or rebased onto a new upstream release on its own.

## Patches

### Outline move and resize

While a window is dragged with the mouse (`window-move`, `window-resize`),
cwm draws an XOR outline of the frame on the root window and leaves the
window itself where it is. The window is moved or resized once, when the
button is released.

### autogroup placement: `maximize`, `fullscreen`, `left`, `right`, `screen`

`autogroup` takes an optional placement and an optional monitor:

```
autogroup group [placement [screen n]] [windowname,]windowclass
```

| placement    | effect                                              |
|--------------|-----------------------------------------------------|
| `maximize`   | maximize (gap and border are kept)                  |
| `fullscreen` | fullscreen (gap and border removed)                 |
| `left`       | fill the left half of the monitor's work area       |
| `right`      | fill the right half of the monitor's work area      |

`screen n` first moves the window onto monitor `n` (0-based, in RandR order),
then applies the placement. A placement cannot be combined with
`windowname,`. Examples:

```
autogroup 1 maximize screen 0 "firefox"
autogroup 5 left     screen 0 "signal"
autogroup 5 right    screen 0 "Threema Beta"
autogroup 0 fullscreen screen 1 XTerm
```

`left` and `right` let two applications share one monitor inside the same
group. All placement rules are one optional grammar rule, so new placements
do not need a new rule for every combination. Documented in `cwmrc(5)`.

### Group names by purpose, no group 9

Groups are named after what they hold instead of `one` ... `nine`; every name
is at most six characters so bars such as polybar's `xworkspaces` stay narrow:

| # | name     |
|---|----------|
| 0 | `ALL`    |
| 1 | `WEB`    |
| 2 | `MAIL`   |
| 3 | `TEAMS`  |
| 4 | `OFFICE` |
| 5 | `CHAT`   |
| 6 | `RDP`    |
| 7 | `MUSIC`  |
| 8 | `MISC`   |

Group 9 is gone, together with `group-toggle-9`, `group-only-9`,
`group-close-9`, `window-movetogroup-9` and the default `CM-9` binding.
`autogroup` accepts groups 0-8. A config that still refers to group 9 fails
to parse (check with `cwm -n`).

The names can still be changed at runtime by setting `_NET_DESKTOP_NAMES` on
the root window.

### Fix: busy loop when a client sets `_NET_DESKTOP_NAMES`

cwm handles a `PropertyNotify` for `_NET_DESKTOP_NAMES` by reading the names
and writing the property back. The write raises another `PropertyNotify`, so
as soon as any client set the property, cwm rewrote it forever (about a full
CPU core and tens of thousands of events per second). cwm now only writes the
property when the names actually differ.

### `_NET_WM_STATE_ABOVE` (always on top)

cwm ignored `_NET_WM_STATE_ABOVE`, so dialogs that ask to stay on top -
`pinentry-qt`'s PIN prompt, for example - could end up behind the window that
triggered them. Now:

- the state is honoured when cwm manages a window and when a client requests
  it with a `_NET_WM_STATE` client message;
- such windows are raised again after every raise, after a window is mapped
  and after a group is shown, so they stay above all other windows;
- the state is written back to `_NET_WM_STATE` and listed in
  `_NET_SUPPORTED`.

- a client that restacks itself (`XRaiseWindow`, i.e. a `ConfigureRequest`
  with a stack mode) cannot cover them either.

There is no key binding to toggle it; it is driven by the client.

### Fix: windows mapped again from the iconic state stay hidden

`pinentry-qt` maps its dialog iconic first and then maps it again with a
normal initial state. cwm ignored a `MapRequest` for a window it already
managed, so the PIN dialog never appeared - on top or not. Following ICCCM
4.1.4, a `MapRequest` for an iconic window now shows it, and
`_NET_WM_STATE_HIDDEN` is cleared when a window is shown. Windows that ask to
start iconic still do.

## Testing

The new patches were tested in a nested X server (`Xephyr :7`) before being
used on a real display:

- `left` / `right` placement: two windows matching `left` and `right` on a
  1200x700 screen came up at x=0 and x=600, each 598x698 (border 1, gap 0);
  `maximize` still gives 1198x698;
- the desktop-names fix, by setting `_NET_DESKTOP_NAMES` from a client and
  measuring cwm's CPU time and the property-change rate (unpatched: about a
  full core and ~26000 events in 2 s; patched: one event, then idle);
- `_NET_WM_STATE_ABOVE`, by requesting the state for a window and checking
  with `_NET_CLIENT_LIST_STACKING` that it stays on top when another window is
  raised or focused;
- the pinentry fix, with a real `pinentry-qt`: the dialog is mapped and on
  top after it opens, after another window is activated, after that window
  raises itself and after it is clicked; a window with
  `initial_state=IconicState` still starts hidden;
- group names and the group count, by reading `_NET_DESKTOP_NAMES` and
  `_NET_NUMBER_OF_DESKTOPS`; `autogroup 9 ...` is rejected by `cwm -n`.

The outline move/resize and the `maximize` / `fullscreen` / `screen` options
are older patches that have been in daily use; they were not re-tested in
Xephyr.

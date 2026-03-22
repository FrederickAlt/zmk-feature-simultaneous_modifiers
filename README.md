# zmk-feature-simultaneous_modifiers

A ZMK module that gives layer-switching and modifier keys time to activate before a regular key's binding is resolved in case of close to simultaneous presses. This eliminates missed or wrong-layer keystrokes when typing quickly with modifiers like shift, hold-tap or momentary layer (`&mo`) keys.

## The Problem

When you press a regular key and a modifier key (such as shift, an `&mo` layer key or a `&mt` mod-tap) simultaneously, ZMK resolves the regular key's binding the moment it arrives — sometimes before the modifier has had a chance to register. The result is that the key fires on the wrong layer, or without the intended modifier.

## How It Works

The module listens for every key position event before the rest of ZMK sees it and divides all keys into two classes:

**Modifier keys** are a set of key positions you configure explicitly. These are keys bound to layer-switching or modifier behaviors — `&mo`, `&lt`, `&mt`, `&sk`, physical shift, etc. Modifier keys are never deferred. They pass through the event chain immediately so that hold-tap and layer logic can begin resolving as quickly as possible.

**Regular keys** are everything else. When a regular key is pressed, the module holds it in a one-key buffer and starts a timer. The key is not forwarded to the rest of ZMK until one of the following happens:

- The timer expires (default 50ms)
- A modifier key arrives while the key is buffered — the timer is shortened to 1ms so the key fires quickly after the modifier. The modifier can be anything from shift to complex layer hold tap behaviors.
- A modifier key is already held when the key is pressed — the key passes through immediately since the layer is already active
- A second regular key is pressed — the first is passed on and the second is buffered in its place
- The key is released before the timer fires — the press and release are both delivered in order

By the time the regular key reaches `zmk_keymap`, the modifier or layer key has had time to start for example its hold-tap decision. If the modifier is a hold-tap configured as hold-preferred, it will decide hold and activate the layer before the regular key's binding is looked up.

### Example

```
t=0ms   'a' pressed → captured, 50ms timer starts
t=15ms  &mo 1 pressed → modifier, bubbles through immediately
                         hold-tap starts, undecided
         timer rescheduled to 1ms
t=16ms  timer fires → 'a' sent through chain
                       hold-tap sees 'a' while undecided
                       decides hold → layer 1 activates
                       'a' resolves on layer 1  ✓
```

Without this module, `'a'` would have arrived at the keymap at t=0ms, before `&mo 1` was even pressed, and resolved on the base layer.

## Installation

Add the module to your `config/west.yml`:

```yaml
manifest:
  remotes:
    - name: frederickalt
      url-base: https://github.com/FrederickAlt
  projects:
    - name: zmk-feature-simultaneous_modifiers
      remote: frederickalt
      revision: main
  self:
    path: config
```

Then enable the module in your `config/<keyboard>.conf`:

```
CONFIG_ZMK_KEY_LAYER_DEFER=y
CONFIG_ZMK_KEY_LAYER_DEFER_MS=50
CONFIG_ZMK_KEY_LAYER_DEFER_MOD_POSITIONS="30 31 32 33"
```

Note that the above configuration is only necessary in the central/dongle in case of splits or dongle setups.

## Configuration

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `CONFIG_ZMK_KEY_LAYER_DEFER` | bool | `n` | Enable the module |
| `CONFIG_ZMK_KEY_LAYER_DEFER_MS` | int | `50` | How long in milliseconds to hold a regular key before forwarding it. Any modifier press occurring within this window will be seen by the keymap before the regular key resolves. |
| `CONFIG_ZMK_KEY_LAYER_DEFER_MOD_POSITIONS` | string | `""` | Space-separated list of key position numbers that should be treated as modifier keys and passed through immediately. Include every key bound to `&mo`, `&lt`, `&mt`, `&sk`, or a physical modifier like shift. |

Key position numbers are the same indices used in combo `key-positions` — the zero-based index of each key in your keymap's `bindings` array.

## Example: Shift and a Regular Key

Consider a 36-key board where the right shift is at position 35 and is bound to `&sk LSHFT` or `&mt LSHFT X`.

`config/<keyboard>.conf`:
```
CONFIG_ZMK_KEY_LAYER_DEFER=y
CONFIG_ZMK_KEY_LAYER_DEFER_MS=30
CONFIG_ZMK_KEY_LAYER_DEFER_MOD_POSITIONS="35"
```

Now when you type shift+a quickly:

1. `'a'` is pressed first — captured, timer starts
2. Shift arrives 20ms later — passes through immediately, hold-tap begins
3. Timer fires 1ms later (rescheduled) — `'a'` enters the chain, hold-tap sees it while undecided, decides hold, shift activates
4. `'a'` resolves as `A` ✓

Without the module, `'a'` would have fired on its own before shift was even registered.

## Known Limitations

**Combo timeouts**: This module delays all regular key events before combo.c sees them, consuming part of each combo's timing window.

**Single-key buffer**: Only one regular key can be buffered at a time. This implementation is not (yet?) intended to chord together more than a two key interaction.

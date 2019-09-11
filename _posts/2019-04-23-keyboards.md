---
layout: post
title:  "Keyboard design"
date:   April 23, 2019
categories: keyboards
---

Traditional keyboard design is based on typewriters, which had to handle the
mechanics of the moving keys.  Modern keyboards no longer have such constraints,
and modern keyboard design has evolved considerably.  In this post I will go
over various aspects of keyboard design.

I currently use Dvorak layout, with control/escape dualed on the caps lock key,
and control/enter dualed on the enter key.  Dual-use keys, expanded on below,
are keys which have one use when tapped and another use when held down.  I'm
currently using auto-hotkey software for key re-mapping.

## The Physical Keyboard

The aim of physical keyboard design is to allow the body to rest naturally.
Ergonomic keyboards are designed to adjust to the body, rather than have the
body adjust to the keyboard.

### Split vs Combined

The standard keyboard places both hands quite close together.  This can cause
your shoulders to collapse inwards, especially if one has broad shoulders.
Split keyboards allow the elbows and wrists to be aligned naturally with the
shoulders.

### Tented vs Flat

Tented keyboards are angled, as if to form a tent, instead of lying flat
parallel to a table.  Even with split keyboards, the wrists need to be pronated
to align to a keyboard lying flat on a table.  Tenting allows the wrist to be
angled naturally.

### Ortholinear vs Staggered

Traditional keyboards have the keys staggered.  Ortholinear keyboards have the
keys stacked vertically.  This allows the fingers to extend naturally, as
opposed to reaching diagonally to the left.

### Thumb Clusters

Thumb clusters are featured on many modern specialty keyboards.  However, the
design of thumb clusters is new and relatively less mature.  Thumbs are aligned
perpendicularly to fingers, so to press keys on the same plane as the rest of
the keyboard, the thumb has to strike from a lateral motion.  I find this
straining, and have so far not found thumb clusters comfortable.  I imagine a
keyboard shaped like two shelves would be more comfortable for thumbs.

### Smaller Spacebar

The spacebar on a standard keyboard is quite large, spanning six regular keys
(6u).  An Apple keyboard actually has a slightly smaller spacebar, spanning five
regular keys, with command buttons on either side, easily reachable with thumbs.
Happy Hacking Keyboard (HHKB) and Planck keyboards have 2u spacebars, taking
that concept further.

## Keyboard Layout

The standard keyboard layout, QWERTY, was designed to prevent typewriter keys
from jamming and not for typing ergonomics or efficiency.  Further, the control
key and other modifiers have become quite prominent, especially for programmers,
but have remained in very inconvenient positions.

### Base Layout

The most popular alternative to QWERTY is Dvorak.  Dvorak has two major
advantages: first, it favours the home row and second, it makes left and right
hands alternate more frequently.  Another popular alternative, Colemak, is
designed to allow fingers to "roll" by placing common sequences of characters
adjacent to each other.  Personally, I do not find the rolling motion
comfortable.  Keep in mind that this is based on the English language.  Some
people claim that one types faster on Dvorak.  In my experience, I don't type
faster, but I do type more comfortably.

### Caps-lock

There's little reason to dedicate a whole key, let alone one in prime real
estate, to the caps lock function.  It's a common practice to remap it to
control.

### Symmetry in Modifiers

On the standard keyboard, there are two shift keys: one on each side.  This
allows one to avoid having to hold shift and press a key with the same hand.
Following that rationale, I think all modifier keys should be symmetrical.  For
some time, I had put control where caps lock is, which made it much easer to
press in general, but difficult to press with keys under my left ring finger or
pinky (e.g. ctrl-A).  Since then, I've placed a control on either side of the
keyboard.

### One-key-away Principle

40% keyboards are designed with a one-key-away principle, meaning that each
keystroke is at most one key away from the home row.  Given that the number of
keys is reduced, there needs to be methods to combine more functions into each
key.

### Methods to reduce key footprint

#### Layers

The Planck keyboard popularized the concept of "layers" wherein on each layer,
the physical keys map to a different symbol on each layer.  While physically
that is optimal, it requires much more practice to ingrain this into muscle
memory so that it's not mentally taxing.

#### Dual-use Keys

Another way to get more mileage per key is to double up modifier keys (e.g.
control)  and single-tap keys (e.g. escape).  A *modifier* key is one that when
held down modifies the meaning of a key but tapped by itself does nothing.
Therefore, it can be combined with a key that only needs to be tapped and does
not need to be held down.

Currently, I've combined control and escape into a single key.

### Keyboard Shortcut Design

Keyboards aren't only used for textual input; they are also used for application
specific commands or keyboard shortcuts.  Such commands may be chords or
sequences of keystrokes.  A *chord* is when multiple keys are pressed
simultaneously.  A *sequence* is when multiple keys are pressed one at a time in
sequence.  For example, emacs is a chord-heavy program and spacemacs is a
sequence heavy program.  Such use cases should be considered when designing a
keyboard layout, especially if layers are a factor.

## Other Considerations

It is worth considering situations may require the use of a standard keyboard
(colleagues, public computers), or even laptops which may have software key
re-mapping support, but are physically different from your everyday keyboard.
If switching keyboards will be required regularly, the mental effort may
outweigh the benefits of having a significantly different keyboard.

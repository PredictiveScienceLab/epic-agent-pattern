# phi-script: the language of the ultrasound example

The agent does not draw. It writes a **phi-script**: a short program that deforms one fixed
reference outline (a cardiologist's tracing of another patient) onto the image. The script defines a
diffeomorphism φ of the plane, so the outlines can move, stretch and bend but never tear, fold or
cross each other.

This is the language section of the agent's guide, lightly edited.

## Conventions

- One statement per line; `#` starts a comment. Statements apply in order, each to the result of the
  previous ones. The whole file is re-run every time, so earlier lines can be edited or deleted.
- **Coordinates** are millimetres: x to the right, y down, origin at the top-left of the image.
- **Stations.** Positions along a border are stations 0 to 10: 0 = screen-left end, 5 = apex (for the
  atrium, its far wall), 10 = screen-right end. Stations are glued to the outline: "endo 7" is always
  the same piece of wall, wherever earlier statements moved it. Fractions are allowed (`6.5`).
- **out / in.** `out` = away from the chamber the border encloses; `in` = towards it.
- **Borders:** `endo` (cavity border), `epi` (outer border of the heart muscle), `la` (left atrium),
  `wall` (endo and epi together, thickness kept).

## Statements

```
align apex=(x,y) hinge_l=(x,y) hinge_r=(x,y)
```
Poses the whole reference: the affine map that sends the reference's apex and two valve hinges to
the three points given. Always the first statement. Later statements are relative, so editing these
numbers re-poses everything and the rest follows.

```
move LANDMARK by dx dy [span n]        move LANDMARK to x y [span n]
```
Drags one landmark; the border within `span` stations of it follows smoothly. LANDMARK is `apex`,
`hinge_l`, `hinge_r`, `epi_apex`, `epi_l`, `epi_r` or `la_roof`.

```
push BORDER a..b out|in MM [taper n]
offset BORDER station:mm station:mm ... [taper n]
```
Moves a stretch of one border along its normal, fading to nothing over `taper` stations beyond each
end. `offset` sets several signed amounts at once (+ = out) with a smooth curve between them. At most
15 mm per statement.

```
thicken a..b MM        thin a..b MM
```
Changes wall thickness over stations a..b by moving only the epicardium.

```
smooth BORDER a..b [MM]
```
Moves a stretch onto a smoothed copy of itself, to remove kinks left by earlier edits.

```
atrium depth F width F
```
Resizes the atrium below the valve (1.0 = unchanged).

```
translate dx dy     rotate DEG [about P]     scale F [about P]
stretch long|across F [about P]              shear long|across K [about P]
```
Whole-picture affine changes, rarely needed after `align`.

## What the executor guarantees

Each statement compiles to affine maps and small kernel steps whose stretch is certified below a
bound (Lipschitz constant below 0.9), so each step, and therefore φ, is invertible. Guards then check
the result. A statement is **refused**, with the reason, when it would push one border through
another or fold a tight curve onto itself; the agent sees the state before that line.

## A complete example

The final program of one held-out case is in [`files/example_program.phi`](files/example_program.phi)
(15 statements; it compiles to one affine map and 93 certified kernel steps). Its first lines:

```
align apex=(82,30) hinge_l=(64,93) hinge_r=(102,92)   # cavity tip under near-field haze; roots of closed mitral leaflets
push wall 6..9.5 in 7 taper 1.2    # anterior wall: inner edge of bright band is at x~103-106, line was at x~115
atrium depth 0.70 width 1.0        # dark atrial pool ends at y~119, reference roof was at y~133
move la_roof by 9 -1 span 3        # atrial pool is centred under the valve (x 62..113), reference leaned left
```

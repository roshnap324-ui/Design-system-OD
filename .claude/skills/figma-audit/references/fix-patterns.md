# Fix Patterns

Apply only approved fixes. Group by type, one atomic `use_figma` call per group,
`skillNames: "figma-use"`, always a `description`. Re-screenshot after.

Fetch the target and token map at the top of each call:

```js
const vars = await figma.variables.getLocalVariablesAsync();
const V = Object.fromEntries(vars.map(v=>[v.name,v]));
```

## Clear a stray fill (unbound opaque white container)

The most common fix. The container was never meant to be a surface — it inherited
`createAutoLayout`'s default white. Make it transparent so it shows the parent's
background:

```js
for (const id of ["NODE_ID_1","NODE_ID_2"]) {
  const n = await figma.getNodeByIdAsync(id);
  if (n) n.fills = [];
}
```

## Bind a color to its DS token

When a hex matches a token but isn't bound, replace the raw paint with a bound
one so it stays correct if the token changes:

```js
const n = await figma.getNodeByIdAsync("NODE_ID");
const varr = V["color/red"];            // the matched token
n.fills = [ figma.variables.setBoundVariableForPaint(
  { type:"SOLID", color:{r:0,g:0,b:0} }, "color", varr ) ];
```

For strokes use `n.strokes = [...]` the same way.

**Translucent overlays:** binding-then-`paint.opacity = x` does not always stick.
If you need a semi-transparent DS color and the bound opacity reads back as 1,
set a raw paint instead — this is a treatment, not a token value:

```js
n.fills = [{ type:"SOLID", color:{r:1,g:1,b:1}, opacity:0.14 }];  // e.g. social chip on dark
```

## Robustly (re)bind a text fill

`setBoundVariableForPaint` binding does not reliably persist on TEXT nodes (and a
prior `setRangeFills` can leave the base fill unbound at `#000000`). Belt-and-
suspenders: set the resolved literal color **and** bind, so the color is correct
even if the binding is later dropped. Load the font is not needed for fills.

```js
function toRgb(hex){hex=hex.replace('#','');return {r:parseInt(hex.slice(0,2),16)/255,g:parseInt(hex.slice(2,4),16)/255,b:parseInt(hex.slice(4,6),16)/255};}
function bindTextFill(node, variable, resolvedHex){
  const paint = figma.variables.setBoundVariableForPaint(
    { type:"SOLID", color: toRgb(resolvedHex) }, "color", variable);
  node.fills = [paint];                    // whole node
}
// whole-node example: an eyebrow that should be red
const n = await figma.getNodeByIdAsync("NODE_ID");
bindTextFill(n, V["color/red"], "#ED1C24");

// range example (colored word inside a heading): rebuild each range explicitly
const t = await figma.getNodeByIdAsync("HEADING_ID");
t.fills = [figma.variables.setBoundVariableForPaint({type:"SOLID",color:toRgb("#FFFFFF")},"color",V["color/white"])]; // base
const s = t.characters.indexOf("UPGRADED");
t.setRangeFills(s, s+8, [figma.variables.setBoundVariableForPaint({type:"SOLID",color:toRgb("#ED1C24")},"color",V["color/red"])]);
```

After rebinding, verify with the **server `get_screenshot`** (the plugin render can
show the intended color even when the fix didn't take).

## Bind / correct a corner radius

```js
const n = await figma.getNodeByIdAsync("NODE_ID");
n.setBoundVariable("topLeftRadius", V["radius/md"]);
n.setBoundVariable("topRightRadius", V["radius/md"]);
n.setBoundVariable("bottomLeftRadius", V["radius/md"]);
n.setBoundVariable("bottomRightRadius", V["radius/md"]);
// or, if not binding: n.cornerRadius = 8;  (snap to nearest allowed radius)
```

## Fix a wrong font family

Load the correct font before assigning, and preserve the weight where sensible
(map the foreign weight to the nearest allowed Rubik/Noto Sans style):

```js
const n = await figma.getNodeByIdAsync("NODE_ID");
const target = { family:"Rubik", style:"SemiBold" };   // headings → Rubik, body → Noto Sans
await figma.loadFontAsync(target);
n.fontName = target;   // for mixed runs, use setRangeFontName over the affected range
```

## Resolve clipped text

Prefer letting the text grow over hand-sizing its box:

```js
const n = await figma.getNodeByIdAsync("NODE_ID");
n.textAutoResize = "HEIGHT";      // box grows to fit; if inside auto-layout it flows
```

If it must stay fixed height, increase the height to at least the line height, or
reduce font size — but auto-resize is the durable fix.

## Snap off-pixel geometry

```js
const n = await figma.getNodeByIdAsync("NODE_ID");
n.x = Math.round(n.x); n.y = Math.round(n.y);
if ("resize" in n) n.resize(Math.round(n.width), Math.round(n.height));
```

Note: inside an auto-layout parent, position/size are computed — fix the parent's
spacing/sizing instead of nudging the child. Fractional geometry usually
originates from a hand-placed node that should be in an auto-layout flow.

## Equalize asymmetric horizontal padding

```js
const n = await figma.getNodeByIdAsync("SECTION_ID");
n.paddingLeft = 20; n.paddingRight = 20;   // match the DS section gutter
```

## Reattach / flag a detached instance

Detached instances usually can't be safely auto-reattached (the original overrides
are gone). Do **not** guess — report the node id and the component it likely came
from, and let the user swap it deliberately with the component picker, or confirm
before you `swapComponent` to a chosen main.

## After any fix batch

```js
const root = await figma.getNodeByIdAsync("TARGET");
await root.screenshot();      // eyeball it
return { fixed: ["NODE_ID_1","NODE_ID_2"] };   // report ids so the user can undo selectively
```

Then re-run the relevant audit lens from `audit-checks.md` and confirm the
findings cleared before telling the user it's done.

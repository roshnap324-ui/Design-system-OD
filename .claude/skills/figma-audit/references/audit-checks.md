# Audit Checks — traversal scripts & rules

Read-only `use_figma` scripts, one per lens. Each returns structured findings;
none mutate the canvas. Replace `TARGET` with the node id and pass
`skillNames: "figma-use"` and a `description` on every call.

## Traversal gotchas (why the scripts look the way they do)

These are the ways a naive traversal breaks in the `use_figma` context:

- **Some properties are getters that throw.** Accessing `node.findAllWithCriteria`
  on a `RECTANGLE` (or any non-container) throws — even `typeof node.findAllWithCriteria`
  throws. Guard by `node.type`, don't feature-detect. Use `node.findAll(pred)`
  only on container types.
- **`node.findAll(() => true)` from the target walks the whole subtree** and works
  on frames/groups/components/instances. Use it as the base, then filter.
- **Fill opacity may not read back the way you set it.** Read
  `fills[0].opacity` defensively (`?? 1`).
- **A fill can be bound to a variable.** Check `node.boundVariables?.fills` before
  concluding a color is "hardcoded".
- **Image fills exist.** `fills.find(f => f.type === "IMAGE")` — don't treat an
  image frame as a missing color.

## Loading the token map

Run once; reuse the returned maps in your reasoning (not across `use_figma`
calls — re-fetch per call, it's cheap).

```js
// Ooredoo DS variable map: name -> {id, hex}
const vars = await figma.variables.getLocalVariablesAsync();
function toHex(c){const h=n=>Math.round(n*255).toString(16).padStart(2,"0");return "#"+h(c.r)+h(c.g)+h(c.b);}
const map = {};
for (const v of vars) {
  if (v.resolvedType !== "COLOR") { map[v.name] = { id: v.id, type: v.resolvedType }; continue; }
  const modeId = Object.keys(v.valuesByMode)[0];
  const val = v.valuesByMode[modeId];
  map[v.name] = { id: v.id, hex: val && val.r !== undefined ? toHex(val).toUpperCase() : null };
}
return map;   // e.g. {"color/red":{id:"...",hex:"#ED1C24"}, "radius/pill":{id:"...",type:"FLOAT"}}
```

Also grep `ooredoo_design_system_complete-2.html` `:root` for the authoritative
hex list. Build `hex -> tokenName` by inverting the color entries so the audit
can say "this #ED1C24 should be bound to `color/red`".

Allowed fonts: `Rubik` (Regular/Medium/SemiBold/Bold), `Noto Sans`
(Regular/Medium/SemiBold/Bold). Allowed radii: `4, 8, 12, 16, 100`.

## A. Alignment & layout

```js
const CONTAINERS = ["FRAME","GROUP","COMPONENT","INSTANCE","SECTION","COMPONENT_SET"];
const root = await figma.getNodeByIdAsync("TARGET");
const all = root.findAll(() => true);
const f = { offPixel:[], edges:[], padding:[], overlap:[], clipped:[] };

// 1. Off-pixel geometry
for (const n of all) {
  if (["x","y","width","height"].some(k => typeof n[k]==="number" && Math.abs(n[k]-Math.round(n[k]))>0.5))
    f.offPixel.push({id:n.id,name:n.name,x:n.x,y:n.y,w:n.width,h:n.height});
}
// 2. Vertical auto-layout stacks: children leading edges should match (auto-layout guarantees it;
//    flag NON-auto-layout containers whose children x drift)
for (const n of all) {
  if (!CONTAINERS.includes(n.type) || !("children" in n) || n.children.length < 3) continue;
  if (n.layoutMode && n.layoutMode !== "NONE") continue;          // auto-layout handles alignment
  const xs = n.children.map(c=>Math.round(c.x));
  const spread = Math.max(...xs) - Math.min(...xs);
  if (spread > 2) f.edges.push({id:n.id,name:n.name,xSpread:spread});
}
// 3. Inconsistent padding/gap within one auto-layout container is usually intentional per-section,
//    so only flag a container whose OWN four paddings are asymmetric unexpectedly:
for (const n of all) {
  if (!("layoutMode" in n) || !n.layoutMode || n.layoutMode==="NONE") continue;
  const p=[n.paddingTop,n.paddingRight,n.paddingBottom,n.paddingLeft].map(v=>v||0);
  // horizontal padding mismatch (left vs right) on a full-width section is a common slip
  if (Math.abs(p[1]-p[3])>1) f.padding.push({id:n.id,name:n.name,padding:p});
}
// 4. Clipped text: fixed-size text whose box is shorter than one line needs
for (const n of all) {
  if (n.type!=="TEXT") continue;
  if (n.textAutoResize==="NONE" || n.textAutoResize==="TRUNCATE") {
    const lh = (n.lineHeight && n.lineHeight.value) || n.fontSize*1.3;
    if (n.height < lh - 1) f.clipped.push({id:n.id,name:n.name,h:Math.round(n.height),needs:Math.round(lh),chars:n.characters.slice(0,40)});
  }
}
return {scanned:all.length, ...f};
```

Notes: `edges` and `padding` are hypotheses — a designer may want asymmetric
padding. Confirm against the screenshot before reporting as High.

## B. DS token compliance

```js
const root = await figma.getNodeByIdAsync("TARGET");
const all = root.findAll(() => true);
const RADII=[4,8,12,16,100];
const OKFONT={"Rubik":1,"Noto Sans":1};
const f={strayWhite:[],unbound:[],offToken:[],badFont:[],badRadius:[]};
function isSolid(p){return p&&p.type==="SOLID"&&p.color;}
function hex(c){const h=n=>Math.round(n*255).toString(16).padStart(2,"0");return ("#"+h(c.r)+h(c.g)+h(c.b)).toUpperCase();}
// A fill is bound if EITHER the paint carries boundVariables.color (the
// modern setBoundVariableForPaint form) OR the node exposes boundVariables.fills.
// Check both — text fills and frame fills store this differently, and a range
// operation (setRangeFills) can drop the base binding while keeping the node's.
function fillBound(node, paint){
  if (paint && paint.boundVariables && paint.boundVariables.color) return true;
  return !!(node.boundVariables && node.boundVariables.fills && node.boundVariables.fills.length);
}
for (const n of all) {
  // fills
  if (Array.isArray(n.fills) && n.fills.length===1 && isSolid(n.fills[0])) {
    const fp=n.fills[0], bound = fillBound(n, fp);
    const op = fp.opacity ?? 1;
    const white = fp.color.r>0.98&&fp.color.g>0.98&&fp.color.b>0.98;
    if (white && op===1 && !bound && ["FRAME","COMPONENT","INSTANCE"].includes(n.type))
      f.strayWhite.push({id:n.id,name:n.name,parent:n.parent&&n.parent.name});
    else if (!bound && op===1)
      f.unbound.push({id:n.id,name:n.name,type:n.type,hex:hex(fp.color)});
  }
  // fonts
  if (n.type==="TEXT") {
    try { n.getStyledTextSegments(["fontName"]).forEach(s=>{ if(!OKFONT[s.fontName.family]) f.badFont.push({id:n.id,name:n.name,family:s.fontName.family}); }); } catch(e){}
  }
  // radii
  if (typeof n.cornerRadius==="number" && n.cornerRadius>0 && !RADII.includes(Math.round(n.cornerRadius)))
    f.badRadius.push({id:n.id,name:n.name,r:n.cornerRadius});
}
return {scanned:all.length, ...f};
```

After this returns, cross-reference each `unbound` hex against the token map:

- hex **matches** a DS token → Medium: "bind to `color/<name>`".
- hex **matches no** token → decision (not auto-fix): map to nearest token, or
  propose adding it per CLAUDE.md.
- `strayWhite` → High if the node sits on a dark section, Medium otherwise; fix is
  usually `fills = []` (see fix-patterns).

**The black-text-on-dark trap (seen in the wild).** Unbound TEXT fills whose hex
is `#000000` are the signature of a `setBoundVariableForPaint` binding that didn't
persist on a text node (and `setRangeFills` can strip the base binding while
keeping a range's). On a light section black is fine; on a **dark** section it
means the copy is rendering black-on-dark — often invisible, and a plugin
`node.screenshot()` may still show it as the intended white. So when lens B
returns unbound `#000000` text, resolve its severity by the section background:
dark parent → High. Always confirm with a **server `get_screenshot`**, not the
plugin render. The durable fix is to re-set the fill to the resolved literal color
*and* re-bind (see fix-patterns → "Robustly (re)bind a text fill").

## C. Content sync

```js
const CONTAINERS=["FRAME","GROUP","COMPONENT","INSTANCE","SECTION","COMPONENT_SET"];
const PLACE=[/^title$/i,/^heading$/i,/^body$/i,/^button$/i,/lorem ipsum/i,/^text$/i,/^label$/i];
const root = await figma.getNodeByIdAsync("TARGET");
const texts = root.findAll(n=>n.type==="TEXT");
const placeholders = texts.filter(t=>!t.characters.trim() || PLACE.some(re=>re.test(t.characters.trim())))
  .map(t=>({id:t.id,name:t.name,chars:t.characters}));
return {textCount:texts.length, placeholders,
        allText: texts.map(t=>t.characters.replace(/\n/g,"⏎"))};   // for diffing vs a reference node
```

If the user supplies a reference node (e.g. the desktop source), run the same on
it and diff `allText` arrays to surface copy that's outdated or missing.

## D. Instance / component health

```js
const root = await figma.getNodeByIdAsync("TARGET");
const all = root.findAll(()=>true);
const f={detached:[],emptyText:[],defaultNames:[]};
for (const n of all){
  if (n.type==="INSTANCE"){
    // mainComponent access can be async in some contexts; guard
    let mc=null; try{ mc = n.mainComponent; }catch(e){}
    if (mc===null) f.detached.push({id:n.id,name:n.name});
  }
  if (n.type==="TEXT" && !n.characters.trim()) f.emptyText.push({id:n.id,name:n.name});
  if (/^(Frame|Group|Rectangle|Vector) ?\d*$/.test(n.name)) f.defaultNames.push({id:n.id,name:n.name});
}
return {scanned:all.length, detached:f.detached, emptyText:f.emptyText, defaultNamesCount:f.defaultNames.length, defaultNamesSample:f.defaultNames.slice(0,15)};
```

`defaultNames` is Low by itself (generated wrappers are common), but a cluster of
them around a section that looks unfinished is worth mentioning.

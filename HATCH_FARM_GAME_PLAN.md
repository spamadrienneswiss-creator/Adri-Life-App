# 🌾 Hatch Farm Game — Implementation Plan (Stardew-style)

**Goal:** Add a Stardew Valley–style farming game as a new mode inside **The Hatch** (the
DHARMA/Lost-styled overlay in `index.html`). The game is called **THE ORCHARD (Station 9)**.

This plan is written so any model can execute it step by step. Every step gives:
1. an **exact search string** to find the right spot in `index.html` (do NOT rely on line
   numbers — they shift),
2. the **exact code to paste**, and
3. a **verification step**.

Execute phases in order. Commit after each phase passes verification.

---

## Ground rules (read first)

- The entire app is ONE file: `index.html` (~1.8 MB). All CSS, HTML, and JS go in that file.
  **No external images, libraries, or fonts.** Graphics are emoji + CSS only.
- The app is mobile-first (touch). No hover-only interactions. Buttons must be tappable.
- **Never modify existing functions except where this plan explicitly says to.** All new code
  is additive and prefixed `farm`/`_farm`/`FARM` so it can't collide with existing names.
- Persistence: the app stores everything in `localStorage` with keys prefixed `a`
  (e.g. `aTasks`, `aEmber`). The game uses key **`aFarmGame`**. Note: the app overrides
  `localStorage.setItem` for cloud sync — that's fine, just call it normally.
- Aesthetic: inside `#ovHatch` everything is green-CRT-on-black (`#3fc820` on `#070d07`,
  `Courier New`, thin `#142814` borders). The new UI must match. The CSS below already does.

### How the Hatch works (architecture you're plugging into)

- The overlay is `<div id="ovHatch">`. `openHatch()` plays a boot sequence, then calls
  `hatchMode('selector')`.
- Each mode is a `<div id="hPanel<Name>">` inside `.hatch-inner`. `hatchMode(mode)` hides all
  panels listed in a hardcoded array, shows `hPanel<Mode>`, then calls that mode's show
  function. You will register the new panel in **both** places (array + dispatch) or it will
  break.
- `closeHatch()` / `hatchMode()` both call `_clearHatchTimers()` — Phase 1 needs no timers,
  so no cleanup is required. (Phase 3, if attempted, must add cleanup there — see Phase 3.)

---

## Phase 0 — Wire an empty "THE ORCHARD" mode into the Hatch

### Step 0.1 — Register the panel in `hatchMode()`

Search for this exact string (there is exactly one occurrence):

```
['Selector','Mission','Chaos','Transmission','Shock','Cast','Spa'].forEach
```

Change it to:

```
['Selector','Mission','Chaos','Transmission','Shock','Cast','Spa','Farm'].forEach
```

### Step 0.2 — Add the dispatch call

A few lines below, search for:

```
  else if(mode==='cast')_hatchCastShow();
```

Add this line **immediately after** it:

```js
  else if(mode==='farm')_hatchFarmShow();
```

### Step 0.3 — Add the selector button

Search for:

```
<button class="hatch-spa-pill" onclick="hatchMode('spa')">
```

**Immediately before** that line, paste:

```html
        <button class="hatch-mode-btn" onclick="hatchMode('farm')">
          <span class="hatch-mode-icon">🌾</span>
          <span><span class="hatch-mode-label">&gt; THE ORCHARD</span><span class="hatch-mode-sub">Station 9 — grow. harvest. profit.</span></span>
        </button>
```

### Step 0.4 — Add the panel HTML

Search for:

```
      <div id="hPanelSpa" style="display:none">
```

⚠️ **Gotcha:** there are TWO `<!-- SPA -->` comments in the file (one is a stray leftover).
Anchor on `id="hPanelSpa"` — that string is unique.

**Immediately before** that line (and before its `<!-- SPA -->` comment if you like), paste:

```html
      <!-- FARM / THE ORCHARD -->
      <div id="hPanelFarm" style="display:none">
        <div class="hatch-header">
          <button class="hatch-back-btn" onclick="hatchMode('selector')">← Back</button>
          <button class="hatch-close-btn" onclick="closeHatch()">✕</button>
        </div>
        <div class="hatch-label">🌾 STATION 9 — THE ORCHARD</div>
        <div id="hFarmHud" class="farm-hud"></div>
        <div id="hFarmGrid" class="farm-grid"></div>
        <div id="hFarmMsg" class="farm-msg">Tap a tile: till → plant → water. Sleep to grow crops.</div>
        <div id="hFarmSeeds" class="farm-seeds"></div>
        <div class="hatch-keys" style="margin-top:10px">
          <button class="hatch-key hatch-key-skip" onclick="farmShopToggle()">🛒 SHOP</button>
          <button class="hatch-key hatch-key-exec" onclick="farmSleep()">🛏️ SLEEP</button>
        </div>
        <div id="hFarmShop" style="display:none"></div>
      </div>
```

### Step 0.5 — Add a stub show function

Search for:

```
function openHatch(){
```

**Immediately before** that line, paste a temporary stub:

```js
// ─── FARM GAME (STATION 9: THE ORCHARD) ─────────
function _hatchFarmShow(){}
```

(Phase 1 replaces this stub with the real game.)

### ✅ Phase 0 verification

Open `index.html` in a browser. Open the Hatch (the **⚡ Hatch** pill on the Home tab).
After the boot sequence:
- A new **> THE ORCHARD** button appears above the SPA pill.
- Tapping it shows the (mostly empty) farm panel with SHOP/SLEEP keys.
- **← Back** returns to the selector; every OTHER mode (Mission, Chaos, Transmission, Shock,
  Cast, Spa) still opens and closes correctly. If a panel ever stays visible on top of
  another, Step 0.1 was done wrong.
- Check the browser console for errors (there must be none).

**Commit:** `Hatch: wire empty Orchard (farm game) mode`

---

## Phase 1 — Playable farming MVP

Core loop: **till → plant → water → sleep (day advances, watered crops grow) → harvest →
sell in shop → buy better seeds**. Energy limits actions per day; sleeping restores it.

### Step 1.1 — Add the CSS

Search for:

```
#hatchCanvas{position:fixed
```

**Immediately before** that line, paste:

```css
/* ── FARM GAME (THE ORCHARD) ── */
.farm-hud{display:flex;justify-content:space-between;font-family:'Courier New',monospace;font-size:12px;color:#3fc820;margin-bottom:10px;letter-spacing:1px;text-shadow:0 0 5px rgba(63,200,32,.25)}
.farm-grid{display:grid;grid-template-columns:repeat(8,1fr);gap:3px;margin-bottom:10px}
.farm-tile{aspect-ratio:1;border:1px solid #142814;border-radius:3px;background:#0c140c;font-size:15px;padding:0;cursor:pointer;line-height:1;font-family:'Courier New',monospace}
.farm-tile.tilled{background:#241708;border-color:#3a2810}
.farm-tile.wet{background:#101c26;border-color:#1d3a55}
.farm-msg{font-family:'Courier New',monospace;font-size:11px;color:#1a6a10;min-height:30px;margin-bottom:8px;line-height:1.5}
.farm-seeds{display:flex;gap:6px;flex-wrap:wrap}
.farm-seed-btn{padding:6px 10px;border-radius:3px;border:1px solid #142814;background:transparent;color:#3fc820;font-family:'Courier New',monospace;font-size:12px;cursor:pointer}
.farm-seed-btn.on{background:rgba(63,200,32,.15);border-color:#3fc820}
.farm-hint{font-family:'Courier New',monospace;font-size:11px;color:#1a5010}
.farm-shop-t{font-family:'Courier New',monospace;font-size:10px;color:#1a6a10;letter-spacing:1.5px;margin:12px 0 6px}
.farm-shop-row{display:flex;justify-content:space-between;align-items:center;gap:8px;font-family:'Courier New',monospace;font-size:11px;color:#3fc820;padding:6px 0;border-bottom:1px solid #0d200d}
.farm-buy-btn{padding:5px 10px;border-radius:3px;border:1px solid #3fc820;background:transparent;color:#3fc820;font-family:'Courier New',monospace;font-size:10px;cursor:pointer;white-space:nowrap;flex-shrink:0}
.farm-reset-btn{background:none;border:none;font-family:'Courier New',monospace;font-size:9px;color:#1a5010;cursor:pointer;margin-top:14px;letter-spacing:1px}
```

### Step 1.2 — Replace the stub with the game logic

Find the stub from Step 0.5:

```js
// ─── FARM GAME (STATION 9: THE ORCHARD) ─────────
function _hatchFarmShow(){}
```

Replace that entire stub (both lines) with **all** of the following:

```js
// ─── FARM GAME (STATION 9: THE ORCHARD) ─────────
// Plot states: {g:0}=grass  {g:1}=tilled  {g:2,c:cropKey,st:daysGrown,w:wateredToday}
var FARM_CROPS={
  turnip:{name:'Turnip',icon:'🟣',seed:5,sell:12,days:2},
  bean:{name:'Green Bean',icon:'🫛',seed:12,sell:30,days:4},
  melon:{name:'Melon',icon:'🍈',seed:25,sell:70,days:6},
  pumpkin:{name:'Pumpkin',icon:'🎃',seed:50,sell:160,days:8},
  starfruit:{name:'Starfruit',icon:'⭐',seed:120,sell:420,days:10}
};
var FARM_SIZE=8;
var FARM=null;
function _farmDefault(){
  return{day:1,coins:20,energy:50,maxEnergy:50,seed:'turnip',
    seeds:{turnip:4},barn:{},plots:[]};
}
function _farmLoad(){
  if(FARM)return;
  try{FARM=JSON.parse(localStorage.getItem('aFarmGame')||'null');}catch(e){FARM=null;}
  if(!FARM||!FARM.plots)FARM=_farmDefault();
  while(FARM.plots.length<FARM_SIZE*FARM_SIZE)FARM.plots.push({g:0});
}
function _farmSave(){localStorage.setItem('aFarmGame',JSON.stringify(FARM));}
function _hatchFarmShow(){
  _farmLoad();
  var shop=document.getElementById('hFarmShop');if(shop)shop.style.display='none';
  _farmRender();
}
function _farmTileIcon(p){
  if(p.g!==2)return '';
  var c=FARM_CROPS[p.c];
  if(p.st>=c.days)return c.icon;
  return p.st<c.days/2?'🌱':'🌿';
}
function _farmRender(){
  var hud=document.getElementById('hFarmHud');
  if(hud)hud.innerHTML='<span>DAY '+FARM.day+'</span><span>🪙 '+FARM.coins+'</span><span>⚡ '+FARM.energy+'/'+FARM.maxEnergy+'</span>';
  var grid=document.getElementById('hFarmGrid');
  if(grid){
    grid.innerHTML='';
    FARM.plots.forEach(function(p,i){
      var b=document.createElement('button');
      b.className='farm-tile'+(p.g>0?' tilled':'')+(p.g===2&&p.w?' wet':'');
      b.textContent=_farmTileIcon(p);
      b.onclick=function(){_farmTap(i);};
      grid.appendChild(b);
    });
  }
  var seeds=document.getElementById('hFarmSeeds');
  if(seeds){
    seeds.innerHTML='';
    Object.keys(FARM_CROPS).forEach(function(k){
      var n=FARM.seeds[k]||0;
      if(n<=0)return;
      var b=document.createElement('button');
      b.className='farm-seed-btn'+(FARM.seed===k?' on':'');
      b.textContent=FARM_CROPS[k].icon+' ×'+n;
      b.onclick=function(){FARM.seed=k;_farmSave();_farmRender();};
      seeds.appendChild(b);
    });
    if(!seeds.children.length)seeds.innerHTML='<span class="farm-hint">No seeds — buy some in the SHOP ↓</span>';
  }
}
function _farmMsg(t){
  var el=document.getElementById('hFarmMsg');if(el)el.textContent=t;
}
function _farmTap(i){
  var p=FARM.plots[i];
  if(p.g===0){
    if(FARM.energy<2){_farmMsg('⚡ Too tired to till — SLEEP to restore energy.');return;}
    FARM.energy-=2;FARM.plots[i]={g:1};
    _farmMsg('Tilled the soil. Tap again to plant '+FARM_CROPS[FARM.seed].name+'.');
  }else if(p.g===1){
    var k=FARM.seed;
    if((FARM.seeds[k]||0)<=0){_farmMsg('No '+FARM_CROPS[k].name+' seeds — buy some in the SHOP.');return;}
    if(FARM.energy<1){_farmMsg('⚡ Too tired to plant — SLEEP to restore energy.');return;}
    FARM.energy-=1;FARM.seeds[k]--;
    FARM.plots[i]={g:2,c:k,st:0,w:false};
    _farmMsg(FARM_CROPS[k].name+' planted 🌱 — tap to water it.');
  }else{
    var c=FARM_CROPS[p.c];
    if(p.st>=c.days){
      FARM.barn[p.c]=(FARM.barn[p.c]||0)+1;
      FARM.plots[i]={g:1};
      if(navigator.vibrate)navigator.vibrate(30);
      _farmMsg(c.icon+' '+c.name+' harvested! Sell it in the SHOP.');
    }else if(!p.w){
      if(FARM.energy<1){_farmMsg('⚡ Too tired to water — SLEEP to restore energy.');return;}
      FARM.energy-=1;p.w=true;
      _farmMsg('Watered 💧 — it will grow overnight.');
    }else{
      _farmMsg(c.name+': day '+p.st+'/'+c.days+' — watered ✓ SLEEP to grow.');
    }
  }
  _farmSave();_farmRender();
}
function farmSleep(){
  var grew=0;
  FARM.plots.forEach(function(p){
    if(p.g===2){
      if(p.w){p.st++;grew++;}
      p.w=false;
    }
  });
  FARM.day++;FARM.energy=FARM.maxEnergy;
  _farmSave();_farmRender();
  if(navigator.vibrate)navigator.vibrate([20,40,20]);
  _farmMsg('☀️ DAY '+FARM.day+' — '+(grew?grew+' watered crop'+(grew>1?'s':'')+' grew overnight.':'nothing grew (water crops before sleeping!)'));
}
function farmShopToggle(){
  var el=document.getElementById('hFarmShop');if(!el)return;
  var open=el.style.display!=='none';
  el.style.display=open?'none':'block';
  if(!open)_farmShopRender();
}
function _farmShopRender(){
  var el=document.getElementById('hFarmShop');if(!el)return;
  var html='<div class="farm-shop-t">// DHARMA GENERAL STORE — SEEDS</div>';
  Object.keys(FARM_CROPS).forEach(function(k){
    var c=FARM_CROPS[k];
    html+='<div class="farm-shop-row"><span>'+c.icon+' '+c.name+' — 🪙'+c.seed+' · '+c.days+'d · sells 🪙'+c.sell+'</span>'
      +'<button class="farm-buy-btn" onclick="farmBuy(\''+k+'\')">BUY</button></div>';
  });
  var any=Object.keys(FARM.barn).some(function(k){return FARM.barn[k]>0;});
  if(any){
    html+='<div class="farm-shop-t">// YOUR HARVEST</div>';
    Object.keys(FARM.barn).forEach(function(k){
      if(!(FARM.barn[k]>0))return;
      var c=FARM_CROPS[k];
      html+='<div class="farm-shop-row"><span>'+c.icon+' '+c.name+' ×'+FARM.barn[k]+'</span>'
        +'<button class="farm-buy-btn" onclick="farmSell(\''+k+'\')">SELL ALL 🪙'+(FARM.barn[k]*c.sell)+'</button></div>';
    });
  }
  html+='<button class="farm-reset-btn" onclick="farmReset()">[ ABANDON FARM — START OVER ]</button>';
  el.innerHTML=html;
}
function farmBuy(k){
  var c=FARM_CROPS[k];
  if(FARM.coins<c.seed){_farmMsg('Not enough coins — sell some crops first.');return;}
  FARM.coins-=c.seed;FARM.seeds[k]=(FARM.seeds[k]||0)+1;
  _farmSave();_farmRender();_farmShopRender();
}
function farmSell(k){
  var c=FARM_CROPS[k];
  FARM.coins+=(FARM.barn[k]||0)*c.sell;FARM.barn[k]=0;
  if(navigator.vibrate)navigator.vibrate(30);
  _farmSave();_farmRender();_farmShopRender();
}
function farmReset(){
  if(!confirm('Abandon the farm and start over?'))return;
  FARM=_farmDefault();
  while(FARM.plots.length<FARM_SIZE*FARM_SIZE)FARM.plots.push({g:0});
  _farmSave();_hatchFarmShow();
}
```

### ✅ Phase 1 verification (do ALL of these in the browser)

1. Open Hatch → THE ORCHARD. HUD shows `DAY 1  🪙 20  ⚡ 50/50`; an 8×8 grid of dark tiles;
   a seed chip `🟣 ×4`.
2. Tap a tile → turns brown (tilled), energy drops by 2. Tap again → `🌱` appears, seed count
   drops. Tap again → tile turns blue-ish (watered).
3. Tap **SLEEP** → day becomes 2, energy back to 50. After 2 sleeps (watering each day) the
   turnip shows `🟣`. Tap it → harvested message.
4. Tap **SHOP** → buy rows for all 5 crops; a `SELL ALL` row for your turnip. Sell it → coins
   increase by 12.
5. Close the Hatch entirely, reopen, return to THE ORCHARD → **everything persisted** (day,
   coins, crops in ground).
6. Spend all energy → actions refuse with a "too tired" message instead of going negative.
7. `[ ABANDON FARM ]` in the shop asks for confirmation, then resets.
8. All other Hatch modes still work; console has no errors.

**Commit:** `Hatch: playable Orchard farming game (till/plant/water/sleep/harvest/shop)`

---

## Phase 2 — Polish & progression (small independent tasks, do any/all)

Each item below is standalone. Implement, verify, commit separately.

**2a. Growth peek** — tilled/planted tiles show progress on the tile itself: when a crop is
1 sleep from mature, render `🌾` instead of `🌿` (change `_farmTileIcon`: if
`p.st>=c.days-1` return `'🌾'` — but keep the mature check first).

**2b. Farm level & XP** — add `xp:0` to `_farmDefault()`. Each harvest: `FARM.xp += crop.days`.
Level = `Math.floor(FARM.xp/20)+1`, shown in the HUD as `LVL n`. Gate seeds in the shop:
melon needs LVL 2, pumpkin LVL 3, starfruit LVL 4 (render a `🔒 LVL n` disabled button
instead of BUY).

**2c. Random DHARMA event on sleep** — in `farmSleep()`, 15% chance (`Math.random()<0.15`)
of one random event, announced in the message line:
- `📦 DHARMA supply drop — +3 turnip seeds`
- `🐦 Hurley's birds ate a crop` (one random planted plot reverts to `{g:1}`)
- `🌧️ Rain overnight` (all planted plots count as watered for that night's growth, even
  unwatered ones — apply before the growth loop).

**2d. Chicken coop** — shop sells `🐔 Chicken — 🪙200` (max 4). Add `chickens:0` to state.
Each sleep, every chicken lays `🥚` into the barn (`sell: 25`). Show `🐔 ×n` in the HUD.

**2e. Sound-feel** — the app has no audio; keep using `navigator.vibrate` patterns for
harvest/levelup/events (already partially done). Do NOT add audio files.

---

## Phase 3 (optional, hard) — Walkable farmer

Only attempt when Phases 0–2 are done, verified, and committed. This replaces tap-to-act
with a character you steer.

- Replace the grid's tap handling with a `<canvas>` (fits `.hatch-inner`, ~400px), draw the
  same 8×8 tile state, plus a farmer `🧑‍🌾` drawn with `ctx.fillText`.
- On-screen d-pad (4 `.hatch-key` buttons) + keyboard arrows; an ACT button performs the
  contextual action on the tile the farmer faces (reuse `_farmTap(i)` logic unchanged —
  it already takes a tile index).
- Game loop via `requestAnimationFrame`. **Cleanup is mandatory:** store the RAF id in
  a var `_farmRaf`, and add to the existing `_clearHatchTimers()` function:
  `if(typeof _farmRaf!=='undefined'&&_farmRaf){cancelAnimationFrame(_farmRaf);_farmRaf=null;}`
  (`_clearHatchTimers` runs on every mode switch and on close, so this stops the loop.)
- Keep ALL state/save/shop/sleep logic from Phase 1 — only input/rendering changes.

---

## Known gotchas (re-read before editing)

1. **Two `<!-- SPA -->` comments** exist; anchor on `id="hPanelSpa"` instead.
2. **`hatchMode()`'s panel array is a hard registry** — forgetting `'Farm'` there makes the
   farm panel stay visible under other modes.
3. `localStorage.setItem` is overridden by the app's sync layer. Normal calls work; do not
   bypass it with exotic tricks.
4. Inline `onclick="..."` strings in this codebase escape inner quotes as `\'` when built in
   JS strings (see `farmBuy(\''+k+'\')` above) — copy the patterns exactly.
5. The Hatch has fixed scanline/vignette overlays at z-index 10001 and content at 10002 —
   put everything inside `.hatch-inner` (the plan's HTML already is) and z-index issues
   cannot happen.
6. Do not use `let`/`const`/arrow functions/template literals — the file is written in ES5
   style (`var`, `function`); stay consistent.
7. Test on a narrow viewport (~390px wide): the 8×8 grid must fit without horizontal scroll
   (`.hatch-inner` is max-width 400px; the CSS provided fits).

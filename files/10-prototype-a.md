# 10-prototype-a.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 1/6)

5098 righe totali

```html
<!doctype html>
<html lang="it">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>isa · Canvas Fusione</title>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Manrope:wght@500;700;800&display=swap">
<style>
  :root{
    --bg:#F5F3EE;
    --surface-strong:rgba(255,255,255,0.92);
    --panel-border:rgba(38,36,32,0.06);
    --ink:#262420;
    --muted:#847E74;
    --accent:#6C63FF;
    --accent-soft:rgba(108,99,255,0.16);
    --accent-soft-2:rgba(108,99,255,0.34);
    --stage-h:520px;
    padding-top:env(safe-area-inset-top,0px);
    padding-bottom:env(safe-area-inset-bottom,0px);
  }
  *{box-sizing:border-box}
  body{ margin:0; min-height:100%; background:var(--bg); color:var(--ink); font-family:'Manrope', system-ui, sans-serif; }
  .page{ max-width:1040px; margin:0 auto; padding:36px 24px 56px; display:flex; flex-direction:column; gap:18px; }
  .eyebrow{ font-weight:800; font-size:11px; letter-spacing:0.1em; text-transform:uppercase; color:var(--accent); }
  h1{ margin:0; font-size:24px; font-weight:800; }
  .rules{ margin:0; padding-left:18px; font-size:12.5px; color:var(--muted); line-height:1.75; max-width:70ch; }
  .rules b{ color:var(--ink); }
  .hint{
    font-size:12.5px; font-weight:700; color:var(--ink); background:var(--accent-soft);
    display:inline-flex; align-items:center; gap:8px; padding:7px 13px; border-radius:999px; width:fit-content;
  }

  .toolbar{ display:flex; align-items:center; gap:14px; flex-wrap:wrap; }
  .palette{
    display:flex; align-items:center; gap:10px; flex-wrap:wrap;
    padding:10px 12px; border-radius:18px; background:rgba(255,255,255,0.55);
    border:1px solid var(--panel-border);
  }
  .pal-item{
    display:flex; flex-direction:column; align-items:center; gap:5px;
    width:62px; cursor:grab; touch-action:none;
  }
  .pal-item:active{ cursor:grabbing; }
  .pal-chip{
    width:44px; height:44px; border-radius:13px; background:#E1DCF0; color:var(--accent);
    display:flex; align-items:center; justify-content:center;
    transition:transform .18s cubic-bezier(.34,1.56,.64,1);
  }
  .pal-item:hover .pal-chip{ transform:translateY(-2px) scale(1.05); }
  .pal-item.source .pal-chip{ background:var(--accent); color:#fff; border-radius:15px; }
  .pal-chip svg{ width:20px; height:20px; }
  .pal-label{ font-size:9.5px; font-weight:700; color:var(--muted); text-align:center; line-height:1.2; }

  .ghost{
    position:fixed; z-index:60; pointer-events:none; width:88px; height:88px; border-radius:22px;
    display:flex; align-items:center; justify-content:center;
    background:#E1DCF0; color:var(--accent); opacity:.9;
    box-shadow:0 18px 30px -14px rgba(38,36,32,0.45);
  }
  .ghost.source{ background:var(--accent); color:#fff; border-radius:26px; }
  .ghost svg{ width:26px; height:26px; }

  .seg{
    display:inline-flex; padding:3px; border-radius:999px;
    background:rgba(255,255,255,0.7); border:1px solid var(--panel-border);
  }
  .seg button{
    all:unset; cursor:pointer; font-family:inherit; font-weight:700; font-size:12px; color:var(--muted);
    padding:7px 14px; border-radius:999px; transition:background .18s, color .18s;
  }
  .seg button.on{ background:var(--accent); color:#fff; }

  .slot-ghost{
    position:absolute; width:88px; height:88px; border-radius:22px;
    border:2px dashed rgba(108,99,255,0.5); background:rgba(108,99,255,0.07);
    pointer-events:none; z-index:1; opacity:0; transition:opacity .15s, left .16s ease, top .16s ease;
  }
  .slot-ghost.on{ opacity:1; }

  button.tool-btn{
    all:unset; cursor:pointer; font-family:inherit; font-weight:700; font-size:12.5px; color:var(--ink);
    padding:9px 15px; border-radius:999px; background:rgba(255,255,255,0.7);
    border:1px solid var(--panel-border); display:inline-flex; align-items:center; gap:7px;
  }
  button.tool-btn:hover{ background:#fff; }
  button.tool-btn:disabled{ opacity:.4; cursor:not-allowed; }
  button.tool-btn svg{ width:14px; height:14px; }

  .del-btn{
    all:unset; position:absolute; top:-5px; left:-5px; width:19px; height:19px; border-radius:999px;
    background:#3A352E; color:#fff;
    display:flex; align-items:center; justify-content:center; cursor:pointer;
    opacity:0; pointer-events:none;
    transform:scale(.7);
    transition:opacity .16s, transform .16s cubic-bezier(.34,1.56,.64,1), background .16s;
    box-shadow:0 4px 10px -3px rgba(38,36,32,0.5); z-index:4;
  }
  .del-btn svg{ width:9px; height:9px; }
  .del-btn:hover{ background:#B23A3A; }
  .card:hover .del-btn{ opacity:1; pointer-events:auto; transform:scale(1); }
  .card.dragging .del-btn{ opacity:0 !important; pointer-events:none !important; }

  /* un solo gesto: si contrae e sparisce */
  @keyframes popOut{
    0%{ transform:scale(1); opacity:1 }
    100%{ transform:scale(.08); opacity:0 }
  }
  .card.popping{ animation:popOut .3s cubic-bezier(.5,0,.75,0) forwards; pointer-events:none; }

  /* stesso linguaggio del pannello di espansione */
  .confirm{
    position:fixed; z-index:70; width:246px; padding:22px;
    border-radius:26px; background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); box-shadow:0 40px 80px -30px rgba(38,36,32,0.4);
    display:none; flex-direction:column; gap:16px;
    transform:scale(.92); opacity:0;
    transition:transform .25s cubic-bezier(.34,1.56,.64,1), opacity .2s;
  }
  .confirm.open{ display:flex; transform:scale(1); opacity:1; }
  .confirm-title{ font-weight:800; font-size:15px; color:var(--ink); }
  .confirm-text{ font-size:11.5px; font-weight:600; line-height:1.5; color:var(--muted); margin-top:3px; }
  .confirm-actions{ display:flex; gap:8px; justify-content:flex-end; }
  .confirm-actions button{
    all:unset; cursor:pointer; font-family:inherit; font-size:12.5px; font-weight:700;
    padding:9px 15px; border-radius:999px;
  }
  .c-cancel{ color:var(--muted); background:rgba(38,36,32,0.05); }
  .c-cancel:hover{ color:var(--ink); }
  .c-ok{ background:#B23A3A; color:#fff; }
  .c-ok:hover{ background:#96302F; }

  .hit{ pointer-events:stroke; cursor:pointer; }
  .links{ position:absolute; inset:0; overflow:visible; }
  .world{ position:absolute; left:0; top:0; width:100%; height:100%; transform-origin:0 0; }
  .world.easing{ transition:transform .38s cubic-bezier(.32,.72,0,1); }

  .zoom-ctl{
    position:absolute; right:12px; bottom:12px; z-index:15; display:none; align-items:center; gap:2px;
    padding:4px; border-radius:999px; background:var(--surface-strong); backdrop-filter:blur(16px);
    border:1px solid var(--panel-border); box-shadow:0 10px 24px -14px rgba(38,36,32,0.4);
  }
  .zoom-ctl button{
    all:unset; cursor:pointer; min-width:28px; height:28px; padding:0 8px; box-sizing:border-box;
    display:flex; align-items:center; justify-content:center; border-radius:999px;
    font-family:inherit; font-size:12.5px; font-weight:700; color:var(--ink);
  }
  .zoom-ctl button:hover{ background:var(--accent-soft); color:var(--accent); }
  .zoom-ctl .fit{ color:var(--accent); }
  .stage.f-panZoom .zoom-ctl{ display:flex; }

  .minimap{
    position:absolute; left:12px; bottom:12px; width:168px; height:104px; z-index:15; display:none;
    border-radius:14px; background:var(--surface-strong); backdrop-filter:blur(16px);
    border:1px solid var(--panel-border); box-shadow:0 10px 24px -14px rgba(38,36,32,0.4);
    overflow:hidden; cursor:pointer;
  }
  .stage.f-panZoom.f-minimap .minimap{ display:block; }
  .mm-node{ position:absolute; border-radius:2px; background:#CFC9EF; }
  .mm-node.ds{ background:var(--accent); }
  .mm-view{ position:absolute; border:1.5px solid var(--accent); border-radius:4px; background:rgba(108,99,255,0.08); pointer-events:none; }

  .marquee{
    position:absolute; z-index:14; display:none; pointer-events:none;
    border:1.5px dashed var(--accent); background:rgba(108,99,255,0.08); border-radius:6px;
  }

  .port{
    position:absolute; width:11px; height:11px; border-radius:999px; box-sizing:border-box;
    background:#fff; border:2px solid var(--accent); z-index:5; cursor:crosshair;
    opacity:0; transform:scale(.5); transition:opacity .14s, transform .14s cubic-bezier(.34,1.56,.64,1);
    display:none;
  }
  .stage.f-portDrag .port{ display:block; }
  .card:hover .port{ opacity:1; transform:scale(1); }
  .card.dragging .port{ opacity:0 !important; }
  .port.p-t{ left:calc(50% - 5.5px); top:-5.5px; }
  .port.p-b{ left:calc(50% - 5.5px); bottom:-5.5px; }
  .port.p-l{ top:calc(50% - 5.5px); left:-5.5px; }
  .port.p-r{ top:calc(50% - 5.5px); right:-5.5px; }
  .port:hover{ background:var(--accent); }

  .state-dot{
    position:absolute; right:-4px; bottom:-4px; width:13px; height:13px; border-radius:999px;
    background:#E0A23B; border:2.5px solid #F7F5F1; z-index:4; display:none;
  }
  .stage.f-nodeStates .card.warn .state-dot{ display:block; }

  .feat-wrap{ position:relative; }
  .feat-panel{
    position:absolute; top:calc(100% + 8px); right:0; z-index:40; width:300px; display:none;
    padding:16px; border-radius:22px; background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); box-shadow:0 30px 60px -28px rgba(38,36,32,0.45);
    flex-direction:column; gap:4px;
  }
  .feat-panel.open{ display:flex; }
  .feat-title{ font-weight:800; font-size:13.5px; margin-bottom:2px; }
  .feat-sub{ font-size:11px; color:var(--muted); font-weight:600; line-height:1.45; margin-bottom:8px; }
  .feat-row{ display:flex; align-items:flex-start; gap:10px; padding:8px 6px; border-radius:12px; cursor:pointer; }
  .feat-row:hover{ background:var(--accent-soft); }
  .feat-row input{ appearance:none; flex-shrink:0; width:30px; height:18px; border-radius:999px; margin:1px 0 0;
    background:rgba(38,36,32,0.16); position:relative; cursor:pointer; transition:background .18s; }
  .feat-row input::after{ content:''; position:absolute; top:2px; left:2px; width:14px; height:14px; border-radius:999px;
    background:#fff; transition:transform .2s cubic-bezier(.34,1.56,.64,1); }
  .feat-row input:checked{ background:var(--accent); }
  .feat-row input:checked::after{ transform:translateX(12px); }
  .feat-name{ font-size:12.5px; font-weight:700; }
  .feat-desc{ font-size:10.5px; color:var(--muted); font-weight:600; line-height:1.4; margin-top:1px; }
  .links .nohit{ pointer-events:none; }

  /* spazio di lavoro: il canvas al centro, quattro approdi attorno per i pannelli */
  .workspace{
    display:grid; height:var(--stage-h); transition:height .3s cubic-bezier(.32,.72,0,1);
    grid-template-columns:auto minmax(0,1fr) auto; grid-template-rows:auto minmax(0,1fr) auto;
  }
  #dock-top{ grid-column:1 / 4; grid-row:1; display:flex; flex-direction:column; }
  #dock-bottom{ grid-column:1 / 4; grid-row:3; display:flex; flex-direction:column; }
  #dock-left{ grid-column:1; grid-row:2; display:flex; }
  #dock-right{ grid-column:3; grid-row:2; display:flex; }
  .workspace .stage{ grid-column:2; grid-row:2; height:auto; min-height:0; min-width:0; }

  /* un pannello cede spazio aprendosi; il margine verso il canvas fa parte della sua misura */
  .panel{ overflow:hidden; position:relative; flex:0 0 auto;
    transition:width .3s cubic-bezier(.32,.72,0,1), height .3s cubic-bezier(.32,.72,0,1); }
  .panel.side-left, .panel.side-right{ width:0; height:100%; }
  .panel.side-left.open, .panel.side-right.open{ width:calc(var(--pw) + 16px); }
  .panel.side-top, .panel.side-bottom{ height:0; width:100%; }
  .panel.side-top.open, .panel.side-bottom.open{ height:calc(var(--ph) + 16px); }
  .panel > div{ position:absolute; box-sizing:border-box; }
  .panel.side-left > div{ left:0; top:0; width:var(--pw); height:100%; }
  .panel.side-right > div{ right:0; top:0; width:var(--pw); height:100%; }
  .panel.side-top > div{ left:0; top:0; height:var(--ph); width:100%; }
  .panel.side-bottom > div{ left:0; bottom:0; height:var(--ph); width:100%; }

  .tb-inner{
    box-sizing:border-box; overflow-y:auto; overscroll-behavior:contain;
    background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); border-radius:26px; padding:18px 16px;
    display:flex; flex-direction:column; gap:6px;
  }
  .tb-inner > *{ flex:0 0 auto; }
  .tb-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:4px; }
  .tb-title{ font-weight:800; font-size:15px; }
  .tb-sec{ border-top:1px solid var(--panel-border); padding-top:8px; }
  .tb-sec:first-of-type{ border-top:0; }
  .tb-sec-head{
    all:unset; cursor:pointer; width:100%; box-sizing:border-box; display:flex; align-items:center; gap:7px; padding:4px 2px 8px;
  }
  .tb-sec-head .cond-chev svg{ width:11px; height:11px; }
  .tb-sec.open .tb-sec-head .cond-chev{ transform:rotate(90deg); color:var(--accent); }
  .tb-sec-name{ font-size:10px; font-weight:800; letter-spacing:.08em; text-transform:uppercase; color:var(--muted); }
  .tb-sec-body{
    display:grid; grid-template-columns:repeat(3, 1fr); gap:8px 4px;
    max-height:0; overflow:hidden; opacity:0;
    transition:max-height .3s cubic-bezier(.32,.72,0,1), opacity .2s, padding .3s;
  }
  .tb-sec.open .tb-sec-body{ max-height:640px; opacity:1; padding-bottom:10px; }
  .tb-sec-body .pal-item{ width:auto; }
  .tb-sec-body .pal-label{ font-size:9.5px; }

  .tb-upload{
    all:unset; cursor:pointer; grid-column:1 / -1; display:flex; align-items:center; justify-content:center; gap:7px;
    padding:9px 12px; border-radius:12px; border:1.5px dashed rgba(108,99,255,0.45);
    font-size:12px; font-weight:700; color:var(--accent); background:rgba(108,99,255,0.06);
  }
  .tb-upload:hover{ background:var(--accent-soft); }
  .tb-upload svg{ width:14px; height:14px; }
  .tb-empty{ grid-column:1 / -1; font-size:11px; color:var(--muted); font-weight:600; text-align:center; padding:2px 0 4px; }
  .lib-meta{ font-size:8.5px; color:var(--muted); font-weight:600; margin-top:-3px; text-align:center; }

  /* tacca: il pannello chiuso resta raggiungibile, e si trascina su un altro bordo */
  .notch{
    all:unset; cursor:grab; position:absolute; z-index:16; box-sizing:border-box;
    display:flex; align-items:center; justify-content:center;
    background:var(--surface-strong); color:var(--accent);
    border:1px solid var(--panel-border);
    box-shadow:0 6px 18px -10px rgba(38,36,32,0.45);
    transition:opacity .2s, transform .3s cubic-bezier(.32,.72,0,1), left .3s, top .3s;
    touch-action:none;
  }
  .notch svg{ width:14px; height:14px; }
  /* il pulsante per nascondere indica sempre il bordo verso cui il pannello rientra */
  #tbClose svg, #inspCollapse svg{ transition:transform .2s; }
  #toolbox.side-right #tbClose svg{ transform:rotate(180deg); }
  #toolbox.side-top #tbClose svg{ transform:rotate(90deg); }
  #toolbox.side-bottom #tbClose svg{ transform:rotate(-90deg); }
  #inspector.side-left #inspCollapse svg{ transform:rotate(180deg); }
  #inspector.side-top #inspCollapse svg{ transform:rotate(-90deg); }
  #inspector.side-bottom #inspCollapse svg{ transform:rotate(90deg); }
  .notch.n-left{ left:0; width:24px; height:66px; border-left:0; border-radius:0 14px 14px 0; }
  .notch.n-right{ right:0; width:24px; height:66px; border-right:0; border-radius:14px 0 0 14px; }
  .notch.n-top{ top:0; height:24px; width:66px; border-top:0; border-radius:0 0 14px 14px; }
  .notch.n-bottom{ bottom:0; height:24px; width:66px; border-bottom:0; border-radius:14px 14px 0 0; }
  .notch.hidden{ opacity:0; pointer-events:none; }
  .notch.dragging{ position:fixed; transition:none; cursor:grabbing; border-radius:14px; border:1px solid var(--accent);
    width:34px; height:34px; box-shadow:0 14px 30px -12px rgba(108,99,255,0.6); }
  .edge-hint{ position:absolute; z-index:15; background:var(--accent); opacity:0; pointer-events:none;
    border-radius:999px; transition:opacity .15s; }
  .edge-hint.on{ opacity:.55; }
  .edge-hint.e-left{ left:4px; top:12%; width:4px; height:76%; }
  .edge-hint.e-right{ right:4px; top:12%; width:4px; height:76%; }
  .edge-hint.e-top{ top:4px; left:12%; height:4px; width:76%; }
  .edge-hint.e-bottom{ bottom:4px; left:12%; height:4px; width:76%; }

  /* due pannelli sullo stesso bordo diventano schede di un unico pannello */
  .dock-tabs{ position:absolute; display:none; z-index:3; gap:4px; padding:4px; box-sizing:border-box;
    background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); border-radius:16px; }
  .panel.grouped > .dock-tabs{ display:flex; }
  .dock-tab{ all:unset; cursor:pointer; flex:1; display:flex; align-items:center; justify-content:center; gap:7px;
    padding:7px 10px; border-radius:12px; font-family:inherit; font-size:12px; font-weight:700; color:var(--muted);
    transition:background .18s, color .18s; }
  .dock-tab svg{ width:14px; height:14px; flex-shrink:0; }
  .dock-tab:hover{ color:var(--ink); }
  .dock-tab.on{ background:var(--accent); color:#fff; }
  /* sui lati verticali le linguette stanno in cima, su quelli orizzontali di fianco */
  .panel.grouped.side-left > .dock-tabs{ left:0; top:0; width:var(--pw); height:42px; }
  .panel.grouped.side-right > .dock-tabs{ right:0; top:0; width:var(--pw); height:42px; }
  .panel.grouped.side-left > div, .panel.grouped.side-right > div{ top:50px; height:calc(100% - 50px); }
  .panel.grouped.horiz > .dock-tabs{ left:0; width:48px; height:var(--ph); flex-direction:column; }
  .panel.grouped.side-top > .dock-tabs{ top:0; }
  .panel.grouped.side-bottom > .dock-tabs{ bottom:0; }
  .panel.grouped.horiz > div{ left:56px; width:calc(100% - 56px); }
  .panel.grouped.horiz .dock-tab span{ display:none; }
  .panel.instant{ transition:none !important; }
  @keyframes tabIn{ from{ opacity:0; transform:translateY(6px); } to{ opacity:1; transform:none; } }
  .panel.tab-in > div{ animation:tabIn .26s cubic-bezier(.32,.72,0,1); }

  /* orientamento orizzontale della cassetta: le sezioni scorrono in orizzontale */
  .panel.horiz .tb-inner{ flex-direction:row; overflow-x:auto; overflow-y:hidden; gap:14px; align-items:stretch; }
  .panel.horiz .tb-head{ flex-direction:column; align-items:flex-start; justify-content:flex-start; gap:8px; margin:0; }
  .panel.horiz .tb-sec{ border-top:0; border-left:1px solid var(--panel-border); padding:0 0 0 12px; }
  .panel.horiz .tb-sec-head{ pointer-events:none; }
  .panel.horiz .tb-sec-head .cond-chev{ display:none; }
  .panel.horiz .tb-sec-body{ max-height:none; opacity:1; padding:0;
    grid-template-columns:none; grid-template-rows:repeat(2, auto); grid-auto-flow:column; grid-auto-columns:66px; }
  .panel.horiz .tb-upload{ grid-column:auto; grid-row:span 2; flex-direction:column; width:66px; box-sizing:border-box; padding:8px 4px; text-align:center; }

  /* orientamento orizzontale dell'inspector: il contenuto si distribuisce in colonne */
  .panel.horiz .insp-inner{ display:block; column-width:250px; column-gap:22px; column-fill:auto;
    overflow-x:auto; overflow-y:hidden; }
  .panel.horiz .insp-inner > *{ break-inside:avoid; margin-bottom:14px; }

  /* in orizzontale l'inspector diventa master-dettaglio: elenco a sinistra, configurazione a destra */
  .panel.horiz .insp-inner.md{ display:flex; flex-direction:row; gap:0; column-width:auto; overflow:hidden; padding:16px 0 16px 18px; }
  .panel.horiz .insp-inner.md > *{ margin-bottom:0; }
  .insp-master{ flex:0 0 300px; overflow-y:auto; overscroll-behavior:contain; display:flex; flex-direction:column; gap:10px; padding-right:16px; }
  .insp-master > *{ flex:0 0 auto; }
  .insp-master .cond.open{ border-color:var(--accent); background:var(--accent-soft); }
  .insp-master .cond .cond-chev{ transform:none !important; }
  .insp-master .cond.open .cond-chev{ color:var(--accent); }
  .insp-detail{ flex:1; min-width:0; overflow-y:auto; overscroll-behavior:contain;
    border-left:1px solid var(--panel-border); padding:0 18px; }
  .insp-detail.menu{ overflow:visible; }
  .md-title{ font-size:10px; font-weight:800; letter-spacing:.08em; text-transform:uppercase; color:var(--accent); margin:2px 0 2px; }
  .md-sum{ font-size:14px; font-weight:800; color:var(--ink); margin-bottom:14px; }
  .md-sum.empty{ color:var(--muted); font-style:italic; font-weight:700; }
  /* piu specifiche della regola generale che impedisce ai figli dell'inspector di allargarsi */
  .insp-inner.md > .insp-general{ flex:0 0 240px; overflow-y:auto; overscroll-behavior:contain;
    display:flex; flex-direction:column; gap:12px; padding-right:16px; }
  .insp-inner.md > .insp-general > *{ flex:0 0 auto; }
  .insp-inner.md > .insp-master{ flex:0 0 310px; border-left:1px solid var(--panel-border); padding-left:16px; }
  .md-col{ font-size:10px; font-weight:800; letter-spacing:.08em; text-transform:uppercase; color:var(--muted);
    padding-bottom:6px; border-bottom:1px solid var(--panel-border); margin-bottom:2px; flex:0 0 auto; }
  .insp-master .cond-area{ display:flex; flex-direction:column; gap:10px; }
  .insp-master .cond-area + .cond-area{ border-top:1px solid var(--panel-border); padding-top:12px; }
  /* in verticale la struttura resta sotto le impostazioni, separata da una linea */
  .insp-inner:not(.md) .cond-area{ display:flex; flex-direction:column; gap:16px;
    border-top:1px solid var(--panel-border); padding-top:14px; }
  .insp-inner.md > .insp-detail{ flex:1 1 auto; }
  .insp-detail .cond-body{ display:none; max-height:none; opacity:1; padding:0; overflow:visible; }
  .insp-detail .cond-body.shown{ display:grid; grid-template-columns:repeat(auto-fill, minmax(210px, 1fr)); gap:14px 18px;
    align-items:start; animation:tabIn .22s cubic-bezier(.32,.72,0,1); }
  .insp-detail .cond-body .jk-eq{ display:none; }
  .insp-rail{
    all:unset; cursor:pointer; width:34px; height:34px; border-radius:12px;
    display:flex; align-items:center; justify-content:center;
    background:var(--accent-soft); color:var(--accent);
  }
  .insp-rail svg{ width:15px; height:15px; }
  .insp-rail-label{
    writing-mode:vertical-rl; font-size:10.5px; font-weight:800; letter-spacing:.08em;
    text-transform:uppercase; color:var(--muted);
  }
  .insp-actions{ display:flex; gap:6px; flex-shrink:0; }
  .insp-inner{
    box-sizing:border-box;
    background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); border-radius:26px; padding:22px;
    display:flex; flex-direction:column; gap:16px; overflow-y:auto; overscroll-behavior:contain; scroll-behavior:smooth;
  }
  /* in una colonna ad altezza fissa i figli si comprimerebbero: qui devono traboccare e scorrere */
  .insp-inner > *{ flex:0 0 auto; }
  .insp-head{ display:flex; align-items:flex-start; justify-content:space-between; gap:10px; }
  .insp-kind{ font-size:10px; font-weight:800; letter-spacing:.09em; text-transform:uppercase; color:var(--accent); }
  .insp-name{
    font-weight:800; font-size:15px; color:var(--ink); margin-top:3px;
    border-bottom:1px dashed var(--accent); outline:none; cursor:text;
  }
  .insp-empty{ font-size:12px; color:var(--muted); line-height:1.6; font-weight:600; }

  .step-list{ display:flex; flex-direction:column; gap:4px; }
  .insp-step{
    display:flex; align-items:center; gap:8px; padding:7px 9px; border-radius:12px;
    background:rgba(38,36,32,0.04); cursor:pointer; touch-action:none; will-change:transform;
  }
  .insp-step.on{ background:var(--accent-soft); }
  .insp-step.drag-row{ background:var(--accent-soft); box-shadow:0 12px 22px -12px rgba(38,36,32,0.4); z-index:5; }
  .insp-step .grip{ width:16px; color:var(--muted); display:flex; align-items:center; justify-content:center; cursor:grab; flex-shrink:0; }
  .insp-step .grip svg{ width:12px; height:12px; }
  .insp-step .st-ico{
    width:26px; height:26px; border-radius:9px; background:#fff; color:var(--accent);
    display:flex; align-items:center; justify-content:center; flex-shrink:0;
  }
  .insp-step .st-ico svg{ width:14px; height:14px; }
  .insp-step .st-name{ font-size:11.5px; font-weight:700; color:var(--ink); min-width:0; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  @keyframes stepIn{
    0%{ opacity:0; transform:scale(.7) }
    62%{ opacity:1; transform:scale(1.06) }
    100%{ transform:scale(1) }
  }
  .insp-step.in{ animation:stepIn .38s cubic-bezier(.34,1.56,.64,1) both; }
  .seq-head{
    all:unset; cursor:pointer; display:flex; align-items:center; gap:7px; padding:2px 0 6px;
  }
  .seq-chev{ display:flex; color:var(--muted); transition:transform .2s; }
  .seq-chev svg{ width:11px; height:11px; }
  .seq.open .seq-chev{ transform:rotate(90deg); color:var(--accent); }
  .seq-label{ font-size:10px; font-weight:700; letter-spacing:.06em; text-transform:uppercase; color:var(--muted); }
  .seq-count{ font-size:10px; font-weight:700; color:var(--accent); background:var(--accent-soft); padding:2px 7px; border-radius:999px; }
  .seq .step-list{
    max-height:0; opacity:0; overflow:hidden;
    transition:max-height .32s cubic-bezier(.32,.72,0,1), opacity .22s;
  }
  .seq.open .step-list{ max-height:520px; opacity:1; }

  .insp-step .st-sp{ flex:1; }
  .st-del{
    all:unset; cursor:pointer; width:19px; height:19px; border-radius:999px; flex-shrink:0;
    display:flex; align-items:center; justify-content:center; color:var(--muted);
    background:rgba(38,36,32,0.06); opacity:0; transition:opacity .15s, background .15s, color .15s;
  }
  .st-del svg{ width:9px; height:9px; }
  .insp-step:hover .st-del{ opacity:1; }
  .st-del:hover{ background:#B23A3A; color:#fff; }

  .insp-step .st-n{ font-size:10px; font-weight:800; color:var(--muted); flex-shrink:0; }

  .field{ display:flex; flex-direction:column; gap:5px; }
  .field label{ font-size:10px; font-weight:700; letter-spacing:.06em; text-transform:uppercase; color:var(--muted); }
  /* menu a discesa con lo stesso linguaggio del pannello di espansione */
  .sel{ position:relative; }
  .sel-btn{
    all:unset; cursor:pointer; box-sizing:border-box; width:100%;
    display:flex; align-items:center; justify-content:space-between; gap:8px;
    font-size:12.5px; font-weight:600; color:var(--ink);
    background:rgba(255,255,255,0.75); border:1px solid var(--panel-border);
    border-radius:11px; padding:9px 11px;
    transition:border-color .15s, background .15s;
  }
  .sel-btn:hover{ background:#fff; }
  .sel.open .sel-btn{ border-color:var(--accent); }
  .sel-btn .chev{ display:flex; color:var(--muted); transition:transform .18s; flex-shrink:0; }
  .sel-btn .chev svg{ width:12px; height:12px; }
  .sel.open .sel-btn .chev{ transform:rotate(180deg); color:var(--accent); }
  .sel-val{ overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  .sel-menu{
    position:absolute; top:calc(100% + 6px); left:0; right:0; z-index:12;
    background:var(--surface-strong); backdrop-filter:blur(20px);
    border:1px solid var(--panel-border); border-radius:14px;
    box-shadow:0 20px 40px -20px rgba(38,36,32,0.35);
    padding:6px; display:none; flex-direction:column; gap:2px;
    max-height:190px; overflow-y:auto;
  }
  .sel.open .sel-menu{ display:flex; }
  .sel-menu button{
    all:unset; cursor:pointer; font-family:inherit; font-size:12.5px; font-weight:600; color:var(--ink);
    padding:8px 10px; border-radius:9px;
  }
  .sel-menu button:hover{ background:var(--accent-soft); color:var(--accent); }
  .sel-menu button.on{ background:var(--accent-soft); color:var(--accent); }
  .sel-sep{ height:1px; background:var(--panel-border); margin:5px 3px; }
  .sel-manual-label{ font-size:9.5px; font-weight:800; letter-spacing:.06em; text-transform:uppercase; color:var(--muted); padding:2px 10px 4px; }
  .sel-manual{
    font-family:inherit; font-size:12.5px; font-weight:600; color:var(--ink);
    background:rgba(255,255,255,0.8); border:1px solid var(--panel-border);
    border-radius:9px; padding:7px 9px; outline:none; margin:0 3px 3px; box-sizing:border-box; width:calc(100% - 6px);
  }
  .sel-manual:focus{ border-color:var(--accent); }
  .sel-val.free{ font-style:italic; }

  .field input, .field select{
    font-family:inherit; font-size:12.5px; font-weight:600; color:var(--ink);
    background:rgba(255,255,255,0.75); border:1px solid var(--panel-border);
    border-radius:11px; padding:9px 11px; outline:none; width:100%; box-sizing:border-box;
  }
  .field input:focus, .field select:focus{ border-color:var(--accent); }
  .insp-lock{
    display:flex; gap:10px; align-items:flex-start;
    font-size:11.5px; font-weight:600; color:var(--muted); line-height:1.55;
    background:rgba(38,36,32,0.05); border-radius:14px; padding:13px 14px;
  }
  .insp-lock svg{ width:15px; height:15px; flex-shrink:0; margin-top:1px; color:var(--muted); }
  .insp-note{
    font-size:11px; font-weight:600; color:var(--muted); line-height:1.5;
    background:var(--accent-soft); border-radius:12px; padding:10px 12px;
  }
  .card.selected .icon-wrap{ box-shadow:0 0 0 3px var(--accent); }

  .cond{
    border:1px solid var(--panel-border); border-radius:16px;
    background:rgba(255,255,255,0.45); overflow:hidden;
  }
  .cond.open{ background:rgba(255,255,255,0.72); border-color:rgba(108,99,255,0.35); }
  .cond-head{ display:flex; align-items:center; gap:6px; padding:9px 10px; }
  .cond-toggle{
    all:unset; cursor:pointer; flex:1; min-width:0; display:flex; align-items:center; gap:8px;
  }
  .cond-chev{ color:var(--muted); display:flex; transition:transform .2s; flex-shrink:0; }
  .cond-chev svg{ width:12px; height:12px; }
  .cond.open .cond-chev{ transform:rotate(90deg); color:var(--accent); }
  .cond-sum{ min-width:0; display:flex; flex-direction:column; text-align:left; }
  .cond-n{ display:block; font-size:9.5px; font-weight:800; letter-spacing:.07em; text-transform:uppercase; color:var(--accent); }
  .cond-txt{
    display:block; font-size:11.5px; font-weight:600; color:var(--ink); margin-top:1px;
    white-space:nowrap; overflow:hidden; text-overflow:ellipsis; max-width:186px;
  }
  .cond-txt.empty{ color:var(--muted); font-style:italic; }
  .cond-body{
    display:flex; flex-direction:column; gap:10px;
    max-height:0; padding:0 12px; overflow:hidden; opacity:0;
    transition:max-height .32s cubic-bezier(.32,.72,0,1), padding .32s cubic-bezier(.32,.72,0,1), opacity .22s;
  }
  .cond.open .cond-body{ max-height:660px; padding:2px 12px 12px; opacity:1; }
  /* con un menu aperto il ritaglio va sospeso, altrimenti il menu resta invisibile */
  .cond-body.menu, .cond.menu{ overflow:visible; }
  .sel.up .sel-menu{ top:auto; bottom:calc(100% + 6px); }
  .cond-del{
    all:unset; cursor:pointer; width:22px; height:22px; border-radius:999px; color:var(--muted);
    display:flex; align-items:center; justify-content:center; background:rgba(38,36,32,0.05);
  }
  .cond-del:hover{ background:#B23A3A; color:#fff; }
  .cond-del svg{ width:10px; height:10px; }
  .col-type{ font-size:9.5px; font-weight:700; color:var(--muted); text-transform:uppercase; letter-spacing:.05em; }

  .mode-seg{ display:inline-flex; padding:2px; border-radius:999px; background:rgba(38,36,32,0.06); }
  .mode-seg button{
    all:unset; cursor:pointer; font-size:10.5px; font-weight:700; color:var(--muted);
    padding:5px 11px; border-radius:999px;
  }
  .mode-seg button.on{ background:#fff; color:var(--accent); }

  .checklist{
    max-height:150px; overflow-y:auto; display:flex; flex-direction:column; gap:2px;
    border:1px solid var(--panel-border); border-radius:11px; padding:7px; background:rgba(255,255,255,0.6);
  }
  .checkrow{ display:flex; align-items:center; gap:8px; font-size:12px; font-weight:600; padding:4px 5px; border-radius:8px; cursor:pointer; }
  .checkrow:hover{ background:var(--accent-soft); }
  .checkrow input{ width:14px; height:14px; accent-color:var(--accent); margin:0; }
  /* un valore si mostra come e nei dati: niente maiuscolo ereditato dalle etichette dei campi */
  .field label.checkrow{ text-transform:none; letter-spacing:0; font-size:12px; font-weight:600; color:var(--ink); }
  .sel-count{ font-size:10.5px; font-weight:700; color:var(--muted); }

  /* connettore logico tra due condizioni */
  .vpick{ display:flex; flex-direction:column; gap:7px; }
  .vp-chips{ display:flex; flex-wrap:wrap; gap:5px; }
  .vp-chip{ display:inline-flex; align-items:center; gap:3px; padding:3px 4px 3px 9px; border-radius:999px;
    background:var(--accent-soft); color:var(--accent); font-size:11.5px; font-weight:700; }
  .vp-chip.free{ font-style:italic; }
  .vp-chip button{ all:unset; cursor:pointer; width:16px; height:16px; border-radius:999px; display:flex;
    align-items:center; justify-content:center; font-size:12px; line-height:1; }
  .vp-chip button:hover{ background:var(--accent); color:#fff; }
  .vp-none{ font-size:11px; color:var(--muted); font-style:italic; font-weight:600; }
  .vp-list{ max-height:170px; overflow-y:auto; overscroll-behavior:contain; border:1px solid var(--panel-border);
    border-radius:11px; padding:6px; background:rgba(255,255,255,0.6); display:flex; flex-direction:column; gap:1px; }
  .vp-add button{ all:unset; cursor:pointer; display:block; font-size:11.5px; font-weight:700; color:var(--accent);
    padding:7px 10px; border-radius:10px; background:var(--accent-soft); }
  .vp-foot{ display:flex; align-items:center; justify-content:space-between; gap:6px; }
  .vp-btn{ all:unset; cursor:pointer; font-size:10.5px; font-weight:700; color:var(--muted); padding:3px 7px; border-radius:7px; }
  .vp-btn:hover{ color:var(--accent); background:var(--accent-soft); }
  .conn-row{ display:flex; align-items:center; gap:8px; margin:-4px 0; }
  .conn-line{ flex:1; height:1px; background:var(--panel-border); }
  .conn-row .sel{ flex:0 0 auto; }
  .conn-row .sel-btn{ width:auto; padding:5px 11px; border-radius:999px; font-size:11px; font-weight:800;
    letter-spacing:.04em; color:var(--accent); background:var(--accent-soft); border-color:transparent; gap:6px; }
  .conn-row .sel-menu{ left:50%; right:auto; transform:translateX(-50%); min-width:250px; }
  .sel-menu.floating{ position:fixed; display:flex; z-index:300; transform:none; right:auto; }
  .cgroup{ border:1.5px dashed rgba(108,99,255,0.45); border-radius:18px; padding:10px;
    display:flex; flex-direction:column; gap:12px; background:rgba(108,99,255,0.04); }
  .cgroup-head{ display:flex; align-items:center; justify-content:space-between; padding:0 2px; }
  .cgroup-head span{ font-size:10px; font-weight:800; letter-spacing:.08em; text-transform:uppercase; color:var(--accent); }
  .cgroup-head button{ all:unset; cursor:pointer; font-size:11px; font-weight:700; color:var(--muted); padding:4px 8px; border-radius:8px; }
  .cgroup-head button:hover{ background:var(--accent-soft); color:var(--accent); }
  .add-in-group{ all:unset; cursor:pointer; text-align:center; font-size:11.5px; font-weight:700; color:var(--accent);
    padding:7px; border-radius:11px; background:rgba(255,255,255,0.6); }
  .add-in-group:hover{ background:var(--accent-soft); }
  .grp-btn{ all:unset; cursor:pointer; font-family:ui-monospace, SFMono-Regular, Menlo, monospace; font-size:11px; font-weight:800;
    color:var(--muted); padding:4px 8px; border-radius:999px; background:rgba(38,36,32,0.05); flex:0 0 auto; }
  .grp-btn:hover{ color:var(--accent); background:var(--accent-soft); }
  .logic-prev{ font-family:ui-monospace, SFMono-Regular, Menlo, monospace; font-size:11px; line-height:1.55; word-break:break-word; }
  .logic-prev b{ font-family:'Manrope', system-ui, sans-serif; display:block; margin-bottom:3px; }
  .logic-prev .lp-hint{ font-family:'Manrope', system-ui, sans-serif; display:block; margin-top:4px; opacity:.8; }
  .insp-warn{ font-size:11px; font-weight:600; line-height:1.5; color:#8A5A12; background:#FBEFD8; border-radius:12px; padding:10px 12px; }

  .jk{
    border:1px solid var(--panel-border); border-radius:14px; padding:11px;
    display:flex; flex-direction:column; gap:9px; background:rgba(255,255,255,0.45);
  }
  .jk-head{ display:flex; align-items:center; justify-content:space-between; }
  .jk-n{ font-size:9.5px; font-weight:800; letter-spacing:.07em; text-transform:uppercase; color:var(--accent); }
  .jk-eq{ text-align:center; font-size:11px; font-weight:800; color:var(--muted); margin:-3px 0; }

  .add-cond{
    all:unset; cursor:pointer; font-size:12px; font-weight:700; color:var(--accent);
    padding:9px 13px; border-radius:11px; background:var(--accent-soft); text-align:center;
  }
  .add-cond:hover{ background:rgba(108,99,255,0.24); }
  .logic-row{ display:flex; align-items:center; gap:8px; }
  .logic-row .field{ flex:1; }

  .stage{
    position:relative; height:var(--stage-h); user-select:none; touch-action:none;
    border-radius:20px; background:rgba(255,255,255,0.32); overflow:hidden;
  }

  .card{ position:absolute; display:flex; flex-direction:column; align-items:center; gap:8px; width:88px; cursor:grab; touch-action:none; z-index:2; }
  .card.dragging{ cursor:grabbing; z-index:20; }

  .icon-wrap{
    position:relative; width:88px; height:88px; border-radius:22px;
    background:#E1DCF0; color:var(--accent);
    display:grid; place-content:center; justify-items:center; align-items:center;
    gap:6px; padding:7px;
    transition:box-shadow .2s, transform .28s cubic-bezier(.34,1.56,.64,1), background .2s;
  }
  .card.dataset .icon-wrap{ background:var(--accent); color:#fff; border-radius:26px; }
  .card.output .icon-wrap{ background:var(--accent); color:#fff; border-radius:26px; opacity:.92; }

  .card.dragging .icon-wrap{ box-shadow:0 18px 30px -14px rgba(38,36,32,0.4); transform:scale(1.05); }
  .card.drop-target .icon-wrap{ box-shadow:0 0 0 3px var(--accent-soft-2); background:#D2CBEE; transform:scale(1.09); }
  .card.drop-target .icon-wrap svg{ transform:scale(.88); }
  .card.link-target .icon-wrap{ box-shadow:0 0 0 3px var(--accent); transform:scale(1.04); }

  .icon-wrap svg{ display:block; flex-shrink:0;
    transition:width .26s cubic-bezier(.34,1.56,.64,1), height .26s cubic-bezier(.34,1.56,.64,1), transform .24s ease; }
  .icon-wrap.count-1{ grid-template-columns:repeat(1,auto); }
  .icon-wrap.count-2{ grid-template-columns:repeat(2,auto); }
  .icon-wrap.count-3, .icon-wrap.count-6, .icon-wrap.count-many{ grid-template-columns:repeat(3,auto); }
  .icon-wrap.count-1 svg{ width:26px; height:26px; }
  .icon-wrap.count-2 svg{ width:20px; height:20px; }
  .icon-wrap.count-3 svg{ width:17px; height:17px; }
  .icon-wrap.count-6 svg{ width:15px; height:15px; }
  .icon-wrap.count-many svg{ width:12px; height:12px; }

  /* output di un join: tante fette quante sono le tabelle attese */
  .card.dataset .icon-wrap.split,
  .card.output .icon-wrap.split{
    padding:0; display:flex; overflow:hidden; background:#EFEDF7; gap:0;
  }
  .icon-wrap.split .half{
    flex:1 1 0; min-width:0; height:100%; padding:2px; box-sizing:border-box;
    display:flex; align-items:center; justify-content:center;
  }
  .icon-wrap.split .half.full{ background:var(--accent); color:#fff; }
  .icon-wrap.split .half.empty{ background:#E6E3F5; color:#8F88C7; }
  .icon-wrap.split .half + .half{ border-left:1.5px dashed rgba(108,99,255,0.45); }
  /* la fetta detta la misura: nessun numero di tabelle puo sfondare l'icona */
  .icon-wrap.split .half svg{ width:100%; height:auto; max-width:22px; max-height:100%; }
  .icon-wrap.split .half.empty svg{ opacity:.85; animation:waiting 1.9s ease-in-out infinite; }
  @keyframes waiting{ 0%,100%{ opacity:.45 } 50%{ opacity:.95 } }
  .card.partial .label{ color:var(--muted); font-style:italic; }

  .icon-wrap.fusing{ animation:squash .42s cubic-bezier(.34,1.56,.64,1); }
  @keyframes squash{ 0%{transform:scale(1)} 28%{transform:scale(.9,1.08)} 55%{transform:scale(1.1,.94)} 100%{transform:scale(1)} }
  .icon-wrap.recoil{ animation:recoil .45s cubic-bezier(.34,1.56,.64,1); }
  @keyframes recoil{ 0%{transform:scale(1)} 25%{transform:scale(1.1,.88)} 50%{transform:scale(.92,1.06)} 100%{transform:scale(1)} }

  svg.arriving{ animation:arrive .4s cubic-bezier(.34,1.56,.64,1) both; }
  @keyframes arrive{ from{opacity:0;transform:scale(.35)} 60%{opacity:1;transform:scale(1.12)} to{opacity:1;transform:scale(1)} }

  .label{ font-size:10.5px; font-weight:700; text-align:center; line-height:1.25; color:var(--ink);
    max-width:96px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; padding:0 2px; }
  .card.dataset .label, .card.output .label{ color:var(--accent); }

  .expand-btn{
    all:unset; position:absolute; top:-7px; right:-7px; width:24px; height:24px; border-radius:999px;
    background:var(--accent); color:#fff; display:flex; align-items:center; justify-content:center;
    cursor:pointer; opacity:0; pointer-events:none; transform:scale(.7);
    transition:opacity .16s, transform .16s cubic-bezier(.34,1.56,.64,1);
    box-shadow:0 6px 14px -4px rgba(108,99,255,0.65); z-index:3;
  }
  .expand-btn svg{ width:12px; height:12px; }
  .card.combined:hover .expand-btn{ opacity:1; pointer-events:auto; transform:scale(1); }
  .card.dragging .expand-btn, .card.drop-target .expand-btn{ opacity:0 !important; pointer-events:none !important; }

  .burst{ position:absolute; width:14px; height:14px; border-radius:999px;
    background:radial-gradient(circle, var(--accent-soft-2) 0%, transparent 70%);
    transform:translate(-50%,-50%); pointer-events:none; animation:burst .55s ease-out forwards; }
  @keyframes burst{ from{transform:translate(-50%,-50%) scale(.4);opacity:.8} to{transform:translate(-50%,-50%) scale(5.5);opacity:0} }

  button.reset-btn{ all:unset; cursor:pointer; font-family:inherit; font-weight:700; font-size:12px; color:var(--muted); width:fit-content; }
  button.reset-btn:hover{ color:var(--ink); }

  .overlay{
    position:fixed; inset:0; background:rgba(38,36,32,0.32); backdrop-filter:blur(4px);
    display:flex; align-items:center; justify-content:center; padding:24px;
    opacity:0; pointer-events:none; transition:opacity .25s; z-index:30;
  }
  .overlay.open{ opacity:1; pointer-events:auto; }
  .expand-card{
    width:100%; max-width:420px; background:var(--surface-strong); backdrop-filter:blur(24px);
    border:1px solid var(--panel-border); border-radius:26px; padding:24px;
    box-shadow:0 40px 80px -30px rgba(38,36,32,0.4);
    transform:scale(.92); transition:transform .25s cubic-bezier(.34,1.56,.64,1);
  }
  .overlay.open .expand-card{ transform:scale(1); }
  .expand-head{ display:flex; align-items:flex-start; justify-content:space-between; margin-bottom:6px; gap:10px; }
  .expand-title{ font-weight:800; font-size:15px; outline:none; border-bottom:1px dashed var(--accent); cursor:text; }
  .expand-sub{ font-size:11.5px; color:var(--muted); margin-top:3px; }
  .reorder-note{ font-size:11px; color:var(--muted); margin:0 0 12px; font-weight:600; }
  button.close-btn{ all:unset; cursor:pointer; color:var(--muted); width:26px; height:26px; border-radius:999px;
    display:flex; align-items:center; justify-content:center; background:rgba(38,36,32,0.05); flex-shrink:0; }

  .steps{ display:flex; flex-direction:column; gap:6px; position:relative; }
  .step{
    display:flex; align-items:center; gap:10px; position:relative; padding:8px 6px;
    border-radius:12px; background:transparent; touch-action:none; will-change:transform;
  }
  .step.drag-row{ background:var(--accent-soft); box-shadow:0 14px 26px -14px rgba(38,36,32,0.45); z-index:5; }
  .step.menu-open{ z-index:40; }   /* il menu aperto non deve finire sotto le icone successive */
  .grip{ width:20px; display:flex; align-items:center; justify-content:center; color:var(--muted); cursor:grab; flex-shrink:0; }
  .grip svg{ width:14px; height:14px; }
  .step.drag-row .grip{ cursor:grabbing; }
  .step-num{ width:34px; height:34px; border-radius:11px; background:var(--accent-soft); color:var(--accent);
    display:flex; align-items:center; justify-content:center; flex-shrink:0; }
  .step-num svg{ width:16px; height:16px; }
  .step-info{ flex:1; min-width:0; }
  .step-name{ font-weight:700; font-size:13px; }
  .step-order{ font-size:10.5px; color:var(--muted); font-weight:600; }

  .step-settings{ position:relative; flex-shrink:0; }
  button.settings-btn{ all:unset; cursor:pointer; width:30px; height:30px; border-radius:10px; color:var(--muted);
    display:flex; align-items:center; justify-content:center; background:rgba(38,36,32,0.05);
    transition:background .15s, color .15s; }
  button.settings-btn:hover, button.settings-btn.active{ background:var(--accent-soft); color:var(--accent); }
  button.settings-btn svg{ width:15px; height:15px; }

  .dropdown{
    position:absolute; top:calc(100% + 6px); right:0; min-width:180px;
    background:var(--surface-strong); backdrop-filter:blur(20px); border:1px solid var(--panel-border);
    border-radius:14px; box-shadow:0 20px 40px -20px rgba(38,36,32,0.35);
    padding:6px; display:none; flex-direction:column; gap:2px; z-index:8;
  }
  .dropdown.open{ display:flex; }
  .dropdown button{ all:unset; cursor:pointer; font-family:inherit; font-size:12.5px; font-weight:600; color:var(--ink);
    padding:8px 10px; border-radius:9px; text-align:left; }
  .dropdown button:hover{ background:var(--accent-soft); color:var(--accent); }
  .dropdown button.danger:hover{ background:rgba(217,108,108,0.15); color:#B23A3A; }
</style>
</head>
<body>
<div class="page">
  <div class="eyebrow">isa · Prototipo interazione</div>
  <h1>Canvas: sorgenti, lavorazioni, output</h1>
  <ul class="rules">
    <li><b>Il dataset è il punto di partenza</b>: sorgente, non lavorazione — non entra mai in un box.</li>
    <li><b>Le lavorazioni si fondono</b> tra loro trascinandole una sull'altra.</li>
    <li><b>Collegando un dataset a un box</b> viene generato automaticamente il dataset di output in uscita.</li>
    <li><b>L'output è a sua volta una sorgente</b>: trascinalo su un altro box per concatenare le lavorazioni.</li>
    <li><b>Il Join vuole due tabelle</b>: la prima collegata è la left table, e finché manca la seconda l'output resta a metà.</li>
    <li><b>Cassetta degli strumenti</b> a sinistra: carica un CSV nella sezione Dataset e trascina dataset e operazioni sul canvas, su un box o su un cavo. Si nasconde in una tacca sul bordo.</li>
    <li><b>Per rimuovere</b>: passa sopra un nodo e premi la ×, oppure clicca un cavo per scollegarlo. Cmd/Ctrl+Z annulla, Cmd/Ctrl+Maiusc+Z ripristina.</li>
    <li><b>Libero o Organizzato</b>: nel secondo i nodi occupano postazioni fisse e si spostano solo scambiandosi di posto.</li>
    <li><b>Novità</b>: porte per collegare, inserimento su cavo, stati, pan e zoom, selezione multipla, tastiera. Ognuna si spegne dal pannello <b>Funzionalità</b>.</li>
    <li><b>Clicca un nodo</b> per aprire l'Inspector laterale e configurarne i parametri, passaggio per passaggio.</li>
  </ul>

  <div class="toolbar">
    <div class="seg" id="modeSeg">
      <button data-mode="free" class="on">Libero</button>
      <button data-mode="grid">Organizzato</button>
    </div>
    <button class="tool-btn" id="autoBtn">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="6" height="6" rx="1.5"/><rect x="15" y="4" width="6" height="6" rx="1.5"/><rect x="9" y="14" width="6" height="6" rx="1.5"/><path d="M6 10v2a2 2 0 0 0 2 2h1"/><path d="M18 10v2a2 2 0 0 1-2 2h-1"/></svg>
      Riordina
    </button>
    <button class="tool-btn" id="undoBtn" disabled>
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 14L4 9l5-5"/><path d="M4 9h11a5 5 0 0 1 0 10h-3"/></svg>
      Annulla
    </button>
    <button class="tool-btn" id="redoBtn" disabled>
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M15 14l5-5-5-5"/><path d="M20 9H9a5 5 0 0 0 0 10h3"/></svg>
      Ripristina
    </button>
    <button class="tool-btn" id="resetBtn2">Reimposta</button>
    <div class="feat-wrap">
      <button class="tool-btn" id="featBtn">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="4" y1="7" x2="20" y2="7"/><line x1="4" y1="17" x2="20" y2="17"/><circle cx="9" cy="7" r="2.4" fill="#fff"/><circle cx="15" cy="17" r="2.4" fill="#fff"/></svg>
        Funzionalità
      </button>
      <div class="feat-panel" id="featPanel"></div>
    </div>
  </div>

  <div class="hint" id="hint">Trascina una lavorazione su un'altra per fonderle, oppure un dataset su un box per collegarlo</div>

  <div class="workspace" id="workspace">
    <div id="dock-top"></div>
    <div id="dock-left"><aside class="panel side-left open" id="toolbox" style="--pw:264px;--ph:206px"><div class="tb-inner" id="tbInner"></div></aside></div>
    <input type="file" id="dsFile" accept=".csv,.tsv,.txt" hidden>
    <div class="stage" id="stage">
      <div class="edge-hint" id="edgeHint"></div>
      <button type="button" class="notch n-left hidden" id="tbNotch" aria-label="Cassetta degli strumenti: clicca per aprire, trascina per spostarla">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="4" width="6.5" height="6.5" rx="1.5"/><rect x="13.5" y="4" width="6.5" height="6.5" rx="1.5"/><rect x="4" y="13.5" width="6.5" height="6.5" rx="1.5"/><rect x="13.5" y="13.5" width="6.5" height="6.5" rx="1.5"/></svg>
      </button>
      <button type="button" class="notch n-right" id="inspNotch" aria-label="Inspector: clicca per aprire, trascina per spostarlo">
```


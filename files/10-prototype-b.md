# 10-prototype-b.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 2/6)

5098 righe totali

```html
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="4" y1="7" x2="20" y2="7"/><line x1="4" y1="17" x2="20" y2="17"/><circle cx="9" cy="7" r="2.4" fill="#fff"/><circle cx="15" cy="17" r="2.4" fill="#fff"/></svg>
      </button>
      <div class="world" id="world">
        <svg class="links" id="links">
          <g id="linkPaths" class="nohit"></g>
          <g id="bubbles" class="nohit"></g>
          <g id="linkHits"></g>
          <g id="tempLink" class="nohit"></g>
        </svg>
        <div class="slot-ghost" id="slotGhost"></div>
      </div>
      <div class="marquee" id="marquee"></div>
      <div class="minimap" id="minimap"><div class="mm-view" id="mmView"></div></div>
      <div class="zoom-ctl" id="zoomCtl">
        <button type="button" id="zoomOut" aria-label="Riduci">−</button>
        <button type="button" id="zoomPct" aria-label="Zoom al 100%">100%</button>
        <button type="button" id="zoomIn" aria-label="Ingrandisci">+</button>
        <button type="button" id="zoomFit" class="fit">Adatta</button>
      </div>
    </div>
    <div id="dock-right"><aside class="panel side-right" id="inspector" style="--pw:308px;--ph:236px"><div class="insp-inner" id="inspInner"></div></aside></div>
    <div id="dock-bottom"></div>
  </div>

</div>

<div class="confirm" id="confirmBox">
  <div>
    <div class="confirm-title">Eliminare il nodo?</div>
    <div class="confirm-text" id="confirmText"></div>
  </div>
  <div class="confirm-actions">
    <button class="c-cancel" id="confirmCancel">Annulla</button>
    <button class="c-ok" id="confirmOk">Elimina</button>
  </div>
</div>

<div class="overlay" id="overlay">
  <div class="expand-card">
    <div class="expand-head">
      <div>
        <div class="expand-title" id="expandTitle" contenteditable="true"></div>
        <div class="expand-sub" id="expandSub"></div>
      </div>
      <button class="close-btn" id="closeBtn" aria-label="Chiudi">
        <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
      </button>
    </div>
    <p class="reorder-note">Trascina le righe per cambiare l'ordine di esecuzione.</p>
    <div class="steps" id="steps"></div>
  </div>
</div>

<script>
  const ICONS = {
    filter: '<path d="M4 4h16l-6 8v6l-4 2v-8z"/>',
    dataset: '<ellipse cx="12" cy="5" rx="8" ry="3"/><path d="M4 5v6c0 1.7 3.6 3 8 3s8-1.3 8-3V5"/><path d="M4 11v6c0 1.7 3.6 3 8 3s8-1.3 8-3v-6"/>',
    join: '<circle cx="9" cy="12" r="6.5"/><circle cx="15" cy="12" r="6.5"/>',
    sort: '<path d="M8 9l4-4 4 4"/><path d="M16 15l-4 4-4-4"/>',
    exportOp: '<path d="M14 3h7v7"/><path d="M21 3l-9 9"/><path d="M5 12v7a2 2 0 0 0 2 2h7"/>',
    empty: '<path d="M9 7l-5 5 5 5"/><path d="M15 7l5 5-5 5"/>',
    dedup: '<rect x="4" y="4" width="11" height="11" rx="2"/><rect x="9" y="9" width="11" height="11" rx="2"/>',
    limit: '<line x1="4" y1="6" x2="20" y2="6"/><line x1="4" y1="11" x2="20" y2="11"/><line x1="4" y1="16" x2="11" y2="16"/><path d="M15 14l3 3 3-3"/>',
    sample: '<circle cx="6" cy="6" r="1.8"/><circle cx="12" cy="12" r="1.8"/><circle cx="18" cy="8" r="1.8"/><circle cx="8" cy="18" r="1.8"/><circle cx="17" cy="17" r="1.8"/>',
    selectCols: '<rect x="4" y="4" width="4" height="16" rx="1"/><rect x="10" y="4" width="4" height="16" rx="1"/><rect x="16" y="4" width="4" height="16" rx="1"/>',
    compute: '<path d="M9 20c2 0 2-4 3-8s1-8 3-8"/><line x1="7" y1="11" x2="15" y2="11"/>',
    cast: '<path d="M5 8h13l-3-3"/><path d="M19 16H6l3 3"/>',
    round: '<circle cx="12" cy="12" r="8"/><circle cx="12" cy="12" r="2"/>',
    scale: '<line x1="6" y1="20" x2="6" y2="14"/><line x1="12" y1="20" x2="12" y2="9"/><line x1="18" y1="20" x2="18" y2="4"/>',
    aggregate: '<path d="M18 5H7l6 7-6 7h11"/>',
    textClean: '<path d="M5 6h14"/><path d="M12 6v13"/>',
    replaceVal: '<path d="M4 7h11"/><path d="M12 4l3 3-3 3"/><path d="M20 17H9"/><path d="M12 14l-3 3 3 3"/>',
    splitCol: '<path d="M12 4v16"/><path d="M4 8l4 4-4 4"/><path d="M20 8l-4 4 4 4"/>',
    rename: '<path d="M4 20h4L19 9l-4-4L4 16z"/>',
    fillNa: '<rect x="4" y="4" width="16" height="16" rx="3"/><path d="M8 12h8"/><path d="M12 8v8"/>',
    union: '<rect x="5" y="4" width="14" height="6" rx="1.5"/><rect x="5" y="14" width="14" height="6" rx="1.5"/>',
    upload: '<path d="M12 16V4"/><path d="M7 9l5-5 5 5"/><path d="M5 20h14"/>'
  };
  const META = {
    filter:{label:'Filtra Righe'}, dataset:{label:'Vendite 2026'}, join:{label:'Unisci (Join)'},
    sort:{label:'Ordina'}, exportOp:{label:'Esporta'},
    dedup:{label:'Rimuovi duplicati'}, limit:{label:'Limita righe'}, sample:{label:'Campiona'},
    selectCols:{label:'Seleziona colonne'},
    compute:{label:'Calcola colonna'}, cast:{label:'Converti tipo'}, round:{label:'Arrotonda'},
    scale:{label:'Normalizza'}, aggregate:{label:'Raggruppa'}, textClean:{label:'Pulisci testo'},
    replaceVal:{label:'Sostituisci valori'}, splitCol:{label:'Dividi colonna'}, rename:{label:'Rinomina'},
    fillNa:{label:'Riempi vuoti'}, union:{label:'Accoda (Union)'}
  };
  const GEAR = '<circle cx="12" cy="12" r="3"/><path d="M12 2v3M12 19v3M4.2 4.2l2.1 2.1M17.7 17.7l2.1 2.1M2 12h3M19 12h3M4.2 19.8l2.1-2.1M17.7 6.3l2.1-2.1"/>';
  const GRIP = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="4" y1="9" x2="20" y2="9"/><line x1="4" y1="15" x2="20" y2="15"/></svg>';
  const EXPAND_ICON = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M8 3H5a2 2 0 0 0-2 2v3"/><path d="M21 8V5a2 2 0 0 0-2-2h-3"/><path d="M3 16v3a2 2 0 0 0 2 2h3"/><path d="M16 21h3a2 2 0 0 0 2-2v-3"/></svg>';

  const CARD = 88;
  const stage = document.getElementById('stage');
  const world = document.getElementById('world');

  // --- Funzionalita attivabili singolarmente: ognuna si prova e si scarta in isolamento ---
  const FEATURES = {
    portDrag:    true,   // collegare trascinando da una porta
    cableInsert: true,   // inserire una lavorazione rilasciandola su un cavo
    nodeStates:  true,   // indicatore di nodo incompleto
    flowGate:    false,  // il flusso scorre solo tra nodi validi
    panZoom:     true,   // canvas navigabile
    minimap:     true,   // minimappa
    multiSelect: true,   // selezione a riquadro e con Maiusc
    keyboard:    true    // scorciatoie da tastiera
  };
  const FEATURE_INFO = [
    ['portDrag',    'Collegamento dalle porte',   'Trascina da uno dei quattro punti sul bordo di un nodo verso un altro.'],
    ['cableInsert', 'Inserimento su cavo',        'Rilascia una lavorazione su un collegamento per inserirla nel mezzo.'],
    ['nodeStates',  'Stati sui nodi',             'Un indicatore segnala i nodi incompleti o da configurare.'],
    ['flowGate',    'Flusso solo se valido',      'Le palline scorrono solo tra nodi configurati correttamente.'],
    ['panZoom',     'Pan e zoom',                 'Rotella o due dita per spostarti, Cmd/Ctrl + rotella per lo zoom.'],
    ['minimap',     'Minimappa',                  'Panoramica del flusso con la porzione visibile.'],
    ['multiSelect', 'Selezione multipla',         'Riquadro sul vuoto o Maiusc+click; i nodi si spostano insieme.'],
    ['keyboard',    'Scorciatoie da tastiera',    'Canc elimina, Cmd+D duplica, frecce spostano, Cmd+A seleziona tutto.']
  ];

  // --- Vista sul mondo: con pan e zoom il mondo e piu grande della finestra ---
  const WORLD_W = 2600, WORLD_H = 1600;
  const view = { x: 0, y: 0, z: 1 };
  function worldW(){ return FEATURES.panZoom ? WORLD_W : stage.clientWidth; }
  function worldH(){ return FEATURES.panZoom ? WORLD_H : stage.clientHeight; }
  function applyView(ease){
    world.classList.toggle('easing', !!ease);
    world.style.transform = 'translate(' + view.x + 'px,' + view.y + 'px) scale(' + view.z + ')';
    if (ease) setTimeout(() => world.classList.remove('easing'), 400);
    const zp = document.getElementById('zoomPct');
    if (zp) zp.textContent = Math.round(view.z * 100) + '%';
  }
  function toWorld(clientX, clientY){
    const r = stage.getBoundingClientRect();
    return { x: (clientX - r.left - view.x) / view.z, y: (clientY - r.top - view.y) / view.z };
  }
  const linkPaths = document.getElementById('linkPaths');
  const bubblesG = document.getElementById('bubbles');
  const hint = document.getElementById('hint');
  const resetBtn = document.getElementById('resetBtn2');
  const paletteEl = document.getElementById('tbInner');
  const undoBtn = document.getElementById('undoBtn');
  const autoBtn = document.getElementById('autoBtn');
  const redoBtn = document.getElementById('redoBtn');
  const linkHits = document.getElementById('linkHits');
  const modeSeg = document.getElementById('modeSeg');
  const slotGhost = document.getElementById('slotGhost');
  const confirmBox = document.getElementById('confirmBox');
  const confirmText = document.getElementById('confirmText');
  const confirmOk = document.getElementById('confirmOk');
  const confirmCancel = document.getElementById('confirmCancel');
  const workspace = document.getElementById('workspace');
  const inspector = document.getElementById('inspector');
  const inspInner = document.getElementById('inspInner');
  const overlay = document.getElementById('overlay');
  const stepsEl = document.getElementById('steps');
  const expandTitle = document.getElementById('expandTitle');
  const expandSub = document.getElementById('expandSub');
  const closeBtn = document.getElementById('closeBtn');
  const DEFAULT_HINT = 'Trascina una lavorazione su un\u2019altra per fonderle, oppure un dataset su un box per collegarlo';

  let cards = {}, linksArr = [], uidCounter = 0, comboCounter = 0, outCounter = 0, dsCounter = 1;
  let currentExpandedUid = null;

  function svgTag(id){
    return '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">' + ICONS[id] + '</svg>';
  }
  function countClass(n){
    if (n<=1) return 'count-1'; if (n===2) return 'count-2';
    if (n<=3) return 'count-3'; if (n<=6) return 'count-6'; return 'count-many';
  }
  function cardEl(uid){ return stage.querySelector('[data-uid="'+uid+'"]'); }

  // accessori sempre presenti su ogni nodo: elimina, porte, indicatore di stato
  const NODE_EXTRAS =
    '<button class="del-btn" aria-label="Elimina nodo"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="17" y1="7" x2="7" y2="17"/><line x1="7" y1="7" x2="17" y2="17"/></svg></button>'+
    '<span class="port p-t" data-port="t"></span><span class="port p-r" data-port="r"></span>'+
    '<span class="port p-b" data-port="b"></span><span class="port p-l" data-port="l"></span>'+
    '<span class="state-dot"></span>';
  function wrapInner(comps, combined){
    return comps.map(svgTag).join('') +
      (combined ? '<button class="expand-btn" aria-label="Espandi sequenza">'+EXPAND_ICON+'</button>' : '') +
      NODE_EXTRAS;
  }

  function createCardEl(uid){
    const d = cards[uid];
    const el = document.createElement('div');
    el.className = 'card' + (d.kind==='dataset' ? (d.isOutput ? ' dataset output' : ' dataset') : '')
                 + (d.kind==='op' && d.components.length>1 ? ' combined' : '');
    el.dataset.uid = uid;
    el.style.left = d.x+'px'; el.style.top = d.y+'px';
    const combined = d.kind === 'op' && d.components.length > 1;
    el.innerHTML =
      '<div class="icon-wrap '+countClass(d.components.length)+'">'+ wrapInner(d.components, combined) +'</div>'+
      '<div class="label">'+d.name+'</div>';
    world.appendChild(el);
    if (d.isOutput) renderOutputIcon(uid);
    return el;
  }

  // --- Ancoraggi: 4 porte per box, il punto scivola lungo il bordo senza scatti ---
  const linkState = {};
  const PORTS = [0, Math.PI/2, Math.PI, -Math.PI/2];
  const PERIOD = 2300;      // durata di un transito completo del bolo
  const STUB = 32, ELBOW_R = 11;
  let MAX_BENDS = 1;            // snodi ammessi per collegamento (futuro parametro di View)

  function nearestPort(angle){
    let best = PORTS[0], bestD = Infinity;
    for (const p of PORTS){
      const d = Math.abs(Math.atan2(Math.sin(angle - p), Math.cos(angle - p)));
      if (d < bestD){ bestD = d; best = p; }
    }
    return best;
  }
  function borderPoint(cx, cy, half, angle){
    const dx = Math.cos(angle), dy = Math.sin(angle);
    const m = Math.max(Math.abs(dx), Math.abs(dy)) || 1;
    return { x: cx + dx / m * half, y: cy + dy / m * half, nx: dx, ny: dy };
  }
  // la porta cambia solo quando lo scostamento e netto: niente oscillazioni
  function stickyPort(current, angle){
    const desired = nearestPort(angle);
    if (current === undefined || desired === current) return desired;
    const dCur = Math.abs(Math.atan2(Math.sin(angle - current), Math.cos(angle - current)));
    return dCur > 0.95 ? desired : current;
  }
  function easeAngle(cur, target, k){
    const diff = Math.atan2(Math.sin(target - cur), Math.cos(target - cur));
    return cur + diff * k;
  }
  function dotFill(card){ return card.kind === 'dataset' ? '#6C63FF' : '#E1DCF0'; }

  // asse dominante dell'ancoraggio: da qui parte il tratto rettilineo
  function axisOf(nx, ny){
    return Math.abs(nx) >= Math.abs(ny) ? { x: Math.sign(nx) || 1, y: 0 } : { x: 0, y: Math.sign(ny) || 1 };
  }

  // --- Instradamento che evita i box ---
  const OBST_PAD = 9;
  function segHitsRect(x1, y1, x2, y2, r){
    if (Math.abs(y1 - y2) < 0.5){
      const lo = Math.min(x1, x2), hi = Math.max(x1, x2);
      return y1 > r.y && y1 < r.y + r.h && hi > r.x && lo < r.x + r.w;
    }
    if (Math.abs(x1 - x2) < 0.5){
      const lo = Math.min(y1, y2), hi = Math.max(y1, y2);
      return x1 > r.x && x1 < r.x + r.w && hi > r.y && lo < r.y + r.h;
    }
    return false;
  }
  function routeCost(pts, obstacles){
    let hits = 0;
    for (let i = 0; i < pts.length - 1; i++)
      for (const r of obstacles)
        if (segHitsRect(pts[i].x, pts[i].y, pts[i+1].x, pts[i+1].y, r)) hits++;
    return hits;
  }
  // knob = coordinata dello snodo intermedio (X se si esce in orizzontale, Y se in verticale)
  function buildRoute(pa, da, pb, db, knob){
    const p1 = { x: pa.x + da.x * STUB, y: pa.y + da.y * STUB };
    const p2 = { x: pb.x + db.x * STUB, y: pb.y + db.y * STUB };
    const pts = [ { x: pa.x, y: pa.y }, p1 ];
    if (da.x !== 0) pts.push({ x: knob, y: p1.y }, { x: knob, y: p2.y });
    else            pts.push({ x: p1.x, y: knob }, { x: p2.x, y: knob });
    pts.push(p2, { x: pb.x, y: pb.y });
    return pts;
  }
  function defaultKnob(pa, da, pb, db){
    const p1 = { x: pa.x + da.x * STUB, y: pa.y + da.y * STUB };
    const p2 = { x: pb.x + db.x * STUB, y: pb.y + db.y * STUB };
    return da.x !== 0 ? (p1.x + p2.x) / 2 : (p1.y + p2.y) / 2;
  }
  function countBends(pts){
    const p = pts.filter((q, i) => i === 0 || Math.hypot(q.x - pts[i-1].x, q.y - pts[i-1].y) > 0.5);
    let bends = 0;
    for (let i = 1; i < p.length - 1; i++){
      const h1 = Math.abs(p[i].y - p[i-1].y) < 0.5;
      const h2 = Math.abs(p[i+1].y - p[i].y) < 0.5;
      if (h1 !== h2) bends++;
    }
    return bends;
  }
  // incroci e sovrapposizioni fra cavi: un aggancio diverso spesso li evita
  function segCross(a, b, c, d){
    const m = 3;
    const sH = Math.abs(a.y - b.y) < 0.5, tH = Math.abs(c.y - d.y) < 0.5;
    if (sH && !tH){
      const x = c.x, y = a.y;
      return x > Math.min(a.x, b.x) + m && x < Math.max(a.x, b.x) - m &&
             y > Math.min(c.y, d.y) + m && y < Math.max(c.y, d.y) - m;
    }
    if (!sH && tH) return segCross(c, d, a, b);
    if (sH && tH && Math.abs(a.y - c.y) < 4){
      const lo = Math.max(Math.min(a.x, b.x), Math.min(c.x, d.x));
      const hi = Math.min(Math.max(a.x, b.x), Math.max(c.x, d.x));
      return hi - lo > 8;                    // due cavi sullo stesso binario
    }
    if (!sH && !tH && Math.abs(a.x - c.x) < 4){
      const lo = Math.max(Math.min(a.y, b.y), Math.min(c.y, d.y));
      const hi = Math.min(Math.max(a.y, b.y), Math.max(c.y, d.y));
      return hi - lo > 8;
    }
    return false;
  }
  function countCrossings(pts, others){
    let n = 0;
    for (let i = 0; i < pts.length - 1; i++)
      for (const o of others)
        for (let j = 0; j < o.length - 1; j++)
          if (segCross(pts[i], pts[i+1], o[j], o[j+1])) n++;
    return n;
  }

  // quanto il tracciato esce dal rettangolo che unisce i due agganci: i ripieghi inutili costano
  function overshoot(pts){
    const qa = pts[0], qb = pts[pts.length - 1];
    const minX = Math.min(qa.x, qb.x), maxX = Math.max(qa.x, qb.x);
    const minY = Math.min(qa.y, qb.y), maxY = Math.max(qa.y, qb.y);
    let o = 0;
    for (const p of pts){
      o += Math.max(0, minX - p.x) + Math.max(0, p.x - maxX)
         + Math.max(0, minY - p.y) + Math.max(0, p.y - maxY);
    }
    return o;
  }
  function routeLength(pts){
    let L = 0;
    for (let i = 0; i < pts.length - 1; i++) L += Math.abs(pts[i+1].x - pts[i].x) + Math.abs(pts[i+1].y - pts[i].y);
    return L;
  }
  // Forme di tracciato disponibili: dritto (0 snodi), a L (1), a Z (2)
  function slide(pt, axis, off){
    if (!off) return { x: pt.x, y: pt.y, nx: pt.nx, ny: pt.ny };
    return axis.x !== 0 ? { x: pt.x, y: pt.y + off, nx: pt.nx, ny: pt.ny }
                        : { x: pt.x + off, y: pt.y, nx: pt.nx, ny: pt.ny };
  }
  function shapeAnchors(pa, da, pb, db, shape){
    return [ slide(pa, da, shape.oa || 0), slide(pb, db, shape.ob || 0) ];
  }
  // nessun segmento obliquo: una diagonale produrrebbe una cuspide
  function orthogonalize(pts, da){
    const out = [ pts[0] ];
    let horizFirst = da.x !== 0;
    for (let i = 1; i < pts.length; i++){
      const p = out[out.length - 1], q = pts[i];
      const dx = Math.abs(q.x - p.x), dy = Math.abs(q.y - p.y);
      if (dx > 0.5 && dy > 0.5){
        out.push(horizFirst ? { x: q.x, y: p.y } : { x: p.x, y: q.y });
      }
      out.push(q);
      const last = out[out.length - 1], prev = out[out.length - 2];
      horizFirst = Math.abs(last.y - prev.y) < 0.5;   // prosegue sull'asse appena percorso
    }
    return out;
  }
  // un tratto che torna indietro sullo stesso asse e una cuspide: il punto intermedio va eliminato
  function removeReversals(pts){
    let p = pts.slice();
    let changed = true;
    while (changed && p.length > 2){
      changed = false;
      for (let i = 1; i < p.length - 1; i++){
        const a = p[i-1], b = p[i], c = p[i+1];
        const sameX = Math.abs(a.x - b.x) < 0.5 && Math.abs(b.x - c.x) < 0.5;
        const sameY = Math.abs(a.y - b.y) < 0.5 && Math.abs(b.y - c.y) < 0.5;
        if (!sameX && !sameY) continue;
        const dot = (b.x - a.x) * (c.x - b.x) + (b.y - a.y) * (c.y - b.y);
        // inversione, oppure punto superfluo sulla stessa retta
        if (dot <= 0 || sameX || sameY){ p.splice(i, 1); changed = true; break; }
      }
    }
    return p;
  }
  function buildFromShape(pa, da, pb, db, shape){
    const [qa, qb] = shapeAnchors(pa, da, pb, db, shape);
    let pts;
    if (shape.kind === 'straight') pts = [ {x:qa.x,y:qa.y}, {x:qb.x,y:qb.y} ];
    else if (shape.kind === 'L'){
      const corner = shape.first === 'h' ? { x: qb.x, y: qa.y } : { x: qa.x, y: qb.y };
      pts = [ {x:qa.x,y:qa.y}, corner, {x:qb.x,y:qb.y} ];
    }
    else pts = buildRoute(qa, da, qb, db, shape.knob);
    return removeReversals(orthogonalize(pts, da));
  }
  function dirSign(p, q){ return { x: Math.sign(q.x - p.x), y: Math.sign(q.y - p.y) }; }
  // penalizza i tracciati che tornano indietro rispetto alla porta da cui escono
  function backtrack(pts, da, db){
    if (pts.length < 2) return 0;
    let pen = 0;
    const d0 = dirSign(pts[0], pts[1]);
    if ((da.x && d0.x && d0.x !== da.x) || (da.y && d0.y && d0.y !== da.y)) pen++;
    const n = pts.length;
    const dn = dirSign(pts[n-2], pts[n-1]);
    if ((db.x && dn.x && dn.x === db.x) || (db.y && dn.y && dn.y === db.y)) pen++;
    return pen;
  }

  function shapeCandidates(pa, da, pb, db, obstacles, slack){
    const out = [];
    if (da.x && db.x && da.x === -db.x){
      const dy = pb.y - pa.y;
      if (Math.abs(dy) < 1.5) out.push({ kind:'straight' });
      else if (Math.abs(dy) <= slack * 2) out.push({ kind:'straight', oa: dy / 2, ob: -dy / 2 });
    }
    if (da.y && db.y && da.y === -db.y){
      const dx = pb.x - pa.x;
      if (Math.abs(dx) < 1.5) out.push({ kind:'straight' });
      else if (Math.abs(dx) <= slack * 2) out.push({ kind:'straight', oa: dx / 2, ob: -dx / 2 });
    }
    if (da.x && db.y) out.push({ kind:'L', first:'h' });
    if (da.y && db.x) out.push({ kind:'L', first:'v' });
    const p1 = { x: pa.x + da.x * STUB, y: pa.y + da.y * STUB };
    const p2 = { x: pb.x + db.x * STUB, y: pb.y + db.y * STUB };
    const base = da.x !== 0 ? (p1.x + p2.x) / 2 : (p1.y + p2.y) / 2;
    const horiz = da.x !== 0;
    const knobs = [base];
    for (const r of obstacles){
      if (horiz) knobs.push(r.x - 14, r.x + r.w + 14);
      else       knobs.push(r.y - 14, r.y + r.h + 14);
    }
    for (let k = 1; k <= 8; k++) knobs.push(base + k * 24, base - k * 24);
    knobs.forEach(kn => out.push({ kind:'Z', knob: kn }));
    return out;
  }

  // Sceglie porte e forma: niente intersezioni, pochi snodi, percorso piu corto
  function chooseRoute(acx, acy, bcx, bcy, half, obstacles, st, others){
    // anche i due nodi collegati sono ostacoli: un cavo non puo attraversarli per raggiungerli
    const inset = 2;
    const selfObs = obstacles.concat([
      { x: acx - half + inset, y: acy - half + inset, w: half * 2 - inset * 2, h: half * 2 - inset * 2 },
      { x: bcx - half + inset, y: bcy - half + inset, w: half * 2 - inset * 2, h: half * 2 - inset * 2 }
    ]);
    let best = null, keep = null;
    for (const pA of PORTS){
      const pa = borderPoint(acx, acy, half, pA);
      const da = axisOf(Math.cos(pA), Math.sin(pA));
      for (const pB of PORTS){
        const pb = borderPoint(bcx, bcy, half, pB);
        const db = axisOf(Math.cos(pB), Math.sin(pB));
        for (const shape of shapeCandidates(pa, da, pb, db, obstacles, half - 14)){
          const pts = buildFromShape(pa, da, pb, db, shape);
          const cost = routeCost(pts, selfObs);
          const overBends = Math.max(0, countBends(pts) - MAX_BENDS);
          const back = backtrack(pts, da, db);
          const changePen = (pA !== st.portA ? 1 : 0) + (pB !== st.portB ? 1 : 0);
          // uscire e rientrare dallo stesso lato (forma a U) solo se non c'e altro modo
          const sameSide = (da.x === db.x && da.y === db.y) ? 1 : 0;
          const bends = countBends(pts);
          const cr = (others && others.length) ? countCrossings(pts, others) : 0;
          const score = cost * 1e6 + overBends * 6000 + cr * 3500 + back * 3000 + sameSide * 1800 +
                        overshoot(pts) * 14 + routeLength(pts) + changePen * 0.8;
          const cand = { score, portA: pA, portB: pB, shape, horiz: da.x !== 0, cost, bends, back, cr };
          if (!best || score < best.score) best = cand;
          if (pA === st.portA && pB === st.portB && (!keep || score < keep.score)) keep = cand;
        }
      }
    }
    // un cavo si sposta solo se serve: il percorso attuale vince finche non attraversa nulla
    // e non costa piu angoli dell'alternativa migliore
    if (keep && best && keep.cost === 0 && keep.back === 0 && keep.bends <= best.bends && keep.cr <= best.cr) return keep;
    return best;
  }

  function roundedPath(pts, r){
    const clean = pts.filter((p, i) => i === 0 || Math.hypot(p.x - pts[i-1].x, p.y - pts[i-1].y) > 0.5);
    if (clean.length < 3) return 'M ' + clean.map(p => p.x.toFixed(2) + ' ' + p.y.toFixed(2)).join(' L ');
    let d = 'M ' + clean[0].x.toFixed(2) + ' ' + clean[0].y.toFixed(2);
    for (let i = 1; i < clean.length - 1; i++){
      const prev = clean[i-1], cur = clean[i], next = clean[i+1];
      const l1 = Math.hypot(cur.x - prev.x, cur.y - prev.y);
      const l2 = Math.hypot(next.x - cur.x, next.y - cur.y);
      const rr = Math.min(r, l1 / 2, l2 / 2);
      const a = { x: cur.x + (prev.x - cur.x) / (l1 || 1) * rr, y: cur.y + (prev.y - cur.y) / (l1 || 1) * rr };
      const b = { x: cur.x + (next.x - cur.x) / (l2 || 1) * rr, y: cur.y + (next.y - cur.y) / (l2 || 1) * rr };
      d += ' L ' + a.x.toFixed(2) + ' ' + a.y.toFixed(2) +
           ' Q ' + cur.x.toFixed(2) + ' ' + cur.y.toFixed(2) + ' ' + b.x.toFixed(2) + ' ' + b.y.toFixed(2);
    }
    const last = clean[clean.length-1];
    d += ' L ' + last.x.toFixed(2) + ' ' + last.y.toFixed(2);
    return d;
  }

  function drawLinks(){
    const half = CARD / 2;
    const now = performance.now();

    // primo passaggio: porte, forme e punti di aggancio
    const items = [];
    linksArr.forEach((l, i) => {
      const a = cards[l.from], b = cards[l.to];
      if (!a || !b) return;
      const acx = a.x + half, acy = a.y + half;
      const bcx = b.x + half, bcy = b.y + half;
      const key = l.from + '|' + l.to;
      if (!linkState[key]) linkState[key] = { portA: undefined, portB: undefined, a: null, b: null,
                                              t0: now, knob: null, knobTarget: null, shape: null, nextEval: 0, horiz: true };
      const st = linkState[key];

      const obstacles = Object.keys(cards)
        .filter(id => id !== l.from && id !== l.to && id !== draggingUid)
        .map(id => ({ x: cards[id].x - OBST_PAD, y: cards[id].y - OBST_PAD,
                      w: CARD + OBST_PAD * 2, h: CARD + LABEL_H + OBST_PAD * 2 }));

      if (now >= st.nextEval){
        st.nextEval = now + 110;
        const others = linksArr
          .filter(o => o !== l)
          .map(o => linkState[o.from + '|' + o.to])
          .filter(os => os && os.pts)
          .map(os => os.pts);
        const best = chooseRoute(acx, acy, bcx, bcy, half, obstacles, st, others);
        if (best){
          const changed = st.horiz !== best.horiz || !st.shape || st.shape.kind !== best.shape.kind;
          st.portA = best.portA; st.portB = best.portB; st.horiz = best.horiz;
          st.shape = best.shape;
          st.knobTarget = (best.shape.kind === 'Z') ? best.shape.knob : null;
          if (st.knob === null || changed) st.knob = st.knobTarget;
          if (st.a === null){ st.a = best.portA; st.b = best.portB; }
        }
      }
      if (st.portA === undefined) return;
      st.a = easeAngle(st.a, st.portA, 0.065);
      st.b = easeAngle(st.b, st.portB, 0.065);
      if (st.knobTarget !== null && st.knob !== null) st.knob += (st.knobTarget - st.knob) * 0.075;

      items.push({ i, l, a, b, st, acx, acy, bcx, bcy });
    });

    // piu cavi sulla stessa porta: si distanziano lungo il bordo invece di sovrapporsi
    const groups = {};
    items.forEach(it => {
      (groups[it.l.from + '#' + it.st.portA] = groups[it.l.from + '#' + it.st.portA] || []).push({ it, end:'a' });
      (groups[it.l.to   + '#' + it.st.portB] = groups[it.l.to   + '#' + it.st.portB] || []).push({ it, end:'b' });
    });
    const spread = {};
    Object.values(groups).forEach(arr => {
      arr.forEach((entry, idx) => {
        const off = arr.length > 1 ? (idx - (arr.length - 1) / 2) * 15 : 0;
        spread[entry.it.i + entry.end] = off;
      });
    });
    function shift(pt, axis, off){
      if (!off) return pt;
      return axis.x !== 0 ? { x: pt.x, y: pt.y + off } : { x: pt.x + off, y: pt.y };
    }

    // cavi che corrono nello stesso corridoio: ognuno riceve una corsia propria
    const lanes = [];
    items.forEach(it => {
      const st = it.st;
      if (!st.shape || st.shape.kind !== 'Z' || st.knob === null) return;
      const da0 = axisOf(Math.cos(st.portA), Math.sin(st.portA));
      const pa0 = borderPoint(it.acx, it.acy, half, st.a);
      const pb0 = borderPoint(it.bcx, it.bcy, half, st.b);
      const horizSeg = da0.x === 0;             // uscita verticale: lo snodo e un tratto orizzontale
      const lo = horizSeg ? Math.min(pa0.x, pb0.x) : Math.min(pa0.y, pb0.y);
      const hi = horizSeg ? Math.max(pa0.x, pb0.x) : Math.max(pa0.y, pb0.y);
      const dir = horizSeg ? (da0.y || 1) : (da0.x || 1);
      let lane = 0;
      while (lanes.some(o => o.horizSeg === horizSeg && o.lane === lane &&
                             Math.abs(o.knob - st.knob) < 12 && o.lo < hi && lo < o.hi)) lane++;
      lanes.push({ horizSeg, knob: st.knob, lo, hi, lane });
      it.laneOff = lane * 14 * dir;             // le corsie successive si allontanano dai nodi
    });

    const hits = [];
    linkPaths.innerHTML = items.map(it => {
      const { i, st, a, b, acx, acy, bcx, bcy } = it;
      const da = axisOf(Math.cos(st.portA), Math.sin(st.portA));
      const db = axisOf(Math.cos(st.portB), Math.sin(st.portB));
      const pa0 = borderPoint(acx, acy, half, st.a);
      const pb0 = borderPoint(bcx, bcy, half, st.b);
      const shape = st.shape.kind === 'Z'
        ? { kind:'Z', knob: st.knob + (it.laneOff || 0), oa: st.shape.oa, ob: st.shape.ob }
        : st.shape;
      const sprA = spread[i + 'a'] || 0, sprB = spread[i + 'b'] || 0;
      // su un tratto rettilineo i due capi devono scostarsi insieme, o smette di essere rettilineo
      const common = shape.kind === 'straight' ? (sprA + sprB) / 2 : null;
      const pa = shift(slide(pa0, da, shape.oa || 0), da, common !== null ? common : sprA);
      const pb = shift(slide(pb0, db, shape.ob || 0), db, common !== null ? common : sprB);

      const routePts = buildFromShape(pa, da, pb, db, { kind: shape.kind, first: shape.first, knob: shape.knob });
      st.pts = routePts;
      const d = roundedPath(routePts, ELBOW_R);
      hits.push('<path class="hit" data-link="'+i+'" d="'+d+'" fill="none" stroke="transparent" stroke-width="16"><title>Scollega</title></path>');
      const hot = i === hoverLinkIdx;
      return '<path id="lp-'+i+'" d="'+d+'" fill="none" stroke="'+(hot ? '#6C63FF' : 'rgba(108,99,255,0.34)')+
             '" stroke-width="'+(hot ? 4 : BASE_W)+'" stroke-linecap="round" stroke-linejoin="round"'+
             (hot ? ' stroke-dasharray="7 5"' : '')+'/>'+
             '<circle cx="'+pa.x.toFixed(2)+'" cy="'+pa.y.toFixed(2)+'" r="2.6" fill="'+dotFill(a)+'"/>'+
             '<circle cx="'+pb.x.toFixed(2)+'" cy="'+pb.y.toFixed(2)+'" r="2.6" fill="'+dotFill(b)+'"/>';
    }).join('');
    linkHits.innerHTML = hits.join('');
  }

  // Il condotto e un tubo elastico: una pallina lo attraversa, il tubo si apre davanti
  // e si richiude dietro. La forma e un unico contorno continuo, non tratti accostati.
  const BASE_W = 2.1;
  const SPEED = 0.16;               // px al millisecondo: stessa andatura su cavi lunghi e corti
  const BALL = 4.4;                 // rigonfiamento massimo per lato
  const FRONT = 7.5, BACK = 19;     // apertura rapida davanti, richiusura piu lenta dietro
  function tubeProfile(u){
    const sg = u >= 0 ? FRONT : BACK;
    return Math.exp(-(u * u) / (sg * sg));
  }
  let flowPaused = false;          // durante lo spostamento di un nodo il flusso si ferma

  // --- Stato di un nodo: completo, oppure cosa manca ---
  function stepMissing(type, par){
    if (!par) return true;
    if (type === 'filter'){
      const cs = par.conditions || [];
      return !cs.length || cs.some(c => !c.column || (!NO_VALUE_OPS.includes(c.op) &&
        !(c.text && c.text.trim()) && !(c.values && c.values.length)));
    }
    if (type === 'join'){
      const ks = ensureKeys(par);
      return !ks.length || ks.some(k => !keyComplete(k));
    }
    if (MULTI_DEFS[type]){
      ensureMulti(type, par);
      return MULTI_DEFS[type].lists.some(L => !par[L.key].length || par[L.key].some(r =>
        L.fields.some(f => (f.req || f.type === 'column') && !fieldFilled(f, r[f.k]))));
    }
    if (type === 'exportOp') return !(par.dest && par.dest.trim());
    const defs = PARAM_DEFS[type];
    if (!Array.isArray(defs)) return false;
    return defs.some(f => (f.req || f.type === 'column') && !(par[f.k] && String(par[f.k]).trim()));
  }
  function nodeState(uid){
    const d = cards[uid];
    if (!d) return null;
    if (d.kind === 'dataset'){
      if (d.isOutput) return (d.capacity > 1 && d.filled < d.capacity) ? 'In attesa delle tabelle mancanti' : null;
      const p0 = (d.params && d.params[0]) || {};
      return (p0.path && p0.path.trim()) ? null : 'Origine da configurare';
    }
    if (inputsOf(uid).length < boxCapacity(d)) return 'Mancano tabelle in ingresso';
    ensureParams(d);
    const bad = d.components.findIndex((c, i) => stepMissing(c, d.params[i]));
    return bad >= 0 ? ('Da configurare: ' + META[d.components[bad]].label) : null;
  }
  let lastStateSweep = 0;
  function sweepStates(now){
    if (now - lastStateSweep < 220) return;
    lastStateSweep = now;
    Object.keys(cards).forEach(uid => {
      const el = cardEl(uid);
      if (!el) return;
      const msg = FEATURES.nodeStates ? nodeState(uid) : null;
      el.classList.toggle('warn', !!msg);
      const dot = el.querySelector('.state-dot');
      if (dot) dot.title = msg || '';
    });
  }
  // un collegamento trasporta dati solo se entrambi i capi sono pronti
  function linkLive(l){
    if (!FEATURES.flowGate) return true;
    return !nodeState(l.from) && !nodeState(l.to);
  }
  function smooth01(x){ x = Math.max(0, Math.min(1, x)); return x * x * (3 - 2 * x); }

  function animateBubbles(){
    drawLinks();
    const now = performance.now();
    sweepStates(now);
    renderMinimap();
    if (flowPaused){ bubblesG.innerHTML = ''; requestAnimationFrame(animateBubbles); return; }
    let out = '';
    linksArr.forEach((l, i) => {
      if (!linkLive(l)) return;
      const p = document.getElementById('lp-' + i);
      if (!p) return;
      const len = p.getTotalLength();
      if (!len) return;
      const st = linkState[l.from + '|' + l.to];
      // la pallina entra dalla porta di uscita e sparisce oltre quella di ingresso, poi riparte
      const cycle = len + BACK * 3.2;
      const sb = ((now - (st ? st.t0 : now)) * SPEED) % cycle - BACK * 1.1;
      const s0 = Math.max(0, sb - BACK * 3), s1 = Math.min(len, sb + FRONT * 3.2);
      if (s1 - s0 < 2) return;

      const n = Math.max(10, Math.ceil((s1 - s0) / 1.6));
      const pts = [];
      for (let k = 0; k <= n; k++){
        const sl = s0 + (s1 - s0) * k / n;
        const q = p.getPointAtLength(sl);
        pts.push({ x: q.x, y: q.y, s: sl });
      }
      const left = [], right = [];
      for (let k = 0; k <= n; k++){
        const a = pts[Math.max(0, k - 1)], b = pts[Math.min(n, k + 1)];
        let tx = b.x - a.x, ty = b.y - a.y;
        const tl = Math.hypot(tx, ty) || 1; tx /= tl; ty /= tl;
        const edge = smooth01(Math.min(pts[k].s, len - pts[k].s) / 22);   // emerge dalla porta, vi rientra
        const w = BASE_W / 2 + BALL * tubeProfile(pts[k].s - sb) * edge;
        left.push((pts[k].x - ty * w).toFixed(2) + ' ' + (pts[k].y + tx * w).toFixed(2));
        right.push((pts[k].x + ty * w).toFixed(2) + ' ' + (pts[k].y - tx * w).toFixed(2));
      }
      out += '<path d="M ' + left.concat(right.reverse()).join(' L ') + ' Z" fill="rgba(108,99,255,0.6)"/>';
    });
    bubblesG.innerHTML = out;
    requestAnimationFrame(animateBubbles);
  }

  // --- Nessun box puo restare sovrapposto a un altro ---
  const LABEL_H = 22;                       // l'etichetta sotto la card occupa spazio
  function applyPositions(animate, skipUid){
    Object.keys(cards).forEach(uid => {
      if (uid === skipUid) return;
      const el = cardEl(uid); if (!el) return;
      if (animate){
        el.style.transition = 'left .42s cubic-bezier(.32,.72,0,1), top .42s cubic-bezier(.32,.72,0,1)';
        setTimeout(() => { el.style.transition = ''; }, 440);
      }
      el.style.left = cards[uid].x + 'px';
      el.style.top = cards[uid].y + 'px';
    });
  }
  function clampCard(c){
    c.x = Math.max(6, Math.min(worldW() - CARD - 6, c.x));
    c.y = Math.max(6, Math.min(worldH() - CARD - LABEL_H - 6, c.y));
  }
  const GRID = 26;
  // due nodi che possono fondersi o collegarsi non si respingono: avvicinarli e l'azione
  function compatiblePair(idA, idB){
    const a = cards[idA], b = cards[idB];
    if (!a || !b) return false;
    if (a.kind === 'op' && b.kind === 'op') return true;
    if (a.kind === 'dataset' && b.kind === 'op') return linkRefusal(idA, idB) === null;
    if (a.kind === 'op' && b.kind === 'dataset') return linkRefusal(idB, idA) === null;
    return false;                                  // dataset con dataset: mai compatibili
  }
  function resolveOverlaps(fixedUid, skipApplyUid, snap, onlyIncompatible){
    if (layoutMode === 'grid'){ placeInSlots(true); return; }
    const PAD = 28;
    const minDX = CARD + PAD, minDY = CARD + LABEL_H + PAD;
    const ids = Object.keys(cards);
    for (let iter = 0; iter < 80; iter++){
      let moved = false;
      for (let i = 0; i < ids.length; i++){
        for (let j = i + 1; j < ids.length; j++){
          const A = cards[ids[i]], B = cards[ids[j]];
          if (!A || !B) continue;
          if (onlyIncompatible && compatiblePair(ids[i], ids[j])) continue;
          const dx = B.x - A.x, dy = B.y - A.y;
          const ox = minDX - Math.abs(dx), oy = minDY - Math.abs(dy);
          if (ox <= 0 || oy <= 0) continue;
          let sx = 0, sy = 0;
          // separa lungo l'asse dove la compenetrazione e minore, in proporzione
          if (ox / minDX < oy / minDY) sx = ((dx >= 0 ? 1 : -1) * ox) / 2 + 0.5;
          else sy = ((dy >= 0 ? 1 : -1) * oy) / 2 + 0.5;
          const aFixed = ids[i] === fixedUid, bFixed = ids[j] === fixedUid;
          if (aFixed){ B.x += sx * 2; B.y += sy * 2; }
          else if (bFixed){ A.x -= sx * 2; A.y -= sy * 2; }
          else { A.x -= sx; A.y -= sy; B.x += sx; B.y += sy; }
          clampCard(A); clampCard(B);     // il clamp va applicato dentro il ciclo
          moved = true;
        }
      }
      if (!moved) break;
    }
    ids.forEach(uid => {
      const c = cards[uid]; if (!c) return;
      if (snap && uid !== fixedUid){          // allineamento a griglia: il canvas resta ordinato
        c.x = Math.round(c.x / GRID) * GRID;
        c.y = Math.round(c.y / GRID) * GRID;
      }
      clampCard(c);
    });
    applyPositions(true, skipApplyUid);
  }

  function overlapsAny(x, y, ignoreUid){
    return Object.keys(cards).some(id => id !== ignoreUid && cards[id] &&
      Math.abs(cards[id].x - x) < CARD + 18 && Math.abs(cards[id].y - y) < CARD + LABEL_H + 18);
  }
  function freeSpot(x, y, ignoreUid){
    const maxX = worldW() - CARD - 6, maxY = worldH() - CARD - LABEL_H - 6;
    const cx = Math.max(6, Math.min(maxX, x)), cy = Math.max(6, Math.min(maxY, y));
    if (!overlapsAny(cx, cy, ignoreUid)) return { x: cx, y: cy };
    const dirs = [[1,0],[1,1],[0,1],[1,-1],[0,-1],[-1,1],[-1,0],[-1,-1]];
    for (let r = 1; r <= 8; r++){
      for (const [ox, oy] of dirs){
        const nx = Math.max(6, Math.min(maxX, cx + ox * r * (CARD + 26)));
        const ny = Math.max(6, Math.min(maxY, cy + oy * r * (CARD + LABEL_H + 22)));
        if (!overlapsAny(nx, ny, ignoreUid)) return { x: nx, y: ny };
      }
    }
    return { x: cx, y: cy };
  }

  // Un join ha bisogno di due tabelle: la capienza del box cresce con i join che contiene
  const MERGE_OPS = ['join', 'union'];
  function boxCapacity(box){ return 1 + box.components.filter(c => MERGE_OPS.includes(c)).length; }
  function inputsOf(boxUid){ return linksArr.filter(l => l.to === boxUid); }
  function outputOf(boxUid){ const l = linksArr.find(l => l.from === boxUid); return l ? l.to : null; }

  function renderOutputIcon(uid){
    const d = cards[uid], el = cardEl(uid);
    if (!d || !el) return;
    const wrap = el.querySelector('.icon-wrap');
    const incomplete = d.capacity > 1 && d.filled < d.capacity;
    if (incomplete){
      const n = d.capacity, f = d.filled;
      wrap.className = 'icon-wrap split';
      wrap.setAttribute('data-n', String(n));
      wrap.innerHTML = Array.from({ length: n }, (_, i) => i < f
        ? '<div class="half full">' + svgTag('dataset') + '</div>'
        : '<div class="half empty">' + svgTag('empty') + '</div>').join('') + NODE_EXTRAS;
      el.classList.add('partial');
    } else {
      wrap.removeAttribute('data-n');
      wrap.className = 'icon-wrap count-1';
      wrap.innerHTML = svgTag('dataset') + NODE_EXTRAS;
      el.classList.remove('partial');
    }
  }

  // se un box perde il join, perde anche il diritto alla seconda tabella
  function enforceCapacity(boxUid){
    const box = cards[boxUid];
    if (!box || box.kind !== 'op') return;
    const cap = boxCapacity(box);
    const ins = linksArr.filter(l => l.to === boxUid);
    if (ins.length > cap){
      const surplus = ins.slice(cap);          // restano i collegamenti piu vecchi
      linksArr = linksArr.filter(l => !surplus.includes(l));
    }
    pruneOutputs();
  }
  // porta il canvas in pari con lo stato: cio che non esiste piu se ne va con un pop
  function reconcileDom(){
    Array.from(stage.querySelectorAll('.card')).forEach(el => {
      if (!cards[el.dataset.uid]){
        el.classList.add('popping');
        setTimeout(() => el.remove(), 320);
      }
    });
  }

  function refreshOutput(boxUid){
    const box = cards[boxUid];
    if (!box || box.kind !== 'op') return;
    const n = inputsOf(boxUid).length;
    if (n === 0) return;
    let outUid = outputOf(boxUid);
    if (!outUid) outUid = spawnOutput(boxUid);
    const out = cards[outUid];
    if (!out) return;
    out.capacity = boxCapacity(box);
    out.filled = Math.min(n, out.capacity);
    renderOutputIcon(outUid);
    syncInspector();
  }

  // colloca un nodo nel primo spazio libero dopo l'ancora: destra, poi sopra, poi sotto
  function relocateAfter(anchorUid, uid){
    const a = cards[anchorUid], c = cards[uid];
    if (!a || !c) return false;
    const stepX = layoutMode === 'grid' ? SLOT_W : CARD + 78;
    const stepY = layoutMode === 'grid' ? SLOT_H : CARD + LABEL_H + 30;
    if (layoutMode === 'grid') computeSlots();
    for (let k = 1; k <= 5; k++){
      const cands = [
        { x: a.x + stepX * k, y: a.y },           // a destra
        { x: a.x + stepX * k, y: a.y - stepY },   // sopra
        { x: a.x + stepX * k, y: a.y + stepY }    // sotto
      ];
      for (const t of cands){
        if (layoutMode === 'grid'){
          const idx = nearestSlot(t.x, t.y, uid, false);
          if (idx < 0) continue;
          const sp = slots[idx];
          if (Math.abs(sp.x - t.x) > SLOT_W / 2 || Math.abs(sp.y - t.y) > SLOT_H / 2) continue;
          if (slotTaken(idx, uid)) continue;
          c.slot = idx; c.x = sp.x; c.y = sp.y;
          return true;
        } else {
          if (t.x < 6 || t.y < 6 || t.x > worldW() - CARD - 6 ||
              t.y > worldH() - CARD - LABEL_H - 6) continue;
          if (overlapsAny(t.x, t.y, uid)) continue;
          c.x = t.x; c.y = t.y;
          return true;
        }
      }
    }
    return false;
  }

  // prima postazione libera nel verso del flusso: a destra del box, sulla stessa riga se possibile
  function outputSlotFor(box){
    let best = -1, bestScore = Infinity;
    slots.forEach((sp, i) => {
      if (slotTaken(i)) return;
      const dx = sp.x - box.x, dy = Math.abs(sp.y - box.y);
      if (dx <= 0) return;                                  // mai a sinistra di chi lo produce
      const score = dy * 3 + Math.abs(dx - SLOT_W);          // ideale: la postazione subito accanto
      if (score < bestScore){ bestScore = score; best = i; }
    });
    if (best === -1){
      // a destra non c'e posto: la libera piu vicina, ovunque sia
      slots.forEach((sp, i) => {
        if (slotTaken(i)) return;
        const d = Math.hypot(sp.x - box.x, sp.y - box.y);
        if (d < bestScore){ bestScore = d; best = i; }
      });
    }
    return best;
  }

  // --- Generazione output con animazione di espulsione ---
  function spawnOutput(boxUid){
    const existing = outputOf(boxUid);
    if (existing) return existing;   // output gia presente
    const box = cards[boxUid];
    if (!box) return;
    outCounter++;
    const uid = 'out-' + (uidCounter++);
    // la destinazione si decide prima dell'espulsione: l'output vola dritto dove restera
    let pos, slot;
    if (layoutMode === 'grid'){
      computeSlots();
      slot = outputSlotFor(box);
      pos = slot >= 0 ? { x: slots[slot].x, y: slots[slot].y }
                      : freeSpot(Math.min(box.x + 200, worldW() - CARD - 12), box.y);
    } else {
      pos = freeSpot(Math.round(Math.min(box.x + 200, worldW() - CARD - 12) / GRID) * GRID, box.y);
    }
    cards[uid] = { kind:'dataset', isOutput:true, components:['dataset'], params:[defaultParams('dataset')],
                   name: 'Output ' + outCounter, x: pos.x, y: pos.y };
    if (slot !== undefined && slot >= 0) cards[uid].slot = slot;
    const el = createCardEl(uid);

    // parte dal centro del box e viene "sputato fuori"
    const dx = (box.x + CARD/2) - (pos.x + CARD/2);
    const dy = (box.y + CARD/2) - (pos.y + CARD/2);
    el.style.transition = 'none';
    el.style.transform = 'translate('+dx+'px,'+dy+'px) scale(.15)';
    el.style.opacity = '0';

    const boxWrap = cardEl(boxUid).querySelector('.icon-wrap');
    boxWrap.classList.remove('recoil'); void boxWrap.offsetWidth; boxWrap.classList.add('recoil');

    requestAnimationFrame(() => {
      el.style.transition = 'transform .55s cubic-bezier(.34,1.56,.64,1), opacity .25s ease';
      el.style.transform = 'translate(0,0) scale(1)';
      el.style.opacity = '1';
    });
    setTimeout(() => { el.style.transition=''; el.style.transform=''; }, 620);

    linksArr.push({ from: boxUid, to: uid });
    resolveOverlaps(uid, uid, true);   // gli altri si scansano, lui resta dove e stato espulso
    return uid;
  }

  function performMerge(draggedUid, targetUid){
    const dd = cards[draggedUid], dt = cards[targetUid];
    if (!dd || !dt || dd.kind !== 'op' || dt.kind !== 'op') return;
    ensureParams(dt); ensureParams(dd);
    const oldLen = dt.components.length;
    const merged = dt.components.concat(dd.components);
    const mergedParams = dt.params.concat(dd.params);
    let name = dt.components.length > 1 ? dt.name : null;
    if (!name){ comboCounter++; name = comboCounter===1 ? 'Combined Box' : 'Combined Box '+comboCounter; }

    linksArr.forEach(l => {
      const oldKey = l.from + '|' + l.to;
      if (l.to === draggedUid) l.to = targetUid;
      if (l.from === draggedUid) l.from = targetUid;
      const newKey = l.from + '|' + l.to;
      if (newKey !== oldKey && linkState[oldKey] && !linkState[newKey]) linkState[newKey] = linkState[oldKey];
    });
    linksArr = linksArr.filter(l => l.from !== l.to);
    // un dataset prodotto dal box fuso non puo anche alimentarlo: resta solo l'uscita
    const produced = new Set(linksArr.filter(l => l.from === targetUid).map(l => l.to));
    linksArr = linksArr.filter(l => !(l.to === targetUid && produced.has(l.from)));
    linksArr = linksArr.filter((l,i,arr) => arr.findIndex(o=>o.from===l.from && o.to===l.to)===i);

    cards[targetUid] = Object.assign({}, dt, { components: merged, params: mergedParams, name });
    delete cards[draggedUid];
    const de = cardEl(draggedUid); if (de) de.remove();

    const targetEl = cardEl(targetUid);
    const wrap = targetEl.querySelector('.icon-wrap');
    const oldRects = Array.from(wrap.querySelectorAll('svg')).map(n=>n.getBoundingClientRect());
    wrap.className = 'icon-wrap ' + countClass(merged.length);
    wrap.innerHTML = wrapInner(merged, true);
    targetEl.classList.add('combined');
    targetEl.querySelector('.label').textContent = name;

    Array.from(wrap.querySelectorAll('svg')).filter(n=>!n.closest('.expand-btn')).forEach((node,i)=>{
      if (i < oldRects.length){
        const nr = node.getBoundingClientRect();
        node.style.transition='none';
        node.style.transform='translate('+((oldRects[i].left-nr.left)/view.z)+'px,'+((oldRects[i].top-nr.top)/view.z)+'px) scale('+(oldRects[i].width/(nr.width||1))+')';
        requestAnimationFrame(()=>{ node.style.transition='transform .4s cubic-bezier(.34,1.56,.64,1)'; node.style.transform='translate(0,0) scale(1)'; });
      } else { node.classList.add('arriving'); node.style.animationDelay = ((i-oldRects.length)*90)+'ms'; }
    });
    wrap.classList.remove('fusing'); void wrap.offsetWidth; wrap.classList.add('fusing');

    const burst = document.createElement('div');
    burst.className='burst';
    burst.style.left=(cards[targetUid].x + CARD/2)+'px';
    burst.style.top=(cards[targetUid].y + CARD/2)+'px';
    world.appendChild(burst);
    burst.addEventListener('animationend',()=>burst.remove());
    drawLinks();
    const ins = inputsOf(targetUid);
    if (ins.length){
      relocateAfter(ins[0].from, targetUid);
      // gli output restano a valle: si spostano solo se ora starebbero alle spalle del box o sopra altri nodi
      linksArr.filter(l => l.from === targetUid).forEach(l => {
        const o = cards[l.to], b = cards[targetUid];
        if (o && (o.x <= b.x || overlapsAny(o.x, o.y, l.to))) relocateAfter(targetUid, l.to);
      });
      applyPositions(true);
    }
    if (selectedUid === draggedUid) selectedUid = targetUid;
    pendingStepAnim = { start: oldLen, end: merged.length - 1 };
    syncInspector();
    if (inputsOf(targetUid).length) setTimeout(()=>refreshOutput(targetUid), 300);
  }

  // --- Inserimento di una lavorazione dentro un collegamento esistente ---
```


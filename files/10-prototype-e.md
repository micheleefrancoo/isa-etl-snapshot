# 10-prototype-e.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 5/6)

5098 righe totali

```html
          r.style.transition = 'transform .2s cubic-bezier(.32,.72,0,1)';
          r.style.transform = 'translateY('+sh+'px)';
        });
      }
      function onMove(ev){
        const dy = ev.clientY - startY;
        if (!dragging){
          if (Math.abs(dy) < 4) return;
          dragging = true; row.classList.add('drag-row'); row.style.transition = 'none';
        }
        row.style.transform = 'translateY('+dy+'px)';
        let idx = Math.max(0, Math.min(rows.length - 1, Math.round(startIndex + dy / stepH)));
        if (idx !== newIndex){ newIndex = idx; shifts(); }
      }
      function onUp(){
        window.removeEventListener('pointermove', onMove);
        window.removeEventListener('pointerup', onUp);
        rows.forEach(r => { r.style.transition = ''; r.style.transform = ''; });
        row.classList.remove('drag-row');
        if (!dragging){                       // un click semplice seleziona il passaggio
          selectedStep = startIndex; openCond = 0; openKey = 0; openRows = {}; renderInspector(); return;
        }
        if (newIndex !== startIndex){
          pushHistory();
          const d = cards[selectedUid];
          ensureParams(d);
          const [mc] = d.components.splice(startIndex, 1); d.components.splice(newIndex, 0, mc);
          const [mp] = d.params.splice(startIndex, 1);     d.params.splice(newIndex, 0, mp);
          const wrap = cardEl(selectedUid).querySelector('.icon-wrap');
          wrap.innerHTML = wrapInner(d.components, true);
          selectedStep = newIndex;
        }
        openCond = 0;
        renderInspector();
      }
      window.addEventListener('pointermove', onMove);
      window.addEventListener('pointerup', onUp);
    });
  }

  function bindInspector(){
    const close = document.getElementById('inspClose');
    if (close) close.addEventListener('click', deselect);
    const coll = document.getElementById('inspCollapse');
    if (coll) coll.addEventListener('click', () => setCollapsed(true));
    const nameEl = document.getElementById('inspName');
    if (nameEl){
      nameEl.addEventListener('blur', () => {
        const v = nameEl.textContent.trim() || cards[selectedUid].name;
        nameEl.textContent = v;
        cards[selectedUid].name = v;
        const le = cardEl(selectedUid) && cardEl(selectedUid).querySelector('.label');
        if (le) le.textContent = v;
      });
      nameEl.addEventListener('keydown', e => { if (e.key === 'Enter'){ e.preventDefault(); nameEl.blur(); } });
    }
    bindStepList();
    inspInner.querySelectorAll('input[data-key]').forEach(el => {
      el.addEventListener('input', () => {
        const step = parseInt(el.dataset.step, 10);
        const d = cards[selectedUid];
        if (!d) return;
        ensureParams(d);
        d.params[step] = d.params[step] || {};
        d.params[step][el.dataset.key] = el.value;
      });
    });
    bindSelects((sel, value) => {
      const d = cards[selectedUid];
      if (!d) return;
      if (sel.dataset.key !== undefined){
        const step = parseInt(sel.dataset.step, 10);
        ensureParams(d);
        d.params[step] = d.params[step] || {};
        d.params[step][sel.dataset.key] = value;
        const lbl = sel.querySelector('.sel-val'); if (lbl) lbl.textContent = value;
        sel.querySelectorAll('.sel-menu button').forEach(b => b.classList.toggle('on', b.dataset.opt === value));
        return;
      }
      if (sel.dataset.cconn !== undefined && activeFilterPar){
        activeFilterPar.conditions[parseInt(sel.dataset.cconn, 10)].conn = value; renderInspector(); return;
      }
      if (sel.dataset.kconn !== undefined && activeJoinPar){
        activeJoinPar.keys[parseInt(sel.dataset.kconn, 10)].conn = value; renderInspector(); return;
      }
      if (sel.dataset.ml !== undefined){ multiSelectChange(sel, value); return; }
      if (sel.dataset.jk !== undefined){ joinSelectChange(sel, value); return; }
      if (sel.dataset.logic !== undefined || sel.dataset.f !== undefined) filterSelectChange(sel, value);
    });
  }

  // --- Due modi di disporre i nodi ---
  let layoutMode = 'free';            // 'free' | 'grid'
  const SLOT_W = 128, SLOT_H = 142, SLOT_M = 14;
  let slots = [];
  function computeSlots(){
    const cols = Math.max(1, Math.floor((worldW() - SLOT_M * 2) / SLOT_W));
    const rows = Math.max(1, Math.floor((worldH() - SLOT_M * 2) / SLOT_H));
    slots = [];
    for (let r = 0; r < rows; r++)
      for (let c = 0; c < cols; c++)
        slots.push({ x: SLOT_M + c * SLOT_W + (SLOT_W - CARD) / 2, y: SLOT_M + r * SLOT_H });
  }
  function slotTaken(idx, exceptUid){
    return Object.keys(cards).some(id => id !== exceptUid && cards[id].slot === idx);
  }
  function nearestSlot(x, y, exceptUid, freeOnly){
    let best = -1, bestD = Infinity;
    slots.forEach((sp, i) => {
      if (freeOnly && slotTaken(i, exceptUid)) return;
      const d = Math.hypot(sp.x - x, sp.y - y);
      if (d < bestD){ bestD = d; best = i; }
    });
    return best;
  }
  function firstFreeSlot(nearX, nearY, exceptUid){
    const i = nearestSlot(nearX, nearY, exceptUid, true);
    return i >= 0 ? i : nearestSlot(nearX, nearY, exceptUid, false);
  }
  function placeInSlots(animate){
    Object.keys(cards).forEach(uid => {
      const c = cards[uid];
      if (c.slot === undefined || !slots[c.slot]) return;
      c.x = slots[c.slot].x; c.y = slots[c.slot].y;
    });
    applyPositions(animate);
    drawLinks();
  }
  function assignSlots(){
    computeSlots();
    Object.keys(cards).forEach(uid => { cards[uid].slot = undefined; });
    // chi sta piu in alto a sinistra sceglie per primo: l'ordine visivo resta riconoscibile
    const order = Object.keys(cards).sort((A, B) =>
      (cards[A].y - cards[B].y) || (cards[A].x - cards[B].x));
    order.forEach(uid => { cards[uid].slot = firstFreeSlot(cards[uid].x, cards[uid].y, uid); });
    placeInSlots(true);
  }
  function setMode(mode){
    if (mode === layoutMode) return;
    pushHistory();
    layoutMode = mode;
    Array.from(modeSeg.querySelectorAll('button')).forEach(b =>
      b.classList.toggle('on', b.dataset.mode === mode));
    if (mode === 'grid'){
      assignSlots();
      hint.textContent = 'Modalita organizzata: i nodi occupano postazioni fisse e si scambiano di posto';
    } else {
      Object.keys(cards).forEach(uid => { cards[uid].slot = undefined; });
      hint.textContent = 'Modalita libera: disponi i nodi come preferisci';
    }
    setTimeout(() => { hint.textContent = DEFAULT_HINT; }, 2200);
  }
  modeSeg.addEventListener('click', (e) => {
    const b = e.target.closest('button[data-mode]');
    if (b) setMode(b.dataset.mode);
  });

  // --- Collegamento dalle porte: si tira un cavo da un nodo verso un altro ---
  const tempLink = document.getElementById('tempLink');
  stage.addEventListener('pointerdown', (e) => {
    const port = e.target.closest('.port');
    if (!port || !FEATURES.portDrag) return;
    e.preventDefault(); e.stopPropagation();
    const card = port.closest('.card');
    const srcUid = card.dataset.uid;
    const c = cards[srcUid];
    const side = port.dataset.port;
    const start = {
      x: c.x + (side === 'r' ? CARD : side === 'l' ? 0 : CARD / 2),
      y: c.y + (side === 'b' ? CARD : side === 't' ? 0 : CARD / 2)
    };
    let target = null, rel = null;
    card.classList.add('dragging');
    flowPaused = true;

    const mv = (ev) => {
      const p = toWorld(ev.clientX, ev.clientY);
      const under = document.elementFromPoint(ev.clientX, ev.clientY);
      const tc = under ? under.closest('.card') : null;
      let r = (tc && tc !== card) ? relation(srcUid, tc.dataset.uid) : null;
      if (r === 'merge' || r === 'displace') r = null;       // dalle porte si collega soltanto
      if (tc !== target){
        if (target) target.classList.remove('link-target');
        target = r ? tc : null;
        if (target) target.classList.add('link-target');
      }
      rel = r;
      hint.textContent = r ? 'Rilascia per collegare' :
        (tc && tc !== card ? (displaceReason || 'Questi due nodi non si possono collegare') : 'Trascina fino al nodo da collegare');
      const ok = !!r;
      tempLink.innerHTML = '<path d="M '+start.x+' '+start.y+' L '+p.x+' '+p.y+'" fill="none" stroke="'+(ok ? '#6C63FF' : 'rgba(108,99,255,0.5)')+
        '" stroke-width="2.2" stroke-dasharray="6 5" stroke-linecap="round"/>'+
        '<circle cx="'+p.x+'" cy="'+p.y+'" r="4" fill="'+(ok ? '#6C63FF' : 'rgba(108,99,255,0.5)')+'"/>';
    };
    const up = () => {
      window.removeEventListener('pointermove', mv); window.removeEventListener('pointerup', up);
      tempLink.innerHTML = '';
      card.classList.remove('dragging');
      flowPaused = false;
      hint.textContent = DEFAULT_HINT;
      if (target) target.classList.remove('link-target');
      if (target && rel){
        pushHistory();
        const tUid = target.dataset.uid;
        if (rel === 'link') connect(srcUid, tUid);
        else if (rel === 'link-reverse') connect(tUid, srcUid);
      }
    };
    window.addEventListener('pointermove', mv); window.addEventListener('pointerup', up);
  }, true);

  // --- Navigazione e selezione sul vuoto ---
  let suppressClick = false;
  const marqueeEl = document.getElementById('marquee');
  const minimapEl = document.getElementById('minimap');
  const mmView = document.getElementById('mmView');

  function isBackground(t){
    return !t.closest('.card') && !t.closest('.zoom-ctl') && !t.closest('.minimap') && !t.closest('.hit');
  }
  stage.addEventListener('pointerdown', (e) => {
    const t = e.target;
    const wantPan = FEATURES.panZoom && (spaceDown || e.button === 1 || (!FEATURES.multiSelect && isBackground(t)));
    if (wantPan && (isBackground(t) || spaceDown || e.button === 1)){
      // spostamento della vista
      e.preventDefault();
      const sx = e.clientX, sy = e.clientY, ox = view.x, oy = view.y;
      let moved = false;
      stage.style.cursor = 'grabbing';
      const mv = ev => {
        if (Math.hypot(ev.clientX - sx, ev.clientY - sy) > 3) moved = true;
        view.x = ox + (ev.clientX - sx); view.y = oy + (ev.clientY - sy); applyView(false);
      };
      const up = () => {
        window.removeEventListener('pointermove', mv); window.removeEventListener('pointerup', up);
        stage.style.cursor = spaceDown ? 'grab' : '';
        if (moved) suppressClick = true;
      };
      window.addEventListener('pointermove', mv); window.addEventListener('pointerup', up);
      return;
    }
    if (!FEATURES.multiSelect || e.button !== 0 || !isBackground(t)) return;
    // selezione a riquadro
    const sr = stage.getBoundingClientRect();
    const sx = e.clientX, sy = e.clientY;
    let moved = false;
    const additive = e.shiftKey;
    const base = additive ? new Set(selectedSet) : new Set();
    const mv = ev => {
      if (!moved && Math.hypot(ev.clientX - sx, ev.clientY - sy) < 4) return;
      moved = true;
      const x1 = Math.min(sx, ev.clientX) - sr.left, y1 = Math.min(sy, ev.clientY) - sr.top;
      const x2 = Math.max(sx, ev.clientX) - sr.left, y2 = Math.max(sy, ev.clientY) - sr.top;
      Object.assign(marqueeEl.style, { display:'block', left:x1+'px', top:y1+'px', width:(x2-x1)+'px', height:(y2-y1)+'px' });
      // il riquadro e in coordinate di schermo: i nodi vanno riportati allo stesso sistema
      const hit = new Set(base);
      Object.keys(cards).forEach(id => {
        const c = cards[id];
        const cx1 = c.x * view.z + view.x, cy1 = c.y * view.z + view.y;
        const cx2 = cx1 + CARD * view.z, cy2 = cy1 + CARD * view.z;
        if (cx2 > x1 && cx1 < x2 && cy2 > y1 && cy1 < y2) hit.add(id);
      });
      selectedSet.clear(); hit.forEach(id => selectedSet.add(id));
      if (!selectedSet.has(selectedUid)) selectedUid = selectedSet.size ? [...selectedSet][0] : null;
      paintSelection();
    };
    const up = () => {
      window.removeEventListener('pointermove', mv); window.removeEventListener('pointerup', up);
      marqueeEl.style.display = 'none';
      if (!moved) return;
      suppressClick = true;
      const ids = [...selectedSet];
      if (ids.length) setSelection(ids); else deselect();
    };
    window.addEventListener('pointermove', mv); window.addEventListener('pointerup', up);
  });

  // rotella: due dita spostano la vista, Cmd/Ctrl (o il pinch) la ingrandisce attorno al puntatore
  stage.addEventListener('wheel', (e) => {
    if (!FEATURES.panZoom) return;
    e.preventDefault();
    if (e.ctrlKey || e.metaKey){
      const r = stage.getBoundingClientRect();
      zoomAt(e.clientX - r.left, e.clientY - r.top, view.z * Math.exp(-e.deltaY * 0.0022), false);
    } else {
      view.x -= e.deltaX; view.y -= e.deltaY; applyView(false);
    }
  }, { passive: false });

  function zoomAt(px, py, z, ease){
    const nz = Math.max(0.35, Math.min(2, z));
    view.x = px - (px - view.x) * (nz / view.z);
    view.y = py - (py - view.y) * (nz / view.z);
    view.z = nz;
    applyView(ease);
  }
  function fitView(ease){
    const ids = Object.keys(cards);
    if (!ids.length){ view.x = 0; view.y = 0; view.z = 1; applyView(ease); return; }
    let x1 = Infinity, y1 = Infinity, x2 = -Infinity, y2 = -Infinity;
    ids.forEach(id => {
      const c = cards[id];
      x1 = Math.min(x1, c.x); y1 = Math.min(y1, c.y);
      x2 = Math.max(x2, c.x + CARD); y2 = Math.max(y2, c.y + CARD + LABEL_H);
    });
    const pad = 48, W = stage.clientWidth, H = stage.clientHeight;
    const z = Math.max(0.35, Math.min(1.25, Math.min(W / (x2 - x1 + pad * 2), H / (y2 - y1 + pad * 2))));
    view.z = z;
    view.x = (W - (x2 - x1) * z) / 2 - x1 * z;
    view.y = (H - (y2 - y1) * z) / 2 - y1 * z;
    applyView(ease);
  }
  document.getElementById('zoomIn').addEventListener('click', () => zoomAt(stage.clientWidth / 2, stage.clientHeight / 2, view.z * 1.2, true));
  document.getElementById('zoomOut').addEventListener('click', () => zoomAt(stage.clientWidth / 2, stage.clientHeight / 2, view.z / 1.2, true));
  document.getElementById('zoomPct').addEventListener('click', () => zoomAt(stage.clientWidth / 2, stage.clientHeight / 2, 1, true));
  document.getElementById('zoomFit').addEventListener('click', () => fitView(true));

  // minimappa: il flusso in miniatura e la porzione visibile
  let mmFrame = null;
  function renderMinimap(){
    if (!(FEATURES.panZoom && FEATURES.minimap)) return;
    const ids = Object.keys(cards);
    const W = stage.clientWidth, H = stage.clientHeight;
    const vx1 = -view.x / view.z, vy1 = -view.y / view.z, vx2 = vx1 + W / view.z, vy2 = vy1 + H / view.z;
    let x1 = vx1, y1 = vy1, x2 = vx2, y2 = vy2;
    ids.forEach(id => { const c = cards[id];
      x1 = Math.min(x1, c.x); y1 = Math.min(y1, c.y); x2 = Math.max(x2, c.x + CARD); y2 = Math.max(y2, c.y + CARD); });
    const pad = 20; x1 -= pad; y1 -= pad; x2 += pad; y2 += pad;
    const mw = 168, mh = 104;
    const k = Math.min(mw / (x2 - x1), mh / (y2 - y1));
    const ox = (mw - (x2 - x1) * k) / 2, oy = (mh - (y2 - y1) * k) / 2;
    mmFrame = { x1, y1, k, ox, oy };
    let html = '';
    ids.forEach(id => { const c = cards[id];
      html += '<div class="mm-node'+(c.kind==='dataset'?' ds':'')+'" style="left:'+(ox+(c.x-x1)*k)+'px;top:'+(oy+(c.y-y1)*k)+'px;width:'+Math.max(3,CARD*k)+'px;height:'+Math.max(3,CARD*k)+'px"></div>'; });
    minimapEl.innerHTML = html + '<div class="mm-view" id="mmView" style="left:'+(ox+(vx1-x1)*k)+'px;top:'+(oy+(vy1-y1)*k)+'px;width:'+((vx2-vx1)*k)+'px;height:'+((vy2-vy1)*k)+'px"></div>';
  }
  minimapEl.addEventListener('pointerdown', (e) => {
    e.stopPropagation();
    const go = ev => {
      if (!mmFrame) return;
      const r = minimapEl.getBoundingClientRect();
      const wx = mmFrame.x1 + (ev.clientX - r.left - mmFrame.ox) / mmFrame.k;
      const wy = mmFrame.y1 + (ev.clientY - r.top - mmFrame.oy) / mmFrame.k;
      view.x = stage.clientWidth / 2 - wx * view.z;
      view.y = stage.clientHeight / 2 - wy * view.z;
      applyView(false);
    };
    go(e);
    const up = () => { window.removeEventListener('pointermove', go); window.removeEventListener('pointerup', up); };
    window.addEventListener('pointermove', go); window.addEventListener('pointerup', up);
  });

  // barra spaziatrice: navigazione temporanea
  document.addEventListener('keydown', (e) => {
    if (e.code === 'Space' && FEATURES.panZoom && !isTyping(e.target)){
      if (!spaceDown){ spaceDown = true; stage.style.cursor = 'grab'; }
      e.preventDefault();
    }
  });
  document.addEventListener('keyup', (e) => {
    if (e.code === 'Space'){ spaceDown = false; stage.style.cursor = ''; }
  });
  function isTyping(t){
    return t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable);
  }

  // --- Pannello funzionalita: ogni novita si prova e si spegne in isolamento ---
  const featPanel = document.getElementById('featPanel');
  const featBtn = document.getElementById('featBtn');
  function buildFeaturePanel(){
    featPanel.innerHTML =
      '<div class="feat-title">Funzionalità</div>'+
      '<div class="feat-sub">Accendi o spegni ciascuna novità per valutarla da sola. Lo stato precedente del prototipo è conservato a parte.</div>'+
      FEATURE_INFO.map(([k, name, desc]) =>
        '<label class="feat-row"><input type="checkbox" data-feat="'+k+'"'+(FEATURES[k] ? ' checked' : '')+'>'+
        '<span><div class="feat-name">'+name+'</div><div class="feat-desc">'+desc+'</div></span></label>').join('');
    featPanel.querySelectorAll('input[data-feat]').forEach(inp =>
      inp.addEventListener('change', () => { FEATURES[inp.dataset.feat] = inp.checked; applyFeatures(inp.dataset.feat); }));
  }
  function applyFeatures(changed){
    Object.keys(FEATURES).forEach(k => stage.classList.toggle('f-' + k, !!FEATURES[k]));
    if (FEATURES.panZoom){
      world.style.width = WORLD_W + 'px'; world.style.height = WORLD_H + 'px';
      applyView(false);
    } else {
      world.style.width = '100%'; world.style.height = '100%';
      view.x = 0; view.y = 0; view.z = 1; applyView(false);
      if (changed === 'panZoom'){
        // si torna alla finestra fissa: nessun nodo deve restarne fuori
        if (layoutMode === 'grid'){ computeSlots(); assignSlotsKeepingOrder(); }
        else { Object.keys(cards).forEach(uid => clampCard(cards[uid])); if (anyOverlap()) resolveOverlaps(null, null, true); else applyPositions(true); }
      }
    }
    if (changed === 'panZoom' && FEATURES.panZoom && layoutMode === 'grid'){ computeSlots(); assignSlotsKeepingOrder(); }
    if (!FEATURES.multiSelect && selectedSet.size > 1){ const keep = selectedUid; if (keep) selectCard(keep); else deselect(); }
    if (!FEATURES.minimap || !FEATURES.panZoom) minimapEl.innerHTML = '';
    lastStateSweep = 0;
    drawLinks();
  }
  featBtn.addEventListener('click', (e) => { e.stopPropagation(); featPanel.classList.toggle('open'); });
  featPanel.addEventListener('click', (e) => e.stopPropagation());
  document.addEventListener('click', () => featPanel.classList.remove('open'));

  // --- Riordino automatico: la disposizione deriva dal flusso, non dai gesti ---
  function autoLayout(){
    const all = Object.keys(cards);
    if (!all.length) return;
    pushHistory();
    // un nodo senza collegamenti non appartiene al flusso: non deve occuparne le colonne
    const isolated = all.filter(id => !linksArr.some(l => l.from === id || l.to === id));
    const ids = all.filter(id => !isolated.includes(id));

    // 1. profondita = percorso piu lungo dalle sorgenti: definisce la colonna
    const depth = {};
    ids.forEach(id => { depth[id] = 0; });
    let changed = true, guard = 0;
    while (changed && guard++ < ids.length + 6){
      changed = false;
      linksArr.forEach(l => {
        if (cards[l.from] && cards[l.to] && depth[l.to] < depth[l.from] + 1){
          depth[l.to] = depth[l.from] + 1; changed = true;
        }
      });
    }
    // una sorgente non deve restare in prima colonna se chi la consuma e lontano:
    // la si avvicina, mettendola appena prima del suo consumatore piu vicino
    for (let pass = 0; pass < 3; pass++){
      ids.forEach(id => {
        const hasInput = linksArr.some(l => l.to === id);
        if (hasInput) return;
        const succ = linksArr.filter(l => l.from === id).map(l => depth[l.to]);
        if (!succ.length) return;
        depth[id] = Math.max(0, Math.min.apply(null, succ) - 1);
      });
    }
    const minD = Math.min.apply(null, ids.map(id => depth[id]));
    ids.forEach(id => { depth[id] -= minD; });

    const layers = [];
    ids.forEach(id => { (layers[depth[id]] = layers[depth[id]] || []).push(id); });
    const cols = layers.filter(Boolean);
    if (isolated.length) cols.push(isolated);   // colonna di parcheggio a destra del flusso
    if (!cols.length) return;

    // 2. ordine dentro la colonna: baricentro dei vicini, per ridurre gli incroci
    const posIn = {};
    const reindex = () => cols.forEach(layer => layer.forEach((id, i) => { posIn[id] = i; }));
    cols.forEach(layer => layer.sort((a, b) => cards[a].y - cards[b].y));
    reindex();
    for (let it = 0; it < 6; it++){
      cols.forEach((layer, li) => {
        const bary = {};
        layer.forEach(id => {
          const nb = [];
          linksArr.forEach(l => {
            if (l.to === id && posIn[l.from] !== undefined) nb.push(posIn[l.from]);
            if (l.from === id && posIn[l.to] !== undefined) nb.push(posIn[l.to]);
          });
          bary[id] = nb.length ? nb.reduce((a, b) => a + b, 0) / nb.length : posIn[id];
        });
        layer.sort((a, b) => bary[a] - bary[b]);
        reindex();
      });
    }

    // 3. griglia di partenza: colonne equidistanti, ogni colonna centrata
    const COL = CARD + 78;
    const maxCount = Math.max.apply(null, cols.map(c => c.length));
    const avail = stage.clientHeight - 16 - (CARD + LABEL_H);
    const ROW = maxCount > 1
      ? Math.max(CARD + LABEL_H + 18, Math.min(CARD + LABEL_H + 28, avail / (maxCount - 1)))
      : CARD + LABEL_H + 28;
    const totalW = (cols.length - 1) * COL + CARD;
    const x0 = Math.max(12, (stage.clientWidth - totalW) / 2);
    cols.forEach((layer, li) => {
      const h = (layer.length - 1) * ROW + CARD + LABEL_H;
      const y0 = Math.max(8, (stage.clientHeight - h) / 2);
      layer.forEach((id, i) => { cards[id].x = x0 + li * COL; cards[id].y = y0 + i * ROW; });
    });

    // 4. simmetria: ogni nodo tende al baricentro dei suoi vicini, senza sovrapporsi
    for (let pass = 0; pass < 5; pass++){
      cols.forEach(layer => {
        layer.forEach(id => {
          const nb = [];
          linksArr.forEach(l => {
            if (l.to === id && cards[l.from]) nb.push(cards[l.from].y);
            if (l.from === id && cards[l.to]) nb.push(cards[l.to].y);
          });
          if (nb.length) cards[id].y = (cards[id].y + nb.reduce((a, b) => a + b, 0) / nb.length) / 2;
        });
        layer.sort((a, b) => cards[a].y - cards[b].y);
        for (let i = 1; i < layer.length; i++){
          const prev = cards[layer[i-1]], cur = cards[layer[i]];
          if (cur.y - prev.y < ROW) cur.y = prev.y + ROW;
        }
        const first = cards[layer[0]].y;
        const last = cards[layer[layer.length - 1]].y;
        const off = (stage.clientHeight - (last - first + CARD + LABEL_H)) / 2 - first;
        layer.forEach(id => { cards[id].y += off; });
      });
    }

    all.forEach(id => { cards[id].x = Math.round(cards[id].x); cards[id].y = Math.round(cards[id].y); clampCard(cards[id]); });

    if (layoutMode === 'grid'){ computeSlots(); assignSlotsKeepingOrder(); }
    else if (anyOverlap()) resolveOverlaps(null, null, false);   // separa senza riallineare alla griglia
    else applyPositions(true);
    drawLinks();
    syncInspector();
    if (FEATURES.panZoom) setTimeout(() => fitView(true), 60);
    hint.textContent = 'Disposizione riordinata secondo il flusso';
    setTimeout(() => { if (hint.textContent === 'Disposizione riordinata secondo il flusso') hint.textContent = DEFAULT_HINT; }, 1800);
  }
  autoBtn.addEventListener('click', autoLayout);

  // --- Cronologia: ogni azione e annullabile ---
  let history = [];
  let redoStack = [];                  // azioni annullate, ripristinabili
  const HIST_MAX = 50;
  function snapshot(){
    return JSON.stringify({ cards, linksArr, comboCounter, outCounter, uidCounter });
  }
  function updateHistoryButtons(){
    undoBtn.disabled = history.length === 0;
    redoBtn.disabled = redoStack.length === 0;
  }
  function pushHistory(){
    history.push(snapshot());
    if (history.length > HIST_MAX) history.shift();
    redoStack = [];                    // una nuova azione chiude il ramo delle azioni annullate
    updateHistoryButtons();
  }
  // aggiorna il contenuto di una card esistente senza ricrearla: cosi puo scivolare
  function refreshCardEl(id){
    const el = cardEl(id);
    if (!el) return;
    const tmp = createCardEl(id);
    el.className = tmp.className;
    el.innerHTML = tmp.innerHTML;
    tmp.remove();
    if (cards[id].isOutput) renderOutputIcon(id);
  }

  function restoreSnapshot(st, message){
    const before = new Set(Object.keys(cards));
    cards = st.cards; linksArr = st.linksArr;
    comboCounter = st.comboCounter; outCounter = st.outCounter; uidCounter = st.uidCounter;

    before.forEach(id => {
      if (cards[id]) return;
      const el = cardEl(id);
      if (el){ el.classList.add('popping'); setTimeout(() => el.remove(), 320); }
    });
    Object.keys(cards).forEach(id => {
      if (before.has(id) && cardEl(id)){ refreshCardEl(id); return; }
      const el = createCardEl(id);
      el.style.transition = 'none'; el.style.transform = 'scale(.5)'; el.style.opacity = '0';
      requestAnimationFrame(() => {
        el.style.transition = 'transform .34s cubic-bezier(.34,1.56,.64,1), opacity .2s';
        el.style.transform = 'scale(1)'; el.style.opacity = '1';
      });
      setTimeout(() => { el.style.transition = ''; el.style.transform = ''; }, 360);
    });
    applyPositions(true);
    Object.keys(cards).forEach(uid => { if (cards[uid].kind === 'op') refreshOutput(uid); });
    syncInspector();
    drawLinks();
    updateHistoryButtons();
    hint.textContent = message;
    setTimeout(() => { if (hint.textContent === message) hint.textContent = DEFAULT_HINT; }, 1400);
  }

  function undo(){
    const snap = history.pop();
    if (!snap) return;
    redoStack.push(snapshot());
    restoreSnapshot(JSON.parse(snap), 'Ultima azione annullata');
  }
  function redo(){
    const snap = redoStack.pop();
    if (!snap) return;
    history.push(snapshot());
    if (history.length > HIST_MAX) history.shift();
    restoreSnapshot(JSON.parse(snap), 'Azione ripristinata');
  }

  function renderAll(){
    Array.from(stage.querySelectorAll('.card')).forEach(c => c.remove());
    Object.keys(linkState).forEach(k => delete linkState[k]);
    Object.keys(cards).forEach(createCardEl);
    Object.keys(cards).forEach(uid => { if (cards[uid].kind === 'op') refreshOutput(uid); });
    if (selectedUid && !cards[selectedUid]) deselect(); else if (selectedUid){ const el=cardEl(selectedUid); if (el) el.classList.add('selected'); renderInspector(); }
    drawLinks();
  }

  // --- Eliminazione di nodi e collegamenti ---
  // un output senza produttore, o il cui box ha perso gli ingressi, non ha ragione di esistere
  function pruneOutputs(){
    let changed = true;
    while (changed){
      changed = false;
      for (const id of Object.keys(cards)){
        const c = cards[id];
        if (!c || !c.isOutput) continue;
        const prod = linksArr.find(l => l.to === id);
        const alive = prod && cards[prod.from] && linksArr.some(l => l.to === prod.from);
        if (!alive){
          delete cards[id];
          linksArr = linksArr.filter(l => l.from !== id && l.to !== id);
          changed = true;
        }
      }
    }
  }
  // quali nodi sparirebbero davvero: il nodo piu gli output che restano senza produttore
  function nodesRemovedBy(uidOrIds){
    const ids = Array.isArray(uidOrIds) ? uidOrIds : [uidOrIds];
    const c = Object.assign({}, cards); ids.forEach(id => delete c[id]);
    let l = linksArr.filter(x => !ids.includes(x.from) && !ids.includes(x.to));
    let changed = true;
    while (changed){
      changed = false;
      for (const id of Object.keys(c)){
        if (!c[id].isOutput) continue;
        const prod = l.find(x => x.to === id);
        const alive = prod && c[prod.from] && l.some(x => x.to === prod.from);
        if (!alive){
          delete c[id];
          l = l.filter(x => x.from !== id && x.to !== id);
          changed = true;
        }
      }
    }
    const removed = new Set(ids);
    Object.keys(cards).forEach(id => { if (!c[id]) removed.add(id); });
    return removed;
  }

  // l'output rientra nel box ripercorrendo la strada da cui era uscito
  function retract(ids, targets, dur, done){
    const start = performance.now();
    const from = {};
    ids.forEach(id => { from[id] = { x: cards[id].x, y: cards[id].y }; });
    function step(now){
      const t = Math.min(1, (now - start) / dur);
      const e = t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2;
      ids.forEach(id => {
        const c = cards[id]; if (!c) return;
        c.x = from[id].x + (targets[id].x - from[id].x) * e;
        c.y = from[id].y + (targets[id].y - from[id].y) * e;
        const el = cardEl(id);
        if (el){
          el.style.transition = 'none';
          el.style.left = c.x + 'px'; el.style.top = c.y + 'px';
          el.style.transform = 'scale(' + (1 - 0.85 * e) + ')';
          el.style.opacity = String(1 - e * 0.92);
        }
      });
      if (t < 1) requestAnimationFrame(step); else done();
    }
    requestAnimationFrame(step);
  }

  function commitDelete(uidOrIds){
    pushHistory();
    const uid = Array.isArray(uidOrIds) ? uidOrIds[0] : uidOrIds;
    const removed = nodesRemovedBy(uidOrIds);
    const outs = [...removed].filter(id => cards[id] && cards[id].isOutput);
    const targets = {};
    outs.forEach(id => {
      const prod = linksArr.find(l => l.to === id);
      const pb = (prod && cards[prod.from]) ? cards[prod.from] : cards[uid];
      targets[id] = { x: pb.x, y: pb.y };
    });

    const finish = () => {
      linksArr = linksArr.filter(l => !removed.has(l.from) && !removed.has(l.to));
      drawLinks();
      // gli output sono gia rientrati: fa pop solo cio che resta, in un solo gesto
      [...removed].forEach(id => {
        if (outs.includes(id)) return;
        const el = cardEl(id); if (el) el.classList.add('popping');
      });
      setTimeout(() => { removed.forEach(id => delete cards[id]); renderAll(); }, 320);
    };

    if (outs.length) retract(outs, targets, 420, finish);
    else finish();
    hint.textContent = DEFAULT_HINT;
  }

  let pendingDelete = null;
  function closeConfirm(){ confirmBox.classList.remove('open'); pendingDelete = null; }
  function askConfirm(uid, removedCount){
    pendingDelete = uid;
    document.querySelector('.confirm-title').textContent = 'Eliminare il nodo?';
    const el = cardEl(uid);
    const r = el.getBoundingClientRect();
    confirmText.textContent = removedCount > 1
      ? 'Fa parte del flusso. Verranno rimossi i suoi collegamenti e ' + (removedCount - 1) + (removedCount > 2 ? ' risultati a valle.' : ' risultato a valle.')
      : 'Fa parte del flusso: i suoi collegamenti verranno rimossi.';
    confirmBox.classList.add('open');
    const bw = confirmBox.offsetWidth, bh = confirmBox.offsetHeight;
    let left = r.left + r.width / 2 - bw / 2;
    let top = r.top - bh - 10;
    if (top < 8) top = r.bottom + 10;
    left = Math.max(8, Math.min(window.innerWidth - bw - 8, left));
    confirmBox.style.left = left + 'px';
    confirmBox.style.top = top + 'px';
  }
  confirmCancel.addEventListener('click', (e) => { e.stopPropagation(); closeConfirm(); });
  confirmOk.addEventListener('click', (e) => {
    e.stopPropagation();
    const uid = pendingDelete;
    closeConfirm();
    if (uid) commitDelete(uid);
  });
  document.addEventListener('pointerdown', (e) => {
    if (pendingDelete && !e.target.closest('.confirm') && !e.target.closest('.del-btn')) closeConfirm();
  });

  function deleteMany(ids){
    ids = ids.filter(id => cards[id]);
    if (!ids.length) return;
    if (ids.length === 1){ deleteCard(ids[0]); return; }
    const attached = linksArr.some(l => ids.includes(l.from) || ids.includes(l.to));
    if (!attached){ commitDelete(ids); return; }
    const removed = nodesRemovedBy(ids);
    pendingDelete = ids;
    const el = cardEl(ids[0]);
    document.querySelector('.confirm-title').textContent = 'Eliminare ' + ids.length + ' nodi?';
    confirmText.textContent = 'Alcuni fanno parte del flusso: verranno rimossi i loro collegamenti' +
      (removed.size > ids.length ? ' e ' + (removed.size - ids.length) + ' risultati a valle.' : '.');
    confirmBox.classList.add('open');
    const r = el.getBoundingClientRect();
    const bw = confirmBox.offsetWidth, bh = confirmBox.offsetHeight;
    let top = r.top - bh - 10; if (top < 8) top = r.bottom + 10;
    confirmBox.style.left = Math.max(8, Math.min(window.innerWidth - bw - 8, r.left + r.width / 2 - bw / 2)) + 'px';
    confirmBox.style.top = top + 'px';
  }

  function deleteCard(uid){
    if (!cards[uid]) return;
    const attached = linksArr.some(l => l.from === uid || l.to === uid);
    if (!attached){ commitDelete(uid); return; }   // isolato: via subito, senza domande
    askConfirm(uid, nodesRemovedBy(uid).size);
  }

  function deleteLink(index){
    const l = linksArr[index];
    if (!l) return;
    pushHistory();
    linksArr.splice(index, 1);
    pruneOutputs();
    renderAll();
    hint.textContent = 'Collegamento rimosso';
    setTimeout(() => { if (hint.textContent === 'Collegamento rimosso') hint.textContent = DEFAULT_HINT; }, 1400);
  }

  linkHits.addEventListener('click', (e) => {
    const p = e.target.closest('.hit');
    if (!p) return;
    deleteLink(parseInt(p.dataset.link, 10));
  });

  stage.addEventListener('click', (e) => {
    if (suppressClick){ suppressClick = false; return; }     // un trascinamento sul vuoto non e un click
    if (!e.target.closest('.card') && !e.target.closest('.hit') && !e.target.closest('.zoom-ctl') && !e.target.closest('.minimap') && !e.target.closest('.notch')) deselect();
    const btn = e.target.closest('.del-btn');
    if (!btn) return;
    e.preventDefault(); e.stopPropagation();
    deleteCard(btn.closest('.card').dataset.uid);
  });

  undoBtn.addEventListener('click', undo);
  redoBtn.addEventListener('click', redo);
  // duplica i nodi selezionati, senza collegamenti, accanto agli originali
  function duplicateSelection(){
    const ids = (selectedSet.size ? [...selectedSet] : (selectedUid ? [selectedUid] : [])).filter(id => cards[id] && !cards[id].isOutput);
    if (!ids.length) return;
    pushHistory();
    const created = [];
    ids.forEach(id => {
      const src = cards[id];
      const uid = (src.kind === 'dataset' ? 'ds-' : 'op-') + (uidCounter++);
      cards[uid] = JSON.parse(JSON.stringify(src));
      delete cards[uid].slot; delete cards[uid].home;
      cards[uid].name = src.name + ' copia';
      cards[uid].x = src.x + GRID * 2; cards[uid].y = src.y + GRID * 2;
      if (layoutMode === 'grid'){ computeSlots(); cards[uid].slot = firstFreeSlot(cards[uid].x, cards[uid].y, uid); }
      const el = createCardEl(uid);
      el.style.transition = 'none'; el.style.transform = 'scale(.5)'; el.style.opacity = '0';
      requestAnimationFrame(() => {
        el.style.transition = 'transform .34s cubic-bezier(.34,1.56,.64,1), opacity .2s';
        el.style.transform = 'scale(1)'; el.style.opacity = '1';
      });
      setTimeout(() => { el.style.transition = ''; el.style.transform = ''; }, 360);
      created.push(uid);
    });
    if (layoutMode === 'grid') placeInSlots(true); else if (anyOverlap()) resolveOverlaps(null, null, true);
    setSelection(created);
  }
  function nudgeSelection(dx, dy){
    if (layoutMode === 'grid') return;           // in modalita organizzata le postazioni sono fisse
    const ids = [...selectedSet].filter(id => cards[id]);
    if (!ids.length) return;
    pushHistory();
    ids.forEach(id => { cards[id].x += dx; cards[id].y += dy; clampCard(cards[id]); delete cards[id].home; });
    applyPositions(false);
    drawLinks();
  }

  document.addEventListener('keydown', (e) => {
    if (FEATURES.keyboard && !isTyping(e.target)){
      if ((e.key === 'Delete' || e.key === 'Backspace') && (selectedSet.size || selectedUid)){
        e.preventDefault();
        deleteMany(selectedSet.size ? [...selectedSet] : [selectedUid]);
        return;
      }
      if (e.key === 'Escape'){
        closeConfirm();
        if (overlay.classList.contains('open')) overlay.classList.remove('open');
        else deselect();
        return;
      }
      const step = e.shiftKey ? GRID : 2;
      const arrows = { ArrowLeft:[-step,0], ArrowRight:[step,0], ArrowUp:[0,-step], ArrowDown:[0,step] };
      if (arrows[e.key] && selectedSet.size){ e.preventDefault(); nudgeSelection(arrows[e.key][0], arrows[e.key][1]); return; }
      if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'd'){ e.preventDefault(); duplicateSelection(); return; }
      if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'a'){ e.preventDefault(); setSelection(Object.keys(cards)); return; }
    }
    if (!(e.metaKey || e.ctrlKey)) return;
    const k = e.key.toLowerCase();
    if (k === 'z' && e.shiftKey){ e.preventDefault(); redo(); }
    else if (k === 'z'){ e.preventDefault(); undo(); }
    else if (k === 'y'){ e.preventDefault(); redo(); }
  });

  // --- Palette: si trascina da qui per creare un nodo ---
  // --- Cassetta degli strumenti: sezioni per famiglia di operazioni ---
  const SECTIONS = [
    { id:'data',  name:'Dataset' },
    { id:'rows',  name:'Filtra e ordina', items:['filter','sort','dedup','limit','sample','selectCols'] },
    { id:'xform', name:'Trasforma dati',  items:['compute','cast','round','scale','aggregate','textClean','replaceVal','splitCol','rename','fillNa'] },
    { id:'merge', name:'Merge e union',   items:['join','union'] },
    { id:'out',   name:'Output',          items:['exportOp'] }
  ];
  const PALETTE = [{ type:'dataset', kind:'dataset' }].concat(
    SECTIONS.filter(sc => sc.items).reduce((a, sc) => a.concat(sc.items), []).map(t => ({ type:t, kind:'op' })));
  const secOpen = { data:true, rows:true, xform:true, merge:true, out:true };

  // libreria dei dataset caricati: ognuno si trascina sul canvas quante volte serve
  const LIBRARY = [ { id:'lib-vendite', name:'Vendite 2026', path:'vendite_2026.csv', columns: null, rows: 1240 } ];
  let libCounter = 0;

  function parseCSV(text){
    const lines = text.replace(/\r/g, '').split('\n').filter(l => l.trim().length);
    if (!lines.length) return null;
    const head = lines[0];
    const delim = [',', ';', '\t', '|'].sort((a, b) => head.split(b).length - head.split(a).length)[0];
    const parseLine = (line) => {
      const out = []; let cur = '', q = false;
      for (let i = 0; i < line.length; i++){
        const ch = line[i];
        if (q){
          if (ch === '"'){ if (line[i+1] === '"'){ cur += '"'; i++; } else q = false; }
          else cur += ch;
        } else if (ch === '"') q = true;
        else if (ch === delim){ out.push(cur); cur = ''; }
        else cur += ch;
      }
      out.push(cur);
      return out.map(v => v.trim());
    };
    const header = parseLine(head);
    const rows = lines.slice(1, 1001).map(parseLine);
    // tipo e valori distinti dedotti da un campione delle righe
    const columns = header.map((name, ci) => {
      const vals = rows.map(r => r[ci]).filter(v => v !== undefined && v !== '');
      const isInt = vals.length && vals.every(v => /^-?\d+$/.test(v));
      const isNum = vals.length && vals.every(v => /^-?\d+([.,]\d+)?$/.test(v));
      const isDate = vals.length && vals.every(v => /^\d{4}-\d{2}-\d{2}/.test(v) || /^\d{1,2}\/\d{1,2}\/\d{2,4}$/.test(v));
      const type = isInt ? 'integer' : isNum ? 'numerico' : isDate ? 'data' : 'stringa';
      const distinct = Array.from(new Set(vals));
      return { name: name || ('colonna_' + (ci + 1)), type,
               // i valori distinti di ogni colonna, di qualunque tipo: il selettore li legge da qui
               values: (type === 'integer' || type === 'numerico'
                 ? distinct.slice().sort((a, b) => parseFloat(String(a).replace(',', '.')) - parseFloat(String(b).replace(',', '.')))
                 : distinct).slice(0, 500) };
    });
    return { columns, rows: lines.length - 1 };
  }
  // cosa succede rilasciando un elemento della palette sopra un nodo esistente
  function paletteRelation(kind, targetUid){
    const t = cards[targetUid];
    if (!t) return null;
    if (kind === 'op' && t.kind === 'op') return 'merge';
    if (kind === 'dataset' && t.kind === 'op'){
      if (inputsOf(targetUid).length >= boxCapacity(t)) return null;
      return 'link';
    }
    if (kind === 'op' && t.kind === 'dataset'){
      if (t.capacity > 1 && t.filled < t.capacity) return null;   // output incompleto
      return 'link-reverse';
    }
    return null;
  }

  const CHEV_R = '<span class="cond-chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.8" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg></span>';
  function palItem(type, extra){
    const src = type === 'dataset';
    return '<div class="pal-item'+(src ? ' source' : '')+'" data-type="'+type+'"'+(extra && extra.lib ? ' data-lib="'+extra.lib+'"' : '')+'>'+
      '<div class="pal-chip">'+svgTag(type)+'</div>'+
      '<div class="pal-label">'+esc(extra && extra.label ? extra.label : META[type].label)+'</div>'+
      (extra && extra.meta ? '<div class="lib-meta">'+esc(extra.meta)+'</div>' : '')+
    '</div>';
  }
  function buildPalette(){
    let html = '<div class="tb-head"><div class="tb-title">Strumenti</div>'+
      '<button class="close-btn" id="tbClose" aria-label="Nascondi la cassetta degli strumenti">'+
      '<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><polyline points="15 6 9 12 15 18"/></svg>'+
      '</button></div>';
    SECTIONS.forEach(sc => {
      let body = '';
      if (sc.id === 'data'){
        body += '<button type="button" class="tb-upload" id="tbUpload">'+svgTag('upload')+'Carica dataset</button>';
        body += LIBRARY.length ? LIBRARY.map(lb => palItem('dataset', {
                  lib: lb.id, label: lb.name,
                  meta: (lb.columns ? lb.columns.length : SCHEMA.length) + ' col · ' + lb.rows + ' righe' })).join('')
                : '<div class="tb-empty">Nessun dataset caricato</div>';
      } else body += sc.items.map(t => palItem(t)).join('');
      html += '<div class="tb-sec'+(secOpen[sc.id] ? ' open' : '')+'" data-sec="'+sc.id+'">'+
        '<button type="button" class="tb-sec-head">'+CHEV_R+'<span class="tb-sec-name">'+sc.name+'</span></button>'+
        '<div class="tb-sec-body">'+body+'</div></div>';
    });
    paletteEl.innerHTML = html;
  }

  // --- Pannelli agganciabili a qualunque bordo del canvas ---
  const toolboxEl = document.getElementById('toolbox');
  const edgeHint = document.getElementById('edgeHint');
  const PANELS = {
    tools: { el: toolboxEl, notch: document.getElementById('tbNotch'), side:'left',  open:true,  W:264, H:206, name:'gli strumenti' },
    insp:  { el: inspector, notch: document.getElementById('inspNotch'), side:'right', open:false, W:308, H:300, name:'l\u2019inspector' }
  };
  const SIDE_NAME = { left:'sinistro', right:'destro', top:'superiore', bottom:'inferiore' };
  // misure proprie di ciascun pannello; in gruppo entrambi prendono la maggiore, cosi cambiare scheda non sposta il canvas
  Object.values(PANELS).forEach(P => { P.W0 = P.W; P.H0 = P.H; });
  function panelExtent(P){ return (P.side === 'left' || P.side === 'right') ? P.W + 16 : P.H + 16; }
  const TAB_ICON = {
    tools: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="4" width="7" height="7" rx="1.5"/><rect x="13" y="4" width="7" height="7" rx="1.5"/><rect x="4" y="13" width="7" height="7" rx="1.5"/><rect x="13" y="13" width="7" height="7" rx="1.5"/></svg>',
    insp:  '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="4" y1="7" x2="20" y2="7"/><line x1="4" y1="17" x2="20" y2="17"/><circle cx="9" cy="7" r="2.4" fill="#fff"/><circle cx="15" cy="17" r="2.4" fill="#fff"/></svg>'
  };
  const TAB_NAME = { tools:'Strumenti', insp:'Inspector' };
  function renderTabs(){
    const grouped = PANELS.tools.side === PANELS.insp.side;
    const W = grouped ? Math.max(PANELS.tools.W0, PANELS.insp.W0) : null;
    const H = grouped ? Math.max(PANELS.tools.H0, PANELS.insp.H0) : null;
    Object.keys(PANELS).forEach(k => {
      const P = PANELS[k];
      const oldExt = panelExtent(P);
      P.W = grouped ? W : P.W0; P.H = grouped ? H : P.H0;
      P.el.style.setProperty('--pw', P.W + 'px');
      P.el.style.setProperty('--ph', P.H + 'px');
      // un pannello aperto a sinistra che cambia misura sposterebbe i nodi: la vista compensa
      if (P.open && P.side === 'left' && FEATURES.panZoom){
        const d = panelExtent(P) - oldExt;
        if (d){ view.x -= d; applyView(true); }
      }
      P.el.classList.toggle('grouped', grouped);
      let nav = P.el.querySelector(':scope > .dock-tabs');
      if (!nav){
        nav = document.createElement('nav');
        nav.className = 'dock-tabs';
        P.el.insertBefore(nav, P.el.firstChild);
        nav.addEventListener('click', (e) => {
          const b = e.target.closest('.dock-tab');
          if (b && b.dataset.tab !== k) switchTab(b.dataset.tab);
        });
      }
      nav.innerHTML = ['tools', 'insp'].map(t =>
        '<button type="button" class="dock-tab' + (t === k ? ' on' : '') + '" data-tab="' + t + '" title="' + TAB_NAME[t] + '">' +
        TAB_ICON[t] + '<span>' + TAB_NAME[t] + '</span></button>').join('');
    });
  }
  // cambiare scheda sostituisce il contenuto sul posto: nessuna animazione di larghezza, solo una dissolvenza
  function switchTab(key){
    const els = Object.values(PANELS).map(P => P.el);
    els.forEach(el => el.classList.add('instant'));
    setPanelOpen(key, true);
    const el = PANELS[key].el;
    el.classList.remove('tab-in'); void el.offsetWidth; el.classList.add('tab-in');
    setTimeout(() => el.classList.remove('tab-in'), 280);
    requestAnimationFrame(() => requestAnimationFrame(() => els.forEach(x => x.classList.remove('instant'))));
  }

  // a sinistra il canvas cede larghezza dal suo lato d'origine: la vista compensa e i nodi restano fermi.
  // sopra e sotto il canvas conserva la sua altezza: e l'area di lavoro a crescere
  function compensate(P, opening){
    if (!FEATURES.panZoom) return;
    if (P.side === 'left'){ view.x += opening ? -panelExtent(P) : panelExtent(P); applyView(true); }
  }
  function fitWorkspace(){
    const extra = Object.values(PANELS)
      .filter(P => P.open && (P.side === 'top' || P.side === 'bottom'))
      .reduce((a, P) => a + panelExtent(P), 0);
    workspace.style.height = 'calc(var(--stage-h) + ' + extra + 'px)';
  }
  function applyOpen(key, open){
    const P = PANELS[key];
    if (P.open === open) return;
    compensate(P, open);
    P.open = open;
    P.el.classList.toggle('open', open);
    fitWorkspace();
    layoutNotches();
    onStageResize();
  }
  function setPanelOpen(key, open){
    if (open){
      // sullo stesso bordo si apre un pannello alla volta
      const other = key === 'tools' ? 'insp' : 'tools';
      if (PANELS[other].side === PANELS[key].side) applyOpen(other, false);
    }
    applyOpen(key, open);
  }
  function layoutNotches(){
    const bySide = {};
    Object.keys(PANELS).forEach(k => { (bySide[PANELS[k].side] = bySide[PANELS[k].side] || []).push(k); });
    Object.keys(PANELS).forEach(k => {
      const P = PANELS[k], n = P.notch;
      if (n.classList.contains('dragging')) return;
      n.classList.remove('n-left', 'n-right', 'n-top', 'n-bottom');
      n.classList.add('n-' + P.side);
      // due tacche sullo stesso bordo si affiancano invece di sovrapporsi
      const group = bySide[P.side], off = group.length > 1 ? (group.indexOf(k) === 0 ? -40 : 40) : 0;
      n.style.left = n.style.top = n.style.right = n.style.bottom = '';
      if (P.side === 'left' || P.side === 'right') n.style.top = 'calc(50% - 33px + ' + off + 'px)';
      else n.style.left = 'calc(50% - 33px + ' + off + 'px)';
      const partner = PANELS[k === 'tools' ? 'insp' : 'tools'];
      const reachableByTab = partner.side === P.side && partner.open;
      n.classList.toggle('hidden', P.open || reachableByTab);
    });
    renderTabs();
  }
  function setSide(key, side){
    const P = PANELS[key];
    if (P.side === side) return;
    if (P.open) applyOpen(key, false);
    P.side = side;
    P.el.classList.remove('side-left', 'side-right', 'side-top', 'side-bottom');
    P.el.classList.add('side-' + side);
    P.el.classList.toggle('horiz', side === 'top' || side === 'bottom');   // l'orientamento segue il bordo
    document.getElementById('dock-' + side).appendChild(P.el);
    layoutNotches();
    if (key === 'insp') renderInspector();
    setTimeout(() => setPanelOpen(key, true), 40);
  }
  function nearestSide(cx, cy){
```


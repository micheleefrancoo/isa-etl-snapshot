# 10-prototype-c.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 3/6)

5098 righe totali

```html
  let hoverLinkIdx = -1;            // cavo evidenziato come bersaglio di inserimento
  let draggingUid = null;           // il nodo in mano non e un ostacolo per i cavi altrui
  let spaceDown = false;            // barra spaziatrice premuta: trascinare sposta la vista
  function insertable(idx, uid){
    const l = linksArr[idx];
    if (!l || l.from === uid || l.to === uid) return false;
    const a = cards[l.from], b = cards[l.to];
    // si interviene solo dove un dataset entra in una lavorazione: il prodotto di un box e suo
    return !!(a && b && a.kind === 'dataset' && b.kind === 'op');
  }
  function insertOnLink(idx, uid){
    const l = linksArr[idx];
    if (!l || !cards[uid]) return;
    const src = l.from, dst = l.to;
    const a = cards[src], b = cards[dst], x = cards[uid];
    // la lavorazione si colloca a meta strada, poi il flusso si riallaccia attraverso di lei
    x.x = Math.round((a.x + b.x) / 2); x.y = Math.round((a.y + b.y) / 2);
    if (layoutMode === 'grid'){ computeSlots(); x.slot = firstFreeSlot(x.x, x.y, uid); }
    linksArr.splice(idx, 1);
    linksArr.push({ from: src, to: uid });
    const out = spawnOutput(uid);
    if (out) linksArr.push({ from: out, to: dst });
    refreshOutput(uid);
    refreshOutput(dst);
    if (layoutMode === 'grid') placeInSlots(true);
    else resolveOverlaps(uid, null, true);
    drawLinks();
    syncInspector();
    hint.textContent = 'Lavorazione inserita nel collegamento';
    setTimeout(() => { if (hint.textContent === 'Lavorazione inserita nel collegamento') hint.textContent = DEFAULT_HINT; }, 1600);
  }

  function connect(sourceUid, targetUid){
    const box = cards[targetUid];
    if (!box) return;
    const cur = inputsOf(targetUid);
    if (cur.some(l => l.from === sourceUid)) return;
    if (cur.length >= boxCapacity(box)) return;
    linksArr.push({ from: sourceUid, to: targetUid });
    drawLinks();
    syncInspector();
    setTimeout(() => refreshOutput(targetUid), 240);
  }

  // un nodo non puo alimentare qualcosa che a sua volta lo alimenta: il flusso e a senso unico
  function reaches(fromUid, toUid){
    const seen = new Set();
    const stack = [fromUid];
    while (stack.length){
      const cur = stack.pop();
      if (cur === toUid) return true;
      if (seen.has(cur)) continue;
      seen.add(cur);
      linksArr.forEach(l => { if (l.from === cur) stack.push(l.to); });
    }
    return false;
  }

  let displaceReason = '';
  function linkRefusal(dsUid, boxUid){
    const ds = cards[dsUid], box = cards[boxUid];
    if (ds.capacity > 1 && ds.filled < ds.capacity)
      return 'Questo output non e ancora completo: al join manca una tabella';
    if (reaches(boxUid, dsUid))
      return 'Un box non puo agganciarsi a cio che produce: sarebbe un ciclo infinito';
    const cur = inputsOf(boxUid);
    if (cur.some(l => l.from === dsUid))
      return 'Questa tabella e gia collegata al box';
    const cap = boxCapacity(box);
    if (cur.length >= cap)
      return cap > 1 ? 'Il join ha gia le sue due tabelle' : 'Il box accetta una sola tabella in ingresso';
    return null;
  }

  function relation(aUid, bUid){
    const a = cards[aUid], b = cards[bUid];
    if (!a || !b) return null;
    if (a.kind==='op' && b.kind==='op') return 'merge';
    if (a.kind==='dataset' && b.kind==='op'){
      const why = linkRefusal(aUid, bUid);
      if (why){ displaceReason = why; return 'displace'; }
      return 'link';
    }
    if (a.kind==='op' && b.kind==='dataset'){
      const why = linkRefusal(bUid, aUid);
      if (why){ displaceReason = why; return 'displace'; }
      return 'link-reverse';
    }
    if (a.kind==='dataset' && b.kind==='dataset'){
      displaceReason = 'Due dataset non si fondono: serve una lavorazione, ad esempio un Join';
      return 'displace';
    }
    return null;
  }

  // I dataset non si sovrappongono mai: quello sotto si sposta per fare largo
  function displace(draggedUid, targetUid){
    const d = cards[draggedUid], t = cards[targetUid];
    if (!d || !t) return;
    let vx = (t.x + CARD/2) - (d.x + CARD/2);
    let vy = (t.y + CARD/2) - (d.y + CARD/2);
    const len = Math.hypot(vx, vy) || 1;
    if (len < 1){ vx = 0; vy = 1; }
    const push = CARD + 36;
    let nx = t.x + (vx / len) * push;
    let ny = t.y + (vy / len) * push;
    nx = Math.max(6, Math.min(worldW() - CARD - 6, nx));
    ny = Math.max(6, Math.min(worldH() - CARD - 26, ny));
    t.x = nx; t.y = ny;
    const el = cardEl(targetUid);
    el.style.transition = 'left .32s cubic-bezier(.34,1.56,.64,1), top .32s cubic-bezier(.34,1.56,.64,1)';
    el.style.left = nx + 'px';
    el.style.top = ny + 'px';
    setTimeout(() => { el.style.transition = ''; resolveOverlaps(draggedUid, null, true); }, 340);
  }

  stage.addEventListener('pointerdown',(e)=>{
    if (e.button!==0 && e.pointerType==='mouse') return;
    if (spaceDown) return;                                   // con la barra spaziatrice si naviga
    if (e.target.closest('.expand-btn') || e.target.closest('.del-btn') || e.target.closest('.port')) return;
    const card = e.target.closest('.card');
    if (!card || !cards[card.dataset.uid]) return;
    const uid = card.dataset.uid;
    const startX=e.clientX, startY=e.clientY;
    const origX=cards[uid].x, origY=cards[uid].y;
    const shiftAtStart = e.shiftKey;
    // un nodo che fa parte di una selezione multipla trascina con se tutto il gruppo
    const group = (FEATURES.multiSelect && selectedSet.has(uid) && selectedSet.size > 1)
      ? [...selectedSet].filter(id => cards[id]) : null;
    const groupOrig = {};
    if (group) group.forEach(id => { groupOrig[id] = { x: cards[id].x, y: cards[id].y }; });
    // solo una lavorazione ancora slegata puo essere inserita dentro un collegamento
    const canInsert = FEATURES.cableInsert && !group && cards[uid].kind === 'op' &&
                      !linksArr.some(l => l.from === uid || l.to === uid);
    let dragging=false, currentTarget=null, currentRel=null, lastDisplaced=null, lastPush=0, insertIdx=-1;

    function onMove(ev){
      const dx=(ev.clientX-startX)/view.z, dy=(ev.clientY-startY)/view.z;   // lo spostamento vive nel mondo
      if (!dragging){
        if (Math.hypot(ev.clientX-startX, ev.clientY-startY)<5) return;
        dragging=true; pushHistory();
        (group || [uid]).forEach(id => { delete cards[id].home; });
        card.classList.add('dragging'); flowPaused = true;
        if (!group) draggingUid = uid;
      }
      if (group){
        group.forEach(id => {
          cards[id].x = groupOrig[id].x + dx; cards[id].y = groupOrig[id].y + dy;
          const el = cardEl(id);
          if (el){ el.style.left = cards[id].x + 'px'; el.style.top = cards[id].y + 'px'; }
        });
        drawLinks();
        hint.textContent = group.length + ' nodi in movimento';
        return;
      }
      cards[uid].x=origX+dx; cards[uid].y=origY+dy;
      card.style.left=cards[uid].x+'px'; card.style.top=cards[uid].y+'px';
      if (layoutMode === 'grid'){
        const idx = nearestSlot(cards[uid].x, cards[uid].y, uid, false);
        if (idx >= 0){
          slotGhost.classList.add('on');
          slotGhost.style.left = slots[idx].x + 'px';
          slotGhost.style.top = slots[idx].y + 'px';
        }
      } else {
        const tnow = performance.now();
        if (tnow - lastPush > 90){ lastPush = tnow; resolveOverlaps(uid, uid, false, true); }  // solo le incompatibili si scansano
      }
      drawLinks();
      card.style.pointerEvents='none';
      const under=document.elementFromPoint(ev.clientX,ev.clientY);
      card.style.pointerEvents='';
      const tc = under ? under.closest('.card') : null;
      const rel = (tc && tc!==card) ? relation(uid, tc.dataset.uid) : null;

      // sopra un cavo, senza un nodo sotto: la lavorazione si puo inserire nel collegamento
      if (!tc && canInsert){
        const hitEl = under ? under.closest('.hit') : null;
        const idx = hitEl ? parseInt(hitEl.dataset.link, 10) : -1;
        const ok = idx >= 0 && insertable(idx, uid) ? idx : -1;
        if (ok !== insertIdx){
          insertIdx = ok; hoverLinkIdx = ok;
          hint.textContent = ok >= 0 ? 'Rilascia per inserire la lavorazione nel collegamento' : DEFAULT_HINT;
        }
      } else if (insertIdx !== -1){ insertIdx = -1; hoverLinkIdx = -1; }

      if (rel === 'displace'){
        if (tc.dataset.uid !== lastDisplaced){
          lastDisplaced = tc.dataset.uid;
          displace(uid, tc.dataset.uid);
        }
        if (currentTarget){ currentTarget.classList.remove('drop-target','link-target'); currentTarget=null; currentRel=null; }
        hint.textContent = displaceReason;
        return;
      }
      lastDisplaced = null;

      const valid = rel ? tc : null;
      if (valid !== currentTarget){
        if (currentTarget) currentTarget.classList.remove('drop-target','link-target');
        currentTarget=valid; currentRel=rel;
        if (currentTarget){
          currentTarget.classList.add(rel==='merge' ? 'drop-target' : 'link-target');
          if (rel==='merge') hint.textContent = 'Rilascia per fondere le lavorazioni';
          else {
            const boxUid = rel==='link' ? tc.dataset.uid : uid;
            const bx = cards[boxUid];
            const need = boxCapacity(bx) - inputsOf(boxUid).length;
            hint.textContent = need > 1
              ? 'Rilascia: sara la tabella di sinistra, poi servira la seconda'
              : 'Rilascia per collegare e generare l\u2019output';
          }
        } else if (insertIdx < 0) hint.textContent = DEFAULT_HINT;
      }
    }
    function onUp(ev){
      window.removeEventListener('pointermove',onMove);
      window.removeEventListener('pointerup',onUp);
      window.removeEventListener('pointercancel',onUp);
      hint.textContent=DEFAULT_HINT;
      card.classList.remove('dragging');
      slotGhost.classList.remove('on');
      hoverLinkIdx = -1;
      draggingUid = null;
      if (!dragging){
        // Maiusc+click aggiunge o toglie dalla selezione; il click semplice seleziona solo questo
        if (FEATURES.multiSelect && (shiftAtStart || (ev && ev.shiftKey))){ toggleInSelection(uid); return; }
        selectCard(uid); return;
      }
      if (flowPaused){
        flowPaused = false;
        const t = performance.now();
        Object.keys(linkState).forEach(k => { linkState[k].t0 = t; });   // i cavi hanno una forma nuova: il flusso riparte dalla sorgente
      }
      if (group){
        if (layoutMode === 'grid'){ computeSlots(); assignSlotsKeepingOrder(); }
        else if (anyOverlap()) resolveOverlaps(null, null, true);
        drawLinks();
        return;
      }
      if (insertIdx >= 0){ insertOnLink(insertIdx, uid); return; }
      if (currentTarget){
        currentTarget.classList.remove('drop-target','link-target');
        const tUid=currentTarget.dataset.uid;
        cards[uid].x=origX; cards[uid].y=origY;
        card.style.left=origX+'px'; card.style.top=origY+'px';
        if (currentRel==='merge') performMerge(uid,tUid);
        else if (currentRel==='link') connect(uid,tUid);
        else if (currentRel==='link-reverse') connect(tUid,uid);
      } else if (layoutMode === 'grid'){
        // postazioni fisse: ci si sposta solo occupandone una libera o scambiandosi
        const idx = nearestSlot(cards[uid].x, cards[uid].y, uid, false);
        if (idx >= 0){
          const occupant = Object.keys(cards).find(id => id !== uid && cards[id].slot === idx);
          if (occupant) cards[occupant].slot = cards[uid].slot;
          cards[uid].slot = idx;
        }
        placeInSlots(true);
      } else {
        resolveOverlaps(uid, null, true);   // rilasciato nel vuoto: separa e riallinea
      }
      drawLinks();
    }
    window.addEventListener('pointermove',onMove);
    window.addEventListener('pointerup',onUp);
    window.addEventListener('pointercancel',onUp);
  });

  function closeAllDropdowns(){
    document.querySelectorAll('.dropdown.open').forEach(d=>d.classList.remove('open'));
    document.querySelectorAll('.settings-btn.active').forEach(b=>b.classList.remove('active'));
    document.querySelectorAll('.step.menu-open').forEach(r=>r.classList.remove('menu-open'));
  }

  function renderSteps(){
    const d = cards[currentExpandedUid]; if (!d) return;
    expandSub.textContent = d.components.length + ' passaggi in sequenza';
    stepsEl.innerHTML = d.components.map((id,i)=>
      '<div class="step" data-step="'+id+'">'+
        '<div class="grip" aria-hidden="true">'+GRIP+'</div>'+
        '<div class="step-num">'+svgTag(id)+'</div>'+
        '<div class="step-info"><div class="step-order">Passo '+(i+1)+'</div><div class="step-name">'+META[id].label+'</div></div>'+
        '<div class="step-settings">'+
          '<button class="settings-btn" aria-label="Impostazioni passaggio" data-idx="'+i+'"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">'+GEAR+'</svg></button>'+
          '<div class="dropdown" data-idx="'+i+'">'+
            '<button type="button" class="config-step" data-idx="'+i+'">Configura parametri</button>'+
            '<button type="button" class="detach-step" data-idx="'+i+'">Sgancia sul canvas</button>'+
            '<button type="button" class="danger delete-step" data-idx="'+i+'">Elimina passaggio</button>'+
          '</div>'+
        '</div>'+
      '</div>').join('');
  }

  // --- Sganciare un passo dal box combinato: torna card singola sul canvas ---
  function detachStep(boxUid, index, dropPoint){
    const box = cards[boxUid];
    if (!box || box.kind !== 'op' || box.components.length < 2) return;
    pushHistory();
    const compId = box.components[index];
    if (compId === undefined) return;
    ensureParams(box);
    const detached = box.params.splice(index, 1)[0] || defaultParams(compId);
    box.components.splice(index, 1);

    const boxEl = cardEl(boxUid);
    const wrap = boxEl.querySelector('.icon-wrap');
    const oldRects = Array.from(wrap.querySelectorAll('svg')).filter(n=>!n.closest('.expand-btn')).map(n=>n.getBoundingClientRect());

    const stillCombined = box.components.length > 1;
    if (!stillCombined){
      box.name = META[box.components[0]].label;
      boxEl.classList.remove('combined');
    }
    wrap.className = 'icon-wrap ' + countClass(box.components.length);
    wrap.innerHTML = wrapInner(box.components, stillCombined);
    boxEl.querySelector('.label').textContent = box.name;

    // le icone rimaste si riassestano scivolando
    Array.from(wrap.querySelectorAll('svg')).filter(n=>!n.closest('.expand-btn')).forEach((node,i)=>{
      const src = oldRects[i < index ? i : i + 1] || oldRects[i];
      if (!src) return;
      const nr = node.getBoundingClientRect();
      node.style.transition='none';
      node.style.transform='translate('+((src.left-nr.left)/view.z)+'px,'+((src.top-nr.top)/view.z)+'px) scale('+(src.width/(nr.width||1))+')';
      requestAnimationFrame(()=>{ node.style.transition='transform .38s cubic-bezier(.34,1.56,.64,1)'; node.style.transform='translate(0,0) scale(1)'; });
    });
    wrap.classList.remove('recoil'); void wrap.offsetWidth; wrap.classList.add('recoil');

    // la card sganciata viene espulsa e torna autonoma sul canvas
    const uid = 'op-' + (uidCounter++);
    let pos;
    if (dropPoint){
      pos = { x: Math.max(6, Math.min(worldW() - CARD - 6, dropPoint.x - CARD/2)),
              y: Math.max(6, Math.min(worldH() - CARD - 26, dropPoint.y - CARD/2)) };
    } else {
      pos = freeSpot(Math.max(6, Math.min(box.x, worldW() - CARD - 6)), box.y + CARD + 34);
    }
    cards[uid] = { kind:'op', components:[compId], params:[detached], name: META[compId].label, x: pos.x, y: pos.y };
    if (layoutMode === 'grid'){ computeSlots(); cards[uid].slot = firstFreeSlot(pos.x, pos.y, uid); }
    const el = createCardEl(uid);
    const dx = (box.x + CARD/2) - (pos.x + CARD/2);
    const dy = (box.y + CARD/2) - (pos.y + CARD/2);
    el.style.transition='none';
    el.style.transform='translate('+dx+'px,'+dy+'px) scale(.2)';
    el.style.opacity='0';
    requestAnimationFrame(()=>{
      el.style.transition='transform .5s cubic-bezier(.34,1.56,.64,1), opacity .22s ease';
      el.style.transform='translate(0,0) scale(1)';
      el.style.opacity='1';
    });
    resolveOverlaps(uid, uid, true);
    setTimeout(()=>{ el.style.transition=''; el.style.transform=''; }, 560);

    enforceCapacity(boxUid);
    refreshOutput(boxUid);
    reconcileDom();
    drawLinks();
    syncInspector();
    if (!stillCombined){ overlay.classList.remove('open'); currentExpandedUid = null; }
    else if (overlay.classList.contains('open')) renderSteps();
  }

  function deleteStep(boxUid, index){
    const box = cards[boxUid];
    if (!box || box.kind !== 'op' || box.components.length < 2) return;
    pushHistory();
    ensureParams(box);
    box.params.splice(index, 1);
    box.components.splice(index, 1);

    const boxEl = cardEl(boxUid);
    const wrap = boxEl.querySelector('.icon-wrap');
    const oldRects = Array.from(wrap.querySelectorAll('svg'))
      .filter(n => !n.closest('.expand-btn')).map(n => n.getBoundingClientRect());

    const stillCombined = box.components.length > 1;
    if (!stillCombined){
      box.name = META[box.components[0]].label;
      boxEl.classList.remove('combined');
    }
    wrap.className = 'icon-wrap ' + countClass(box.components.length);
    wrap.innerHTML = wrapInner(box.components, stillCombined);
    boxEl.querySelector('.label').textContent = box.name;

    // le icone rimaste richiudono il vuoto scivolando
    Array.from(wrap.querySelectorAll('svg')).filter(n => !n.closest('.expand-btn')).forEach((node, i) => {
      const src = oldRects[i < index ? i : i + 1] || oldRects[i];
      if (!src) return;
      const nr = node.getBoundingClientRect();
      node.style.transition = 'none';
      node.style.transform = 'translate('+((src.left-nr.left)/view.z)+'px,'+((src.top-nr.top)/view.z)+'px) scale('+(src.width/(nr.width||1))+')';
      requestAnimationFrame(() => {
        node.style.transition = 'transform .38s cubic-bezier(.34,1.56,.64,1)';
        node.style.transform = 'translate(0,0) scale(1)';
      });
    });
    wrap.classList.remove('recoil'); void wrap.offsetWidth; wrap.classList.add('recoil');

    enforceCapacity(boxUid);
    refreshOutput(boxUid);
    reconcileDom();
    drawLinks();
    syncInspector();
    if (!stillCombined){ overlay.classList.remove('open'); currentExpandedUid = null; }
    else renderSteps();
  }

  function openExpand(uid){
    const d=cards[uid]; if (!d) return;
    currentExpandedUid=uid;
    expandTitle.textContent=d.name;
    renderSteps();
    overlay.classList.add('open');
  }

  stage.addEventListener('click',(e)=>{
    const btn=e.target.closest('.expand-btn'); if (!btn) return;
    e.preventDefault(); e.stopPropagation();
    openExpand(btn.closest('.card').dataset.uid);
  });

  // --- Riordino righe: le altre scorrono, la trascinata segue il puntatore ---
  stepsEl.addEventListener('pointerdown',(e)=>{
    if (e.target.closest('.settings-btn') || e.target.closest('.dropdown')) return;
    const row=e.target.closest('.step'); if (!row) return;
    e.preventDefault();
    const rows=Array.from(stepsEl.querySelectorAll('.step'));
    const rects=rows.map(r=>r.getBoundingClientRect());
    const startIndex=rows.indexOf(row);
    const stepH = rects.length>1 ? (rects[1].top - rects[0].top) : rects[0].height + 6;
    const startY=e.clientY, startX=e.clientX;
    const panel=document.querySelector('.expand-card');
    const panelRect=panel.getBoundingClientRect();
    let dragging=false, newIndex=startIndex, outside=false;

    function applyShifts(){
      rows.forEach((r,i)=>{
        if (r===row) return;
        let shift=0;
        if (startIndex<newIndex && i>startIndex && i<=newIndex) shift=-stepH;
        else if (startIndex>newIndex && i>=newIndex && i<startIndex) shift=stepH;
        r.style.transition='transform .2s cubic-bezier(.32,.72,0,1)';
        r.style.transform='translateY('+shift+'px)';
      });
    }
    function onMove(ev){
      const dy=ev.clientY-startY;
      if (!dragging){
        if (Math.abs(dy)<4) return;
        dragging=true; row.classList.add('drag-row'); row.style.transition='none';
      }
      const dxp=ev.clientX-startX;
      const nowOutside = ev.clientX < panelRect.left - 12 || ev.clientX > panelRect.right + 12
                      || ev.clientY < panelRect.top - 12 || ev.clientY > panelRect.bottom + 12;
      if (nowOutside !== outside){
        outside = nowOutside;
        row.style.opacity = outside ? '0.55' : '1';
        document.querySelector('.reorder-note').textContent = outside
          ? 'Rilascia qui fuori per sganciare il passaggio sul canvas'
          : 'Trascina le righe per cambiare l\u2019ordine di esecuzione.';
      }
      if (outside){ row.style.transform='translate('+dxp+'px,'+dy+'px) scale(.9)'; return; }
      row.style.transform='translateY('+dy+'px)';
      let idx=Math.round(startIndex + dy/stepH);
      idx=Math.max(0,Math.min(rows.length-1,idx));
      if (idx!==newIndex){ newIndex=idx; applyShifts(); }
    }
    function onUp(ev){
      window.removeEventListener('pointermove',onMove);
      window.removeEventListener('pointerup',onUp);
      if (!dragging){ return; }
      row.classList.remove('drag-row');
      row.style.opacity='';
      rows.forEach(r=>{ r.style.transition=''; r.style.transform=''; });
      document.querySelector('.reorder-note').textContent='Trascina le righe per cambiare l\u2019ordine di esecuzione.';
      if (outside){
        const boxUid = currentExpandedUid;
        const sr = stage.getBoundingClientRect();
        let drop = null;
        if (ev && ev.clientX >= sr.left && ev.clientX <= sr.right && ev.clientY >= sr.top && ev.clientY <= sr.bottom){
          drop = toWorld(ev.clientX, ev.clientY);      // il punto di rilascio vive nel mondo
        }
        overlay.classList.remove('open');   // si torna al canvas per vedere la card riapparire
        closeAllDropdowns();
        detachStep(boxUid, startIndex, drop);
        return;
      }
      if (newIndex!==startIndex){
        pushHistory();
        const cd=cards[currentExpandedUid];
        ensureParams(cd);
        const comps=cd.components;
        const [moved]=comps.splice(startIndex,1);
        comps.splice(newIndex,0,moved);
        const [movedP]=cd.params.splice(startIndex,1);
        cd.params.splice(newIndex,0,movedP);
        const wrap=cardEl(currentExpandedUid).querySelector('.icon-wrap');
        wrap.innerHTML=wrapInner(comps, true);
      }
      renderSteps();
    }
    window.addEventListener('pointermove',onMove);
    window.addEventListener('pointerup',onUp);
  });

  stepsEl.addEventListener('click',(e)=>{
    const btn=e.target.closest('.settings-btn');
    if (btn){
      const dd=stepsEl.querySelector('.dropdown[data-idx="'+btn.dataset.idx+'"]');
      const wasOpen=dd.classList.contains('open');
      closeAllDropdowns();
      if (!wasOpen){ dd.classList.add('open'); btn.classList.add('active'); btn.closest('.step').classList.add('menu-open'); }
      e.stopPropagation(); return;
    }
    const cfg = e.target.closest('.config-step');
    if (cfg){
      const idx = parseInt(cfg.dataset.idx, 10);
      const boxUid = currentExpandedUid;
      closeAllDropdowns();
      overlay.classList.remove('open');
      selectCard(boxUid);
      selectedStep = idx;
      renderInspector();
      e.stopPropagation();
      return;
    }
    const dt = e.target.closest('.detach-step');
    if (dt){
      const idx = parseInt(dt.dataset.idx, 10);
      closeAllDropdowns();
      detachStep(currentExpandedUid, idx);
      e.stopPropagation();
      return;
    }
    const dl = e.target.closest('.delete-step');
    if (dl){
      const idx = parseInt(dl.dataset.idx, 10);
      closeAllDropdowns();
      deleteStep(currentExpandedUid, idx);
      e.stopPropagation();
      return;
    }
    if (e.target.closest('.dropdown')) closeAllDropdowns();
  });
  document.addEventListener('click',()=>{ closeAllDropdowns(); closeSelects(); });

  expandTitle.addEventListener('blur',()=>{
    const val=expandTitle.textContent.trim()||'Combined Box';
    expandTitle.textContent=val;
    if (currentExpandedUid && cards[currentExpandedUid]){
      cards[currentExpandedUid].name=val;
      const le=cardEl(currentExpandedUid).querySelector('.label');
      if (le) le.textContent=val;
    }
  });
  expandTitle.addEventListener('keydown',(e)=>{ if (e.key==='Enter'){ e.preventDefault(); expandTitle.blur(); } });

  closeBtn.addEventListener('click',()=>{ overlay.classList.remove('open'); closeAllDropdowns(); });
  overlay.addEventListener('click',(e)=>{ if (e.target===overlay){ overlay.classList.remove('open'); closeAllDropdowns(); } });
  resetBtn.addEventListener('click',init);

  // --- Schema di esempio: colonne, tipi e valori distinti noti ---
  const SCHEMA = [
    { name:'id',        type:'integer', values: Array.from({ length: 30 }, (_, i) => String(i + 1)) },
    { name:'cliente',   type:'stringa', values:['Acme','Borealis','Cedro','Delta','Eureka'] },
    { name:'regione',   type:'stringa', values:['Nord','Centro','Sud','Isole'] },
    { name:'categoria', type:'object',  values:['Hardware','Software','Servizi','Consulenza'] },
    { name:'stato',     type:'stringa', values:['Aperto','In corso','Chiuso','Annullato'] },
    { name:'quantita',  type:'integer', values:['1','2','3','5','8','10','12','20','25','50'] },
    { name:'importo',   type:'numerico', values:['45.2','80','120.5','300','512.9','1049'] },
    { name:'data',      type:'data',    values:['2026-01-03','2026-01-04','2026-01-05','2026-01-06','2026-02-01'] }
  ];
  function schemaOf(uid, depth){
    depth = depth || 0;
    const d = cards[uid];
    if (!d || depth > 24) return null;
    if (d.kind === 'dataset'){
      if (!d.isOutput){ const p0 = d.params && d.params[0]; return (p0 && p0.columns) || null; }
      const prod = linksArr.find(l => l.to === uid);
      return prod ? schemaOf(prod.from, depth + 1) : null;
    }
    // una lavorazione vede l'unione delle colonne delle sue tabelle in ingresso
    const out = [];
    inputsOf(uid).forEach(l => (schemaOf(l.from, depth + 1) || []).forEach(c => {
      if (!out.some(o => o.name === c.name)) out.push(c);
    }));
    return out.length ? out : null;
  }
  function activeSchema(){ return (selectedUid && schemaOf(selectedUid)) || SCHEMA; }
  function columnDef(name){ return activeSchema().find(c => c.name === name) || null; }
  const MULTI_OPS = ['=','≠','è uno di','non è uno di','contiene'];
  const NO_VALUE_OPS = ['è vuoto','non è vuoto'];
  const FILTER_OPS = ['=','≠','è uno di','non è uno di','contiene','>','<','≥','≤','è vuoto','non è vuoto'];
  const SEPARATORS = [
    { label:'virgola', ch:',' }, { label:'punto e virgola', ch:';' },
    { label:'barra verticale', ch:'|' }, { label:'a capo', ch:'\n' }
  ];
  function newCondition(){
    return { column:'', op:'=', mode:'list', values:[], text:'', sep:',' };
  }

  // --- Parametri per tipo di funzione ---
  const PARAM_DEFS = {
    dataset: [
      { k:'source', label:'Origine', type:'select', opts:['CSV','Database','API','Foglio di calcolo'], def:'CSV' },
      { k:'path',   label:'Percorso o tabella', type:'text', def:'' },
      { k:'header', label:'Prima riga di intestazione', type:'select', opts:['Sì','No'], def:'Sì' }
    ],
    filter: 'custom',
    join: [
      { k:'type', label:'Tipo di join', type:'select', opts:['inner','left','right','full'], def:'inner' }
    ],
    sort: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'dir',    label:'Direzione', type:'select', opts:['crescente','decrescente'], def:'crescente' }
    ],
    exportOp: [
      { k:'format', label:'Formato', type:'select', opts:['CSV','XLSX','Parquet','Tabella DB'], def:'CSV' },
      { k:'dest',   label:'Destinazione', type:'text', def:'', req:true }
    ],
    dedup: [
      { k:'column', label:'Colonna chiave', type:'column', def:'' },
      { k:'keep',   label:'Mantieni', type:'select', opts:['la prima','l\u2019ultima'], def:'la prima' }
    ],
    limit: [
      { k:'n',    label:'Numero di righe', type:'text', def:'100', req:true },
      { k:'from', label:'Dall\u2019', type:'select', opts:['inizio','fine'], def:'inizio' }
    ],
    sample: [
      { k:'pct',  label:'Percentuale', type:'text', def:'10', req:true },
      { k:'seed', label:'Seme casuale', type:'text', def:'' }
    ],
    selectCols: [
      { k:'column', label:'Colonna da tenere', type:'column', def:'' },
      { k:'mode',   label:'Modo', type:'select', opts:['tieni','escludi'], def:'tieni' }
    ],
    compute: [
      { k:'name',    label:'Nuova colonna', type:'text', def:'', req:true },
      { k:'formula', label:'Formula', type:'text', def:'', req:true }
    ],
    cast: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'to',     label:'Nuovo tipo', type:'select', opts:['intero','decimale','testo','data','booleano'], def:'decimale' }
    ],
    round: [
      { k:'column',   label:'Colonna', type:'column', def:'' },
      { k:'decimals', label:'Decimali', type:'text', def:'2', req:true }
    ],
    scale: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'method', label:'Metodo', type:'select', opts:['min-max','z-score','percentuale'], def:'min-max' }
    ],
    aggregate: [
      { k:'groupBy', label:'Raggruppa per', type:'column', def:'' },
      { k:'measure', label:'Misura', type:'column', def:'' },
      { k:'fn',      label:'Funzione', type:'select', opts:['somma','media','conteggio','minimo','massimo'], def:'somma' }
    ],
    textClean: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'action', label:'Operazione', type:'select', opts:['rimuovi spazi','maiuscole','minuscole','iniziali maiuscole'], def:'rimuovi spazi' }
    ],
    replaceVal: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'find',   label:'Cerca', type:'text', def:'', req:true },
      { k:'with',   label:'Sostituisci con', type:'text', def:'' }
    ],
    splitCol: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'sep',    label:'Separatore', type:'select', opts:[',',';','|','spazio','-'], def:',' }
    ],
    rename: [
      { k:'column',  label:'Colonna', type:'column', def:'' },
      { k:'newName', label:'Nuovo nome', type:'text', def:'', req:true }
    ],
    fillNa: [
      { k:'column', label:'Colonna', type:'column', def:'' },
      { k:'value',  label:'Valore di riempimento', type:'text', def:'', req:true }
    ],
    union: [
      { k:'mode',  label:'Righe', type:'select', opts:['tutte','senza duplicati'], def:'tutte' },
      { k:'align', label:'Allineamento colonne', type:'select', opts:['per nome','per posizione'], def:'per nome' }
    ]
  };
  // --- Operazioni a righe multiple: una lista di voci, ognuna comprimibile come le condizioni del filtro ---
  const COLF = (label) => ({ k:'column', label: label || 'Colonna', type:'column', def:'' });
  // valori scelti dal dominio di una colonna: elenco a spunta oppure scrittura con separatore
  const VALUES_DEF = () => ({ mode:'list', values:[], text:'', sep:',' });
  function valuesText(v){
    if (!v || typeof v !== 'object') return v || '';
    return v.mode === 'list' ? v.values.join(', ') : v.text;
  }
  function fieldFilled(f, v){
    if (f.type === 'values') return !!(v && ((v.values && v.values.length) || (v.text && v.text.trim())));
    return !!(v && String(v).trim());
  }
  const MULTI_DEFS = {
    cast: { lists:[{ key:'items', label:'Colonne da convertire', noun:'Conversione', add:'Aggiungi conversione',
      fields:[ COLF(), { k:'to', label:'Nuovo tipo', type:'select', opts:['intero','decimale','testo','data','booleano'], def:'decimale' } ],
      sum: r => r.column ? r.column + ' \u2192 ' + r.to : null }] },
    rename: { lists:[{ key:'items', label:'Colonne da rinominare', noun:'Rinomina', add:'Aggiungi colonna',
      fields:[ COLF(), { k:'newName', label:'Nuovo nome', type:'text', def:'', req:true } ],
      sum: r => r.column ? r.column + ' \u2192 ' + (r.newName || '\u2026') : null }] },
    fillNa: { lists:[{ key:'items', label:'Colonne da riempire', noun:'Riempimento', add:'Aggiungi colonna',
      fields:[ COLF(), { k:'value', label:'Valore di riempimento', type:'value', def:'', req:true } ],
      sum: r => r.column ? r.column + ' = ' + (r.value || '\u2026') : null }] },
    replaceVal: { lists:[{ key:'items', label:'Sostituzioni', noun:'Sostituzione', add:'Aggiungi sostituzione',
      fields:[ COLF(),
               { k:'match', label:'Quando il valore', type:'select', opts:['è uguale a','è diverso da','contiene','inizia con','finisce con','corrisponde all\u2019espressione'], def:'è uguale a' },
               { k:'find', label:'Valori da cercare', type:'values', def: VALUES_DEF, req:true },
               { k:'with', label:'Sostituisci con', type:'value', def:'' } ],
      sum: r => r.column ? r.column + ' ' + (r.match || 'è uguale a') + ' ' + (valuesText(r.find) || '\u2026') + ' \u2192 ' + (r['with'] || '\u2205') : null }] },
    round: { lists:[{ key:'items', label:'Colonne da arrotondare', noun:'Arrotondamento', add:'Aggiungi colonna',
      fields:[ COLF(), { k:'decimals', label:'Decimali', type:'text', def:'2', req:true } ],
      sum: r => r.column ? r.column + ' \u00b7 ' + r.decimals + ' decimali' : null }] },
    scale: { lists:[{ key:'items', label:'Colonne da normalizzare', noun:'Normalizzazione', add:'Aggiungi colonna',
      fields:[ COLF(), { k:'method', label:'Metodo', type:'select', opts:['min-max','z-score','percentuale'], def:'min-max' } ],
      sum: r => r.column ? r.column + ' \u00b7 ' + r.method : null }] },
    textClean: { lists:[{ key:'items', label:'Colonne da pulire', noun:'Pulizia', add:'Aggiungi colonna',
      fields:[ COLF(), { k:'action', label:'Operazione', type:'select', opts:['rimuovi spazi','maiuscole','minuscole','iniziali maiuscole'], def:'rimuovi spazi' } ],
      sum: r => r.column ? r.column + ' \u00b7 ' + r.action : null }] },
    compute: { lists:[{ key:'items', label:'Colonne calcolate', noun:'Colonna', add:'Aggiungi colonna calcolata',
      fields:[ { k:'name', label:'Nuova colonna', type:'text', def:'', req:true }, { k:'formula', label:'Formula', type:'text', def:'', req:true } ],
      sum: r => r.name ? r.name + ' = ' + (r.formula || '\u2026') : null }] },
    selectCols: { globals:[ { k:'mode', label:'Modo', type:'select', opts:['tieni','escludi'], def:'tieni' } ],
      lists:[{ key:'items', label:'Colonne', noun:'Colonna', add:'Aggiungi colonna', fields:[ COLF() ], sum: r => r.column || null }] },
    dedup: { globals:[ { k:'keep', label:'Mantieni', type:'select', opts:['la prima','l\u2019ultima'], def:'la prima' } ],
      lists:[{ key:'items', label:'Colonne chiave', noun:'Chiave', add:'Aggiungi chiave',
      fields:[ COLF(), { k:'cmp', label:'Confronto', type:'select', opts:['esatto','ignora maiuscole','ignora spazi','ignora maiuscole e spazi'], def:'esatto' } ],
      sum: r => r.column ? r.column + (r.cmp && r.cmp !== 'esatto' ? ' \u00b7 ' + r.cmp : '') : null,
      note:'Due righe sono duplicate quando coincidono su tutte le chiavi.' }] },
    sort: { lists:[{ key:'items', label:'Criteri di ordinamento', noun:'Criterio', add:'Aggiungi criterio',
      fields:[ COLF(), { k:'dir', label:'Direzione', type:'select', opts:['crescente','decrescente'], def:'crescente' } ],
      sum: r => r.column ? r.column + (r.dir === 'crescente' ? ' \u2191' : ' \u2193') : null,
      note:'Il primo criterio \u00e8 il principale; i successivi decidono a parit\u00e0 del precedente.' }] },
    aggregate: { lists:[
      { key:'groupBy', label:'Raggruppa per', noun:'Chiave', add:'Aggiungi chiave', fields:[ COLF() ], sum: r => r.column || null },
      { key:'measures', label:'Misure', noun:'Misura', add:'Aggiungi misura',
        fields:[ COLF('Colonna'), { k:'fn', label:'Funzione', type:'select', opts:['somma','media','conteggio','minimo','massimo'], def:'somma' },
                 { k:'alias', label:'Nome del risultato', type:'text', def:'' } ],
        sum: r => r.column ? r.fn + '(' + r.column + ')' + (r.alias ? ' \u2192 ' + r.alias : '') : null } ] }
  };
  function blankRow(L){ const r = {}; L.fields.forEach(f => { r[f.k] = typeof f.def === 'function' ? f.def() : f.def; }); return r; }
  function ensureMulti(type, par){
    const md = MULTI_DEFS[type];
    (md.globals || []).forEach(f => { if (par[f.k] === undefined) par[f.k] = f.def; });
    md.lists.forEach(L => {
      if (!Array.isArray(par[L.key])){
        // i parametri gia impostati nel vecchio formato a voce singola diventano la prima voce
        const row = blankRow(L);
        L.fields.forEach(f => { if (par[f.k] !== undefined && par[f.k] !== '') row[f.k] = par[f.k]; });
        par[L.key] = [row];
      }
      // un valore scritto nel vecchio formato testuale diventa una scelta manuale
      par[L.key].forEach(r => L.fields.forEach(f => {
        if (f.type === 'values' && (r[f.k] === undefined || typeof r[f.k] !== 'object'))
          r[f.k] = Object.assign(VALUES_DEF(), r[f.k] ? { mode:'manual', text: String(r[f.k]) } : {});
      }));
    });
    return par;
  }

  function defaultParams(type){
    if (type === 'filter') return { logic:'E', conditions:[ newCondition() ] };
    if (type === 'join') return { type:'inner', keys:[ { left:'', right:'' } ] };
    if (MULTI_DEFS[type]) return ensureMulti(type, {});
    const o = {};
    const defs = PARAM_DEFS[type];
    if (Array.isArray(defs)) defs.forEach(f => { o[f.k] = f.def; });
    return o;
  }
  function ensureParams(card){
    if (!card.params) card.params = card.components.map(c => defaultParams(c));
    while (card.params.length < card.components.length) card.params.push(defaultParams(card.components[card.params.length]));
    return card.params;
  }

  // --- Inspector laterale: occupa spazio proprio, non copre mai il canvas ---
  let inspectorSide = 'right';           // impostabile in seguito dalla sezione View
  let selectedUid = null, selectedStep = 0, inspCollapsed = false;

  function setInspectorSide(side){
    inspectorSide = side;
    workspace.classList.toggle('side-left', side === 'left');
    onStageResize();
  }
  // Il canvas cede spazio solo temporaneamente: si ricorda dove stava ogni nodo
  let lastStageW = null;
  function anyOverlap(){
    const ids = Object.keys(cards);
    for (let i = 0; i < ids.length; i++)
      for (let j = i + 1; j < ids.length; j++){
        const A = cards[ids[i]], B = cards[ids[j]];
        if (Math.abs(A.x - B.x) < CARD + 18 && Math.abs(A.y - B.y) < CARD + LABEL_H + 18) return true;
      }
    return false;
  }
  function handleStageWidthChange(){
    if (FEATURES.panZoom){ lastStageW = stage.clientWidth; drawLinks(); return; }
    const w = stage.clientWidth;
    const narrowing = lastStageW !== null && w < lastStageW - 1;
    const widening  = lastStageW !== null && w > lastStageW + 1;
    lastStageW = w;

    if (narrowing){
      // memorizza la posizione di partenza, solo la prima volta
      Object.keys(cards).forEach(uid => {
        const c = cards[uid];
        if (!c.home) c.home = { x: c.x, y: c.y };
      });
    }
    if (widening){
      // lo spazio e tornato: chi era stato spostato dal restringimento rientra al suo posto
      Object.keys(cards).forEach(uid => {
        const c = cards[uid];
        if (c.home){ c.x = c.home.x; c.y = c.home.y; delete c.home; }
      });
    }

    if (layoutMode === 'grid'){ computeSlots(); assignSlotsKeepingOrder(); }
    else {
      Object.keys(cards).forEach(uid => clampCard(cards[uid]));
      if (anyOverlap()) resolveOverlaps(null, null, true);
      else applyPositions(true);
    }
    if (narrowing){
      // chi non si e mosso non ha nulla da ripristinare
      Object.keys(cards).forEach(uid => {
        const c = cards[uid];
        if (c.home && Math.abs(c.home.x - c.x) < 0.5 && Math.abs(c.home.y - c.y) < 0.5) delete c.home;
      });
    }
    drawLinks();
  }
  function onStageResize(){
    setTimeout(handleStageWidthChange, 310);
  }
  function assignSlotsKeepingOrder(){
    const order = Object.keys(cards).sort((A,B) => (cards[A].y - cards[B].y) || (cards[A].x - cards[B].x));
    Object.keys(cards).forEach(uid => { cards[uid].slot = undefined; });
    order.forEach(uid => { cards[uid].slot = firstFreeSlot(cards[uid].x, cards[uid].y, uid); });
    placeInSlots(true);
  }

  // --- Selezione: un nodo principale, piu eventuali altri ---
  const selectedSet = new Set();
  function paintSelection(){
    Array.from(stage.querySelectorAll('.card')).forEach(el =>
      el.classList.toggle('selected', el.dataset.uid === selectedUid || selectedSet.has(el.dataset.uid)));
  }
  function openInspector(){ setPanelOpen('insp', true); }
  function toggleInSelection(uid){
    if (selectedUid && !selectedSet.size) selectedSet.add(selectedUid);
    if (selectedSet.has(uid)) selectedSet.delete(uid); else selectedSet.add(uid);
    if (!selectedSet.size){ deselect(); return; }
    if (!selectedSet.has(selectedUid)) selectedUid = [...selectedSet][0];
    selectedStep = 0;
    paintSelection();
    renderInspector();
    openInspector();
  }
  function setSelection(ids){
    selectedSet.clear();
    ids.forEach(id => selectedSet.add(id));
    if (!ids.length){ deselect(); return; }
    selectedUid = ids[0]; selectedStep = 0;
    paintSelection();
    renderInspector();
    openInspector();
  }

  function selectCard(uid){
    selectedUid = uid; selectedStep = 0; openCond = 0; openKey = 0; openRows = {};
    selectedSet.clear(); selectedSet.add(uid);
    paintSelection();
    renderInspector();
    openInspector();
  }
  function deselect(){
    selectedUid = null; inspCollapsed = false;
    selectedSet.clear();
    paintSelection();
    renderInspector();
    setPanelOpen('insp', false);
  }

  function fieldHtml(f, val, idx){
    if (f.type === 'select'){
      return '<div class="field"><label>'+f.label+'</label>'+
        selectHtml(f.opts, val || f.def, 'data-key="'+f.k+'" data-step="'+idx+'"')+'</div>';
    }
    if (f.type === 'column'){
      return '<div class="field"><label>'+f.label+'</label>'+
        columnSelect(val || '', 'data-key="'+f.k+'" data-step="'+idx+'"')+'</div>';
    }
    return '<div class="field"><label>'+f.label+'</label>'+
      '<input type="text" data-key="'+f.k+'" data-step="'+idx+'" value="'+esc(val)+'"></div>';
  }

  const CHEV = '<span class="chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg></span>';
  function selectHtml(opts, value, attrs, labels){
    const shown = labels ? (labels[opts.indexOf(value)] || value) : value;
    return '<div class="sel" '+attrs+' data-value="'+esc(value)+'">'+
      '<button type="button" class="sel-btn"><span class="sel-val">'+esc(shown)+'</span>'+CHEV+'</button>'+
      '<div class="sel-menu">'+ opts.map((o, i) =>
        '<button type="button" data-opt="'+esc(o)+'"'+(o === value ? ' class="on"' : '')+'>'+
        esc(labels ? labels[i] : o)+'</button>').join('') +'</div>'+
    '</div>';
  }
  function columnSelect(value, attrs){
    const known = activeSchema().map(c => c.name);
    const isFree = value && !known.includes(value);
    const shown = value || 'scegli o scrivi';
    return '<div class="sel" '+attrs+' data-col="1" data-value="'+esc(value)+'">'+
      '<button type="button" class="sel-btn"><span class="sel-val'+(isFree ? ' free' : '')+'">'+esc(shown)+'</span>'+CHEV+'</button>'+
      '<div class="sel-menu">'+
        known.map(n => '<button type="button" data-opt="'+esc(n)+'"'+(n === value ? ' class="on"' : '')+'>'+esc(n)+'</button>').join('')+
        '<div class="sel-sep"></div>'+
        '<div class="sel-manual-label">oppure scrivi</div>'+
        '<input class="sel-manual" type="text" value="'+esc(isFree ? value : '')+'" placeholder="nome colonna">'+
      '</div>'+
    '</div>';
  }

  function closeSelects(except){
    // i menu staccati tornano nel loro campo prima di chiudersi
    document.querySelectorAll('.sel-menu.floating').forEach(m => {
      if (except && m._home === except) return;
      m.classList.remove('floating');
      m.removeAttribute('style');
      if (m._home && m._home.isConnected) m._home.appendChild(m); else m.remove();
    });
    inspInner.querySelectorAll('.sel.open').forEach(el => {
      if (el !== except){ el.classList.remove('open'); el.classList.remove('up'); }
    });
    inspInner.querySelectorAll('.menu').forEach(b => {
      if (!b.querySelector('.sel.open')) b.classList.remove('menu');
    });
  }
  function bindSelects(onChange){
    inspInner.querySelectorAll('.sel').forEach(sel => {
      const btn = sel.querySelector('.sel-btn');
      btn.addEventListener('click', (e) => {
        e.stopPropagation();
        const wasOpen = sel.classList.contains('open');
        closeSelects();
        if (wasOpen) return;
        sel.classList.add('open');
        // il menu esce dal pannello: nessun contenitore puo tagliarlo, e si dimensiona sulla finestra
        const menu = sel.querySelector('.sel-menu');
        menu._home = sel;
        document.body.appendChild(menu);
        menu.classList.add('floating');
        const r = btn.getBoundingClientRect();
        const inConn = !!sel.closest('.conn-row');
        const w = Math.max(r.width, inConn ? 250 : 0);
        menu.style.width = w + 'px';
        menu.style.left = Math.max(8, Math.min(window.innerWidth - w - 8, inConn ? r.left + r.width / 2 - w / 2 : r.left)) + 'px';
        const below = window.innerHeight - r.bottom - 14, above = r.top - 14;
        const want = Math.min(menu.scrollHeight, 340);
        if (below >= want || below >= above){
          menu.style.top = (r.bottom + 6) + 'px'; menu.style.bottom = 'auto';
          menu.style.maxHeight = Math.max(120, Math.min(340, below)) + 'px';
        } else {
          menu.style.bottom = (window.innerHeight - r.top + 6) + 'px'; menu.style.top = 'auto';
          menu.style.maxHeight = Math.max(120, Math.min(340, above)) + 'px';
        }
      });
      const man = sel.querySelector('.sel-manual');
      if (man){
        man.addEventListener('click', e => e.stopPropagation());
        man.addEventListener('pointerdown', e => e.stopPropagation());
        man.addEventListener('keydown', e => { if (e.key === 'Enter'){ e.preventDefault(); man.blur(); } });
        man.addEventListener('change', () => {
          const v = man.value.trim();
          if (!v) return;
          closeSelects();
          sel.dataset.value = v;
          onChange(sel, v);
        });
      }
      sel.querySelectorAll('.sel-menu button').forEach(opt => {
        opt.addEventListener('click', (e) => {
          e.stopPropagation();
          closeSelects();
          sel.dataset.value = opt.dataset.opt;
          onChange(sel, opt.dataset.opt);
        });
      });
    });
  }

  function esc(v){ return (v == null ? '' : String(v)).replace(/"/g,'&quot;').replace(/</g,'&lt;'); }

  // --- Selettore di valori: legge la colonna, si filtra scrivendo, accetta valori nuovi ---
  let pickerSeq = 0;
  let PICKERS = {};
  function splitTokens(t){ return Array.from(new Set(String(t || '').split(/[,;|\n]/).map(x => x.trim()).filter(Boolean))); }
  function pickerHtml(vobj, domain, changed){
    if (!Array.isArray(vobj.values)) vobj.values = [];
    // quanto era scritto nel vecchio campo manuale diventa una serie di valori
    if (vobj.mode !== 'list' && vobj.text && vobj.text.trim()){
      vobj.text.split(vobj.sep || ',').map(x => x.trim()).filter(Boolean)
        .forEach(x => { if (!vobj.values.includes(x)) vobj.values.push(x); });
      vobj.text = '';
    }
    vobj.mode = 'list';
    const id = 'vp' + (++pickerSeq);
    PICKERS[id] = { v: vobj, domain: (domain || []).map(String), changed };
    return '<div class="vpick" data-picker="' + id + '">' + pickerInner(id) + '</div>';
  }
  function pickerInner(id){
    const P = PICKERS[id], v = P.v;
    const extra = v.values.filter(x => !P.domain.includes(x));        // valori scritti a mano, assenti nei dati
    const all = P.domain.concat(extra);
    const n = v.values.length;
    return '<div class="vp-chips">' + (n ? v.values.map(x =>
        '<span class="vp-chip' + (P.domain.includes(x) ? '' : ' free') + '">' + esc(x) +
        '<button type="button" data-vprm="' + esc(x) + '" aria-label="Rimuovi ' + esc(x) + '">\u00d7</button></span>').join('')
        : '<span class="vp-none">Nessun valore scelto</span>') + '</div>' +
      '<input type="text" class="vp-search" placeholder="' + (P.domain.length ? 'Cerca tra i valori o scrivine di nuovi\u2026' : 'Scrivi i valori, anche pi\u00f9 insieme\u2026') + '">' +
      '<div class="vp-add" hidden></div>' +
      (all.length ? '<div class="vp-list">' + all.map(x =>
        '<label class="checkrow" data-vpv="' + esc(x) + '"><input type="checkbox" value="' + esc(x) + '"' + (v.values.includes(x) ? ' checked' : '') + '><span>' + esc(x) + '</span></label>').join('') + '</div>' : '') +
      '<div class="vp-foot"><span class="sel-count">' + n + ' selezionat' + (n === 1 ? 'o' : 'i') + (P.domain.length ? ' su ' + P.domain.length : '') + '</span>' +
      (P.domain.length ? '<span><button type="button" class="vp-btn" data-vpall>Tutti</button><button type="button" class="vp-btn" data-vpnone>Nessuno</button></span>' : '') + '</div>';
  }
  function filterPicker(el){
```


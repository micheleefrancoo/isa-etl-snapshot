# 10-prototype-f.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 6/6)

5098 righe totali

```html
    const r = stage.getBoundingClientRect();
    const d = { left: Math.abs(cx - r.left), right: Math.abs(r.right - cx), top: Math.abs(cy - r.top), bottom: Math.abs(r.bottom - cy) };
    return Object.keys(d).sort((a, b) => d[a] - d[b])[0];
  }
  Object.keys(PANELS).forEach(key => {
    const n = PANELS[key].notch;
    n.addEventListener('pointerdown', (e) => {
      e.preventDefault(); e.stopPropagation();
      const sx = e.clientX, sy = e.clientY;
      let moved = false;
      const mv = (ev) => {
        if (!moved && Math.hypot(ev.clientX - sx, ev.clientY - sy) < 5) return;
        if (!moved){ moved = true; n.classList.add('dragging'); }
        n.style.right = n.style.bottom = 'auto';
        n.style.left = (ev.clientX - 17) + 'px'; n.style.top = (ev.clientY - 17) + 'px';
        const side = nearestSide(ev.clientX, ev.clientY);
        edgeHint.className = 'edge-hint on e-' + side;
        hint.textContent = 'Rilascia per agganciare ' + PANELS[key].name + ' al bordo ' + SIDE_NAME[side];
      };
      const up = (ev) => {
        window.removeEventListener('pointermove', mv); window.removeEventListener('pointerup', up);
        edgeHint.className = 'edge-hint';
        hint.textContent = DEFAULT_HINT;
        if (!moved){ setPanelOpen(key, true); return; }         // un click apre soltanto
        n.classList.remove('dragging');
        const side = nearestSide(ev.clientX, ev.clientY);
        if (side === PANELS[key].side){ layoutNotches(); setPanelOpen(key, true); }
        else setSide(key, side);
      };
      window.addEventListener('pointermove', mv); window.addEventListener('pointerup', up);
    });
    n.addEventListener('click', (e) => e.stopPropagation());
  });
  function setToolbox(open){ setPanelOpen('tools', open); }

  paletteEl.addEventListener('click', (e) => {
    if (e.target.closest('#tbClose')){ setToolbox(false); return; }
    if (e.target.closest('#tbUpload')){ document.getElementById('dsFile').click(); return; }
    const head = e.target.closest('.tb-sec-head');
    if (head){
      const sec = head.closest('.tb-sec');
      secOpen[sec.dataset.sec] = !secOpen[sec.dataset.sec];
      sec.classList.toggle('open', secOpen[sec.dataset.sec]);    // nessun ridisegno: la vista resta ferma
    }
  });
  document.getElementById('dsFile').addEventListener('change', (e) => {
    const f = e.target.files && e.target.files[0];
    if (!f) return;
    const reader = new FileReader();
    reader.onload = () => {
      const parsed = parseCSV(String(reader.result || ''));
      if (!parsed || !parsed.columns.length){ hint.textContent = 'Il file non contiene colonne leggibili'; return; }
      libCounter++;
      LIBRARY.push({ id: 'lib-' + libCounter, name: f.name.replace(/\.[^.]+$/, ''), path: f.name,
                     columns: parsed.columns, rows: parsed.rows });
      secOpen.data = true;
      buildPalette();
      hint.textContent = f.name + ' caricato: ' + parsed.columns.length + ' colonne, ' + parsed.rows + ' righe. Trascinalo sul canvas.';
    };
    reader.readAsText(f);
    e.target.value = '';
  });

  paletteEl.addEventListener('pointerdown', (e) => {
    const item = e.target.closest('.pal-item');
    if (!item) return;
    e.preventDefault();
    const type = item.dataset.type;
    const spec = PALETTE.find(p => p.type === type);
    const ghost = document.createElement('div');
    ghost.className = 'ghost' + (spec.kind === 'dataset' ? ' source' : '');
    ghost.innerHTML = svgTag(type);
    document.body.appendChild(ghost);
    const place = (ev) => { ghost.style.left = (ev.clientX - CARD/2) + 'px'; ghost.style.top = (ev.clientY - CARD/2) + 'px'; };
    place(e);

    let hoverEl = null, hoverRel = null, palInsert = -1;
    function clearHover(){
      if (hoverEl) hoverEl.classList.remove('drop-target','link-target');
      hoverEl = null; hoverRel = null;
    }
    function onMove(ev){
      place(ev);
      ghost.style.display = 'none';                 // l'anteprima non deve nascondere cio che sta sotto
      const under = document.elementFromPoint(ev.clientX, ev.clientY);
      ghost.style.display = '';
      const tc = under ? under.closest('.card') : null;
      const rel = tc ? paletteRelation(spec.kind, tc.dataset.uid) : null;
      // una lavorazione dalla palette puo entrare direttamente in un collegamento
      const hitEl = (!tc && under) ? under.closest('.hit') : null;
      const li = (FEATURES.cableInsert && spec.kind === 'op' && hitEl) ? parseInt(hitEl.dataset.link, 10) : -1;
      const okIns = li >= 0 && insertable(li, '__nuovo__') ? li : -1;
      if (okIns !== palInsert){
        palInsert = okIns; hoverLinkIdx = okIns;
        if (okIns >= 0) hint.textContent = 'Rilascia per inserire la lavorazione nel collegamento';
      }
      if (tc !== hoverEl || rel !== hoverRel){
        clearHover();
        if (rel){
          hoverEl = tc; hoverRel = rel;
          tc.classList.add(rel === 'merge' ? 'drop-target' : 'link-target');
          hint.textContent = rel === 'merge'
            ? 'Rilascia per fondere direttamente nel box'
            : 'Rilascia per collegare';
        } else hint.textContent = DEFAULT_HINT;
      }
    }
    function onUp(ev){
      window.removeEventListener('pointermove', onMove);
      window.removeEventListener('pointerup', onUp);
      ghost.remove();
      const targetEl = hoverEl, rel = hoverRel;
      const insIdx = palInsert;
      clearHover();
      hoverLinkIdx = -1;
      hint.textContent = DEFAULT_HINT;

      const sr = stage.getBoundingClientRect();
      const inside = ev.clientX >= sr.left && ev.clientX <= sr.right && ev.clientY >= sr.top && ev.clientY <= sr.bottom;
      if (!inside && !targetEl) return;
      pushHistory();

      const uid = (spec.kind === 'dataset' ? 'ds-' : 'op-') + (uidCounter++);
      let name = META[type].label;
      const lib = item.dataset.lib ? LIBRARY.find(x => x.id === item.dataset.lib) : null;
      if (type === 'dataset'){
        if (lib) name = lib.name; else { dsCounter++; name = 'Dataset ' + dsCounter; }
      }
      const baseParams = () => {
        const p0 = defaultParams(type);
        if (lib){ p0.path = lib.path; p0.columns = lib.columns || SCHEMA; }
        return [p0];
      };

      // rilasciata su un collegamento: nasce gia inserita nel flusso
      if (insIdx >= 0 && !targetEl){
        const wp0 = toWorld(ev.clientX, ev.clientY);
        cards[uid] = { kind: spec.kind, components:[type], params:baseParams(), name, x: wp0.x - CARD/2, y: wp0.y - CARD/2 };
        const elI = createCardEl(uid);
        elI.style.transition='none'; elI.style.transform='scale(.5)'; elI.style.opacity='0';
        requestAnimationFrame(() => {
          elI.style.transition='transform .34s cubic-bezier(.34,1.56,.64,1), opacity .2s';
          elI.style.transform='scale(1)'; elI.style.opacity='1';
        });
        setTimeout(() => { elI.style.transition=''; elI.style.transform=''; }, 360);
        insertOnLink(insIdx, uid);
        return;
      }

      // rilasciato sopra un nodo: il nuovo elemento entra subito in fusione o si collega
      if (targetEl && rel){
        const tUid = targetEl.dataset.uid, t = cards[tUid];
        const start = rel === 'merge'
          ? { x: t.x, y: t.y }
          : freeSpot(Math.round((t.x + (rel === 'link' ? -150 : 150))/GRID)*GRID, t.y);
        cards[uid] = { kind: spec.kind, components:[type], params:baseParams(), name, x: start.x, y: start.y };
        if (layoutMode === 'grid' && rel !== 'merge'){ computeSlots(); cards[uid].slot = firstFreeSlot(start.x, start.y, uid); }
        const el = createCardEl(uid);
        if (rel === 'merge'){
          el.style.opacity = '0';                 // non compare mai da sola: viene assorbita
          performMerge(uid, tUid);
        } else {
          el.style.transition='none'; el.style.transform='scale(.5)'; el.style.opacity='0';
          requestAnimationFrame(() => {
            el.style.transition='transform .34s cubic-bezier(.34,1.56,.64,1), opacity .2s';
            el.style.transform='scale(1)'; el.style.opacity='1';
          });
          setTimeout(()=>{ el.style.transition=''; el.style.transform=''; }, 360);
          if (rel === 'link') connect(uid, tUid); else connect(tUid, uid);
          resolveOverlaps(uid, uid, true);
        }
        drawLinks();
        return;
      }

      const wp = toWorld(ev.clientX, ev.clientY);    // il punto di rilascio vive nel mondo
      const raw = { x: wp.x - CARD/2, y: wp.y - CARD/2 };
      const pos = freeSpot(Math.round(raw.x/GRID)*GRID, Math.round(raw.y/GRID)*GRID);
      cards[uid] = { kind: spec.kind, components:[type], params:baseParams(), name, x: pos.x, y: pos.y };
      if (layoutMode === 'grid'){ computeSlots(); cards[uid].slot = firstFreeSlot(pos.x, pos.y, uid); }
      const el = createCardEl(uid);
      el.style.transition='none'; el.style.transform='scale(.5)'; el.style.opacity='0';
      requestAnimationFrame(() => {
        el.style.transition='transform .34s cubic-bezier(.34,1.56,.64,1), opacity .2s';
        el.style.transform='scale(1)'; el.style.opacity='1';
      });
      setTimeout(()=>{ el.style.transition=''; el.style.transform=''; }, 360);
      resolveOverlaps(uid, uid, true);
      drawLinks();
    }
    window.addEventListener('pointermove', onMove);
    window.addEventListener('pointerup', onUp);
  });

  function init(){
    Array.from(stage.querySelectorAll('.card')).forEach(c=>c.remove());
    cards={}; linksArr=[]; uidCounter=0; comboCounter=0; outCounter=0; dsCounter=1;
    history=[]; redoStack=[]; updateHistoryButtons(); buildPalette();
    view.x = 0; view.y = 0; view.z = 1; applyView(false);
    layoutMode='free';
    Array.from(modeSeg.querySelectorAll('button')).forEach(b=>b.classList.toggle('on', b.dataset.mode==='free'));
    Object.keys(linkState).forEach(k=>delete linkState[k]);
    linkPaths.innerHTML='';
    cards['ds1']={kind:'dataset',components:['dataset'],params:[Object.assign(defaultParams('dataset'),{path:'vendite_2026.csv', columns: SCHEMA})],name:META.dataset.label,x:26,y:182};
    cards['op-filter']={kind:'op',components:['filter'],params:[defaultParams('filter')],name:META.filter.label,x:260,y:52};
    cards['op-join']={kind:'op',components:['join'],params:[defaultParams('join')],name:META.join.label,x:260,y:182};
    cards['op-sort']={kind:'op',components:['sort'],params:[defaultParams('sort')],name:META.sort.label,x:260,y:338};
    cards['op-export']={kind:'op',components:['exportOp'],params:[defaultParams('exportOp')],name:META.exportOp.label,x:442,y:338};
    Object.keys(cards).forEach(createCardEl);
    requestAnimationFrame(drawLinks);
  }
  init();
  layoutNotches();
  buildFeaturePanel();
  applyFeatures();
  lastStageW = stage.clientWidth;
  renderInspector();
  requestAnimationFrame(animateBubbles);
  window.addEventListener('resize', handleStageWidthChange);
</script>
</body>
</html>

```


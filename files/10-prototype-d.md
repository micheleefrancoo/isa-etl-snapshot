# 10-prototype-d.md

File in questo blocco:

- `docs/prototype/isa-fusion-prototype.html`

---

### `docs/prototype/isa-fusion-prototype.html` (parte 4/6)

5098 righe totali

```html
    const P = PICKERS[el.dataset.picker]; if (!P) return;
    const raw = el.querySelector('.vp-search').value, q = raw.trim().toLowerCase();
    el.querySelectorAll('.vp-list .checkrow').forEach(r => { r.style.display = !q || r.dataset.vpv.toLowerCase().includes(q) ? '' : 'none'; });
    const toks = splitTokens(raw).filter(t => !P.v.values.some(x => x.toLowerCase() === t.toLowerCase()));
    const onlyExisting = toks.length === 1 && P.domain.some(d => d.toLowerCase() === toks[0].toLowerCase());
    const add = el.querySelector('.vp-add');
    if (toks.length && !onlyExisting){
      add.hidden = false;
      add.innerHTML = '<button type="button" class="vp-addbtn">+ Aggiungi ' + toks.map(t => '\u201c' + esc(t) + '\u201d').join(', ') + '</button>';
    } else add.hidden = true;
  }
  function refreshPicker(el){
    const P = PICKERS[el.dataset.picker]; if (!P) return;
    const s0 = el.querySelector('.vp-search'), q = s0 ? s0.value : '', focused = document.activeElement === s0;
    const l0 = el.querySelector('.vp-list'), st = l0 ? l0.scrollTop : 0;
    el.innerHTML = pickerInner(el.dataset.picker);
    const s1 = el.querySelector('.vp-search'); s1.value = q;
    const l1 = el.querySelector('.vp-list'); if (l1) l1.scrollTop = st;
    filterPicker(el);
    if (focused) s1.focus();
    if (P.changed) P.changed(el);
  }
  function addTokens(el){
    const P = PICKERS[el.dataset.picker]; if (!P) return;
    const inp = el.querySelector('.vp-search');
    splitTokens(inp.value).forEach(t => {
      const match = P.domain.find(d => d.toLowerCase() === t.toLowerCase());   // un valore esistente si usa com'e nei dati
      const val = match || t;
      if (!P.v.values.includes(val)) P.v.values.push(val);
    });
    inp.value = '';
    refreshPicker(el);
  }
  document.addEventListener('input', (e) => { const el = e.target.closest && e.target.closest('.vpick'); if (el && e.target.classList.contains('vp-search')) filterPicker(el); });
  document.addEventListener('keydown', (e) => {
    const el = e.target.closest && e.target.closest('.vpick');
    if (el && e.target.classList.contains('vp-search') && e.key === 'Enter'){ e.preventDefault(); addTokens(el); }
  });
  document.addEventListener('change', (e) => {
    const el = e.target.closest && e.target.closest('.vpick');
    if (!el || !e.target.closest('.vp-list')) return;
    const P = PICKERS[el.dataset.picker]; if (!P) return;
    const v = e.target.value;
    if (e.target.checked){ if (!P.v.values.includes(v)) P.v.values.push(v); }
    else P.v.values = P.v.values.filter(x => x !== v);
    refreshPicker(el);
  });
  document.addEventListener('click', (e) => {
    const el = e.target.closest && e.target.closest('.vpick');
    if (!el) return;
    const P = PICKERS[el.dataset.picker]; if (!P) return;
    const rm = e.target.closest('[data-vprm]');
    if (rm){ P.v.values = P.v.values.filter(x => x !== rm.dataset.vprm); refreshPicker(el); return; }
    if (e.target.closest('[data-vpall]')){
      P.domain.forEach(d => { if (!P.v.values.includes(d)) P.v.values.push(d); }); refreshPicker(el); return;
    }
    if (e.target.closest('[data-vpnone]')){ P.v.values = []; refreshPicker(el); return; }
    if (e.target.closest('.vp-addbtn')) addTokens(el);
  });
  // aggiorna il riassunto della voce a cui appartiene il selettore
  function sumUpdater(fn){
    return (el) => {
      const cnd = condOf(el); if (!cnd) return;
      const txt = cnd.querySelector('.cond-txt'), sm = fn();
      txt.textContent = sm || 'da configurare';
      txt.classList.toggle('empty', !sm);
      if (cnd.classList.contains('open') && typeof updateMdTitle === 'function') updateMdTitle(cnd);
    };
  }

  function valueControl(c, i){
    if (NO_VALUE_OPS.includes(c.op)) return '';
    const def = columnDef(c.column);
    const multi = MULTI_OPS.includes(c.op);
    if (!multi){
      return '<div class="field"><label>Valore</label>'+
        '<input type="text" data-cond="'+i+'" data-f="text" value="'+esc(c.text)+'" placeholder="valore singolo"></div>';
    }
    return '<div class="field"><label>Valori</label>' +
      pickerHtml(c, def ? def.values : [], sumUpdater(() => summarizeCond(c))) + '</div>';
  }

  let openCond = 0;
  let openRows = {};              // voce aperta per ciascuna lista di un'operazione a righe multiple
  let activeMulti = null;
  // un solo controllo per scegliere dall'elenco dei valori o scriverne uno nuovo
  function valueSelect(options, value, attrs){
    const isFree = value && !options.includes(value);
    return '<div class="sel" ' + attrs + ' data-value="' + esc(value) + '">' +
      '<button type="button" class="sel-btn"><span class="sel-val' + (isFree ? ' free' : '') + '">' + esc(value || 'scegli o scrivi') + '</span>' + CHEV + '</button>' +
      '<div class="sel-menu">' +
        options.map(o => '<button type="button" data-opt="' + esc(o) + '"' + (o === value ? ' class="on"' : '') + '>' + esc(o) + '</button>').join('') +
        '<div class="sel-sep"></div><div class="sel-manual-label">oppure scrivi</div>' +
        '<input class="sel-manual" type="text" value="' + esc(isFree ? value : '') + '" placeholder="valore">' +
      '</div></div>';
  }
  function mlField(f, val, L, i, row){
    const attrs = 'data-ml="' + L.key + '" data-mi="' + i + '" data-mf="' + f.k + '"';
    const dom = row && row.column ? columnDef(row.column) : null;
    const domain = dom && dom.values.length ? dom.values : [];
    if (f.type === 'value'){
      return '<div class="field"><label>' + f.label + '</label>' +
        (domain.length ? valueSelect(domain, val || '', attrs)
                       : '<input type="text" ' + attrs + ' value="' + esc(val) + '" placeholder="valore">') + '</div>';
    }
    if (f.type === 'values'){
      const v = val || (row[f.k] = VALUES_DEF());
      return '<div class="field"><label>' + f.label + '</label>' +
        pickerHtml(v, domain, sumUpdater(() => L.sum(row))) + '</div>';
    }
    if (f.type === 'column') return '<div class="field"><label>' + f.label + '</label>' + columnSelect(val || '', attrs) + '</div>';
    if (f.type === 'select') return '<div class="field"><label>' + f.label + '</label>' + selectHtml(f.opts, val || f.def, attrs) + '</div>';
    return '<div class="field"><label>' + f.label + '</label><input type="text" ' + attrs + ' value="' + esc(val) + '"></div>';
  }
  function renderMulti(type, par){
    const md = MULTI_DEFS[type];
    ensureMulti(type, par);
    let html = (md.globals || []).map(f => fieldHtml(f, par[f.k], selectedStep)).join('');
    md.lists.forEach(L => {
      const rows = par[L.key];
      const open = openRows[L.key] === undefined ? 0 : openRows[L.key];
      html += '<div class="cond-area">';
      html += '<div class="field"><label>' + L.label + '</label></div>';
      html += rows.map((r, i) => {
        const sum = L.sum(r);
        return '<div class="cond' + (i === open ? ' open' : '') + '" data-ml-row="' + L.key + '">' +
          '<div class="cond-head">' +
            '<button class="cond-toggle" data-mlopen="' + L.key + '" data-mi="' + i + '">' +
              '<span class="cond-chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg></span>' +
              '<span class="cond-sum"><span class="cond-n">' + L.noun + ' ' + (i + 1) + '</span>' +
              '<span class="cond-txt' + (sum ? '' : ' empty') + '">' + esc(sum || 'da configurare') + '</span></span>' +
            '</button>' +
            (rows.length > 1 ? '<button class="cond-del" data-mldel="' + L.key + '" data-mi="' + i + '" aria-label="Rimuovi voce"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><line x1="17" y1="7" x2="7" y2="17"/><line x1="7" y1="7" x2="17" y2="17"/></svg></button>' : '') +
          '</div>' +
          '<div class="cond-body">' + L.fields.map(f => mlField(f, r[f.k], L, i, r)).join('') + '</div>' +
        '</div>';
      }).join('');
      html += '<button class="add-cond" data-mladd="' + L.key + '">+ ' + L.add + '</button>';
      if (L.note && rows.length > 1) html += '<div class="insp-note">' + L.note + '</div>';
      html += '</div>';
    });
    return html;
  }
  function multiSelectChange(sel, value){
    if (!activeMulti) return;
    const list = activeMulti.par[sel.dataset.ml];
    const i = parseInt(sel.dataset.mi, 10);
    if (!list || !list[i]) return;
    const mf = sel.dataset.mf;
    if (mf.endsWith('__sep')){ list[i][mf.slice(0, -5)].sep = value; renderInspector(); return; }
    list[i][mf] = value;
    if (mf === 'column'){
      // cambiata la colonna, i valori scelti dal vecchio dominio non valgono piu
      const L = MULTI_DEFS[activeMulti.type].lists.find(x => x.key === sel.dataset.ml);
      L.fields.forEach(f => { if (f.type === 'values' && list[i][f.k]) list[i][f.k].values = []; });
    }
    renderInspector();
  }
  function bindMulti(type, par){
    activeMulti = { type, par };
    const md = MULTI_DEFS[type];
    inspInner.querySelectorAll('[data-mlopen]').forEach(el =>
      el.addEventListener('click', () => {
        const key = el.dataset.mlopen, i = parseInt(el.dataset.mi, 10);
        const cur = openRows[key] === undefined ? 0 : openRows[key];
        const willOpen = cur !== i;
        openRows[key] = willOpen ? i : -1;
        // nessun ridisegno: la voce si anima e la vista resta ferma
        inspInner.querySelectorAll('[data-ml-row="' + key + '"]').forEach(c => c.classList.remove('open'));
        if (willOpen){
          const box = el.closest('.cond');
          box.classList.add('open');
          setTimeout(() => box.scrollIntoView({ block:'nearest', behavior:'smooth' }), 120);
        }
      }));
    inspInner.querySelectorAll('[data-mladd]').forEach(el =>
      el.addEventListener('click', () => {
        const L = md.lists.find(x => x.key === el.dataset.mladd);
        par[L.key].push(blankRow(L));
        openRows[L.key] = par[L.key].length - 1;
        renderInspector();
      }));
    inspInner.querySelectorAll('[data-mldel]').forEach(el =>
      el.addEventListener('click', () => {
        const key = el.dataset.mldel;
        par[key].splice(parseInt(el.dataset.mi, 10), 1);
        if ((openRows[key] || 0) >= par[key].length) openRows[key] = par[key].length - 1;
        renderInspector();
      }));
    const rowOf = (el) => {
      const L = md.lists.find(x => x.key === el.dataset.ml);
      return { L, row: par[L.key][parseInt(el.dataset.mi, 10)] };
    };
    const refreshSum = (el, L, row) => {
      const cnd = condOf(el); if (!cnd) return;
      const txt = cnd.querySelector('.cond-txt');
      const sum = L.sum(row);
      txt.textContent = sum || 'da configurare';
      txt.classList.toggle('empty', !sum);
      if (cnd.classList.contains('open')) updateMdTitle(cnd);
    };
    inspInner.querySelectorAll('[data-mlmode]').forEach(el =>
      el.addEventListener('click', () => {
        const { row } = rowOf(el);
        row[el.dataset.mf].mode = el.dataset.mlmode;
        renderInspector();
      }));
    inspInner.querySelectorAll('input[data-mlchk]').forEach(el =>
      el.addEventListener('change', () => {
        const { L, row } = rowOf(el);
        const v = row[el.dataset.mf];
        if (el.checked){ if (!v.values.includes(el.value)) v.values.push(el.value); }
        else v.values = v.values.filter(x => x !== el.value);
        const cnt = el.closest('.field').querySelector('.sel-count');
        if (cnt) cnt.textContent = v.values.length + ' selezionat' + (v.values.length === 1 ? 'o' : 'i');
        refreshSum(el, L, row);
      }));
    inspInner.querySelectorAll('input[data-mltext]').forEach(el =>
      el.addEventListener('input', () => {
        const { L, row } = rowOf(el);
        row[el.dataset.mf].text = el.value;
        refreshSum(el, L, row);
      }));
    inspInner.querySelectorAll('input[data-ml]:not([data-mlchk]):not([data-mltext])').forEach(el => {
      el.addEventListener('input', () => {
        const L = md.lists.find(x => x.key === el.dataset.ml);
        const row = par[L.key][parseInt(el.dataset.mi, 10)];
        row[el.dataset.mf] = el.value;
        // il riassunto si aggiorna mentre scrivi, senza perdere il cursore
        const cnd = condOf(el); if (!cnd) return;
      const txt = cnd.querySelector('.cond-txt');
        const sum = L.sum(row);
        txt.textContent = sum || 'da configurare';
        txt.classList.toggle('empty', !sum);
      });
    });
  }
  let pendingStepAnim = null;   // intervallo dei passaggi appena entrati nella sequenza
  let seqOpen = true;           // la sequenza si comprime per lasciare spazio ai parametri
  function summarizeCond(c){
    if (!c.column) return null;
    if (NO_VALUE_OPS.includes(c.op)) return c.column + ' ' + c.op;
    let v = '';
    if (MULTI_OPS.includes(c.op)){
      const def = columnDef(c.column);
      const hasList = def && def.values.length > 0;
      if (hasList && c.mode === 'list') v = c.values.length ? c.values.join(', ') : '';
      else v = c.text;
    } else v = c.text;
    if (!v) return c.column + ' ' + c.op + ' …';
    return c.column + ' ' + c.op + ' ' + v;
  }

  // una join puo combinare piu colonne: le chiavi sono coppie sinistra/destra
  function ensureKeys(par){
    if (!par.keys){
      par.keys = (par.leftKey || par.rightKey)
        ? [ { left: par.leftKey || '', right: par.rightKey || '' } ]     // vecchio formato a chiave singola
        : [ { left:'', right:'' } ];
      delete par.leftKey; delete par.rightKey;
    }
    // ogni lato di una condizione puo essere una colonna, un valore fisso o (a destra) una lista
    par.keys.forEach(k => {
      if (!k.op) k.op = '=';
      if (!k.lmode) k.lmode = 'col';
      if (!k.rmode) k.rmode = 'col';
      if (!k.rlist || typeof k.rlist !== 'object') k.rlist = VALUES_DEF();
      if (k.lval === undefined) k.lval = '';
      if (k.rval === undefined) k.rval = '';
    });
    return par.keys;
  }
  const LIST_OPS = ['\u00e8 uno di', 'non \u00e8 uno di'];
  function sideText(k, side){
    if (side === 'l') return k.lmode === 'val' ? (k.lval ? '\u201c' + k.lval + '\u201d' : '') : k.left;
    if (k.rmode === 'val') return k.rval ? '\u201c' + k.rval + '\u201d' : '';
    if (k.rmode === 'list'){ const t = valuesText(k.rlist); return t ? '(' + t + ')' : ''; }
    return k.right;
  }
  function keyComplete(k){
    const l = k.lmode === 'val' ? (k.lval && String(k.lval).trim()) : k.left;
    const r = k.rmode === 'val' ? (k.rval && String(k.rval).trim())
            : k.rmode === 'list' ? fieldFilled({ type:'values' }, k.rlist) : k.right;
    return !!(l && r);
  }
  // il dominio da proporre e quello della colonna sull'altro lato del confronto
  function domainOf(colName){ const d = colName ? columnDef(colName) : null; return d && d.values.length ? d.values : []; }
  let openKey = 0;
  const JOIN_OPS = ['=', '\u2260', '<', '\u2264', '>', '\u2265'];
  const JOIN_OP_NAME = ['uguale a', 'diverso da', 'minore di', 'minore o uguale a', 'maggiore di', 'maggiore o uguale a'];
  function jkModeSeg(i, side, modes, cur){
    const names = { col:'Colonna', val:'Valore', list:'Lista' };
    return '<div class="mode-seg">' + modes.map(m =>
      '<button type="button" data-jkmode="' + i + '" data-jside="' + side + '" data-m="' + m + '" class="' + (cur === m ? 'on' : '') + '">' + names[m] + '</button>').join('') + '</div>';
  }
  function joinSideHtml(k, i, side){
    const isL = side === 'l';
    const mode = isL ? k.lmode : k.rmode;
    let h = '<div class="field"><label>' + (isL ? 'Lato sinistro' : 'Lato destro') + '</label>' +
      jkModeSeg(i, side, isL ? ['col', 'val'] : ['col', 'val', 'list'], mode);
    const other = isL ? (k.rmode === 'col' ? k.right : '') : (k.lmode === 'col' ? k.left : '');
    const dom = domainOf(other);
    if (mode === 'col') h += columnSelect(isL ? k.left : k.right, 'data-jk="' + i + '" data-side="' + (isL ? 'left' : 'right') + '"');
    else if (mode === 'val'){
      const v = isL ? k.lval : k.rval, attrs = 'data-jk="' + i + '" data-side="' + (isL ? 'lval' : 'rval') + '"';
      h += dom.length ? valueSelect(dom, v || '', attrs)
                      : '<input type="text" data-jkval="' + i + '" data-jside="' + side + '" value="' + esc(v) + '" placeholder="valore">';
    } else {
      h += '<div style="margin-top:6px">' + pickerHtml(k.rlist, dom, sumUpdater(() => summarizeKey(k))) + '</div>';
    }
    return h + '</div>';
  }
  // con una lista a destra il confronto diventa di appartenenza; altrimenti di valore
  function joinOpHtml(k, i){
    const isList = k.rmode === 'list';
    const ops = isList ? LIST_OPS : JOIN_OPS;
    const names = isList ? LIST_OPS : JOIN_OP_NAME.map((n, j) => JOIN_OPS[j] + '  ' + n);
    return '<div class="field"><label>Confronto</label>' + selectHtml(ops, k.op, 'data-jk="' + i + '" data-side="op"', names) + '</div>';
  }

  function summarizeKey(k){
    const l = sideText(k, 'l'), r = sideText(k, 'r');
    if (!l && !r) return null;
    return (l || '\u2026') + ' ' + (k.op || '=') + ' ' + (r || '\u2026');
  }
  function renderJoinKeys(par){
    const keys = ensureKeys(par);
    normalizeGroups(keys);
    if (openKey >= keys.length) openKey = keys.length - 1;
    let html = '<div class="field"><label>Condizioni di unione</label></div>';
    const one = (k, i) => {
      const isOpen = i === openKey;
      const sum = summarizeKey(k);
      return '<div class="cond'+(isOpen ? ' open' : '')+'" data-key-row="'+i+'">'+
        '<div class="cond-head">'+
          '<button class="cond-toggle" data-openkey="'+i+'">'+
            '<span class="cond-chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg></span>'+
            '<span class="cond-sum"><span class="cond-n">Condizione '+(i+1)+'</span>'+
              '<span class="cond-txt'+(sum?'':' empty')+'">'+esc(sum || 'da configurare')+'</span></span>'+
          '</button>'+
          (keys.length > 1 ? '<button class="cond-del" data-jkdel="'+i+'" aria-label="Rimuovi chiave"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><line x1="17" y1="7" x2="7" y2="17"/><line x1="7" y1="7" x2="17" y2="17"/></svg></button>' : '')+
        '</div>'+
        '<div class="cond-body">'+ joinSideHtml(k, i, 'l') + joinOpHtml(k, i) + joinSideHtml(k, i, 'r') +'</div>'+
      '</div>';
    };
    html += assembleGrouped(keys, one, 'k', i => 'data-kconn="' + i + '"');
    html += '<button class="add-cond" data-addkey="1">+ Aggiungi condizione</button>';
    html += groupedPreview(keys, k => summarizeKey(k) || '\u2026');
    // senza nessuna uguaglianza ogni riga va confrontata con ogni altra: su tabelle grandi e molto lento
    const configured = keys.filter(k => keyComplete(k));
    const equiKey = k => k.lmode === 'col' && k.rmode === 'col' && (k.op || '=') === '=';
    if (configured.length && !configured.some(equiKey))
      html += '<div class="insp-warn">Nessuna condizione di uguaglianza: il Join confronterà ogni riga con tutte le altre. Su tabelle grandi l\u2019esecuzione può essere molto lenta.</div>';
    return '<div class="cond-area">' + html + '</div>';
  }

  let activeJoinPar = null;
  function joinSelectChange(sel, value){
    const par = activeJoinPar;
    if (!par) return;
    const i = parseInt(sel.dataset.jk, 10);
    if (sel.dataset.side === 'rsep') par.keys[i].rlist.sep = value;
    else par.keys[i][sel.dataset.side] = value;
    renderInspector();
  }
  function bindJoin(par){
    activeJoinPar = par;
    const refreshKeySum = (el, k) => {
      const c = condOf(el); if (!c) return;
      const txt = c.querySelector('.cond-txt'), sum = summarizeKey(k);
      txt.textContent = sum || 'da configurare'; txt.classList.toggle('empty', !sum);
      if (c.classList.contains('open')) updateMdTitle(c);
    };
    inspInner.querySelectorAll('[data-jkmode]').forEach(el =>
      el.addEventListener('click', () => {
        const k = par.keys[parseInt(el.dataset.jkmode, 10)], m = el.dataset.m;
        if (el.dataset.jside === 'l') k.lmode = m;
        else {
          const wasList = k.rmode === 'list';
          k.rmode = m;
          // il confronto segue il tipo del lato destro
          if (m === 'list' && !LIST_OPS.includes(k.op)) k.op = LIST_OPS[0];
          if (m !== 'list' && wasList) k.op = '=';
        }
        renderInspector();
      }));
    inspInner.querySelectorAll('[data-jklm]').forEach(el =>
      el.addEventListener('click', () => { par.keys[parseInt(el.dataset.jklm, 10)].rlist.mode = el.dataset.m; renderInspector(); }));
    inspInner.querySelectorAll('input[data-jkchk]').forEach(el =>
      el.addEventListener('change', () => {
        const k = par.keys[parseInt(el.dataset.jkchk, 10)], L = k.rlist;
        if (el.checked){ if (!L.values.includes(el.value)) L.values.push(el.value); }
        else L.values = L.values.filter(v => v !== el.value);
        const cnt = el.closest('.field').querySelector('.sel-count');
        if (cnt) cnt.textContent = L.values.length + ' selezionat' + (L.values.length === 1 ? 'o' : 'i');
        refreshKeySum(el, k);
      }));
    inspInner.querySelectorAll('input[data-jktext]').forEach(el =>
      el.addEventListener('input', () => { const k = par.keys[parseInt(el.dataset.jktext, 10)]; k.rlist.text = el.value; refreshKeySum(el, k); }));
    inspInner.querySelectorAll('input[data-jkval]').forEach(el =>
      el.addEventListener('input', () => {
        const k = par.keys[parseInt(el.dataset.jkval, 10)];
        if (el.dataset.jside === 'l') k.lval = el.value; else k.rval = el.value;
        refreshKeySum(el, k);
      }));
    inspInner.querySelectorAll('.cond-toggle[data-openkey]').forEach(el =>
      el.addEventListener('click', () => {
        const i = parseInt(el.dataset.openkey, 10);
        const box = el.closest('.cond');
        const willOpen = openKey !== i;
        openKey = willOpen ? i : -1;
        // si anima la sezione, la vista resta dov'e
        inspInner.querySelectorAll('[data-key-row]').forEach(c => c.classList.remove('open'));
        if (willOpen){
          box.classList.add('open');
          setTimeout(() => box.scrollIntoView({ block:'nearest', behavior:'smooth' }), 120);
        }
      }));
    inspInner.querySelectorAll('[data-addkey]').forEach(el =>
      el.addEventListener('click', () => { ensureKeys(par).push({ left:'', op:'=', right:'', conn:'AND' }); ensureKeys(par); openKey = par.keys.length - 1; renderInspector(); }));
    inspInner.querySelectorAll('[data-jkdel]').forEach(el =>
      el.addEventListener('click', () => {
        par.keys.splice(parseInt(el.dataset.jkdel, 10), 1);
        if (openKey >= par.keys.length) openKey = par.keys.length - 1;
        renderInspector();
      }));
  }

  // connettori logici: ogni condizione dalla seconda in poi dice come si combina con cio che la precede
  const LOGIC_OPS = ['AND', 'OR', 'XOR', 'NAND', 'NOR', 'XNOR'];
  const LOGIC_HELP = {
    AND:'entrambe vere', OR:'almeno una vera', XOR:'una sola delle due vera',
    NAND:'non entrambe vere', NOR:'nessuna delle due vera', XNOR:'entrambe vere o entrambe false'
  };
  function connRow(conn, attrs){
    return '<div class="conn-row"><span class="conn-line"></span>' +
      selectHtml(LOGIC_OPS, conn || 'AND', attrs, LOGIC_OPS.map(o => o + ' \u00b7 ' + LOGIC_HELP[o])) +
      '<span class="conn-line"></span></div>';
  }
  // gruppi: condizioni contigue con lo stesso identificativo si valutano insieme, tra parentesi
  let groupSeq = 0;
  function newGroupId(){ groupSeq++; return 'g' + groupSeq + '-' + Date.now().toString(36).slice(-4); }
  function groupRuns(list){
    const runs = [];
    let i = 0;
    while (i < list.length){
      const g = list[i].g;
      let j = i;
      if (g) while (j + 1 < list.length && list[j + 1].g === g) j++;
      runs.push({ s: i, e: j, g: g || null });
      i = j + 1;
    }
    return runs;
  }
  // un gruppo di una sola condizione non ha senso: si scioglie
  function normalizeGroups(list){
    const cnt = {};
    list.forEach(x => { if (x.g) cnt[x.g] = (cnt[x.g] || 0) + 1; });
    list.forEach(x => { if (x.g && cnt[x.g] < 2) delete x.g; });
  }
  function groupPair(list, i){
    const a = list[i - 1], b = list[i];
    if (!a || !b) return;
    const g = a.g || b.g || newGroupId();
    const other = (a.g && b.g && a.g !== b.g) ? b.g : null;
    a.g = g; b.g = g;
    if (other) list.forEach(x => { if (x.g === other) x.g = g; });   // due gruppi adiacenti si fondono
  }
  function splitAt(list, i){
    const g = list[i] && list[i].g;
    if (!g) return;
    const ng = newGroupId();
    for (let k = i; k < list.length && list[k].g === g; k++) list[k].g = ng;
  }
  function connRowG(conn, attrs, prefix, i, inside){
    const btn = inside
      ? '<button type="button" class="grp-btn" data-' + prefix + 'split="' + i + '" title="Dividi il gruppo in questo punto">)(</button>'
      : '<button type="button" class="grp-btn" data-' + prefix + 'grp="' + i + '" title="Raggruppa le due condizioni">( )</button>';
    return '<div class="conn-row"><span class="conn-line"></span>' +
      selectHtml(LOGIC_OPS, conn || 'AND', attrs, LOGIC_OPS.map(o => o + ' \u00b7 ' + LOGIC_HELP[o])) +
      btn + '<span class="conn-line"></span></div>';
  }
  function assembleGrouped(list, one, prefix, attrsFor){
    let html = '';
    groupRuns(list).forEach(r => {
      if (r.s > 0) html += connRowG(list[r.s].conn, attrsFor(r.s), prefix, r.s, false);
      if (r.g) html += '<div class="cgroup"><div class="cgroup-head"><span>Gruppo</span>' +
        '<button type="button" data-' + prefix + 'ungroup="' + r.g + '">Sciogli</button></div>';
      for (let k = r.s; k <= r.e; k++){
        if (k > r.s) html += connRowG(list[k].conn, attrsFor(k), prefix, k, true);
        html += one(list[k], k);
      }
      if (r.g) html += '<button type="button" class="add-in-group" data-' + prefix + 'addin="' + r.g + '">+ Condizione nel gruppo</button></div>';
    });
    return html;
  }
  function leftAssoc(parts, conns){
    let expr = parts[0];
    for (let i = 1; i < parts.length; i++) expr = (i > 1 ? '(' + expr + ')' : expr) + ' ' + (conns[i] || 'AND') + ' ' + parts[i];
    return expr;
  }
  function groupedPreview(list, sumFn){
    const runs = groupRuns(list);
    if (list.length < 2) return '';
    const parts = runs.map(r => {
      if (!r.g) return sumFn(list[r.s]);
      const inner = [], ic = [];
      for (let k = r.s; k <= r.e; k++){ inner.push(sumFn(list[k])); ic.push(k === r.s ? null : list[k].conn); }
      return '(' + leftAssoc(inner, ic) + ')';
    });
    const expr = leftAssoc(parts, runs.map(r => list[r.s].conn));
    return '<div class="insp-note logic-prev"><b>Anteprima</b>' + esc(expr) +
      '<span class="lp-hint">Tra parentesi i gruppi; per il resto le condizioni si combinano nell\u2019ordine in cui compaiono.</span></div>';
  }

  // la valutazione procede da sinistra a destra: l'anteprima lo rende esplicito con le parentesi
  function logicPreview(parts, conns){
    if (parts.length < 2) return '';
    let expr = parts[0];
    for (let i = 1; i < parts.length; i++) expr = (i > 1 ? '(' + expr + ')' : expr) + ' ' + (conns[i] || 'AND') + ' ' + parts[i];
    return '<div class="insp-note logic-prev"><b>Anteprima</b>' + esc(expr) +
      '<span class="lp-hint">Le condizioni si combinano nell\u2019ordine in cui compaiono.</span></div>';
  }

  function renderFilter(par){
    const conds = par.conditions || (par.conditions = [ newCondition() ]);
    // il vecchio selettore globale diventa il connettore di ogni condizione
    if (par.logic){
      conds.forEach((c, i) => { if (i > 0 && !c.conn) c.conn = par.logic === 'O' ? 'OR' : 'AND'; });
      delete par.logic;
    }
    normalizeGroups(conds);
    if (openCond >= conds.length) openCond = conds.length - 1;
    let html = '';
    html += '<datalist id="colList">'+SCHEMA.map(c => '<option value="'+c.name+'">').join('')+'</datalist>';

    const one = (c, i) => {
      // ogni condizione si apre e si richiude, anche quando e l'unica
      const isOpen = i === openCond;
      const def = columnDef(c.column);
      const sum = summarizeCond(c);
      const body = '<div class="cond-body">'+
        '<div class="field"><label>Colonna</label>'+
          columnSelect(c.column, 'data-cond="'+i+'" data-f="column"')+
          (def ? '<div class="col-type">tipo: '+def.type+'</div>' : '')+
        '</div>'+
        '<div class="field"><label>Operatore</label>'+
          selectHtml(FILTER_OPS, c.op, 'data-cond="'+i+'" data-f="op"')+
        '</div>'+
        valueControl(c, i)+
      '</div>';

      return '<div class="cond'+(isOpen?' open':'')+'">'+
        '<div class="cond-head">'+
          '<button class="cond-toggle" data-open="'+i+'">'+
            '<span class="cond-chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg></span>'+
            '<span class="cond-sum"><span class="cond-n">Condizione '+(i+1)+'</span>'+
              '<span class="cond-txt'+(sum?'':' empty')+'">'+esc(sum || 'da configurare')+'</span></span>'+
          '</button>'+
          (conds.length > 1 ? '<button class="cond-del" data-cond="'+i+'" aria-label="Rimuovi condizione"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><line x1="17" y1="7" x2="7" y2="17"/><line x1="7" y1="7" x2="17" y2="17"/></svg></button>' : '')+
        '</div>'+ body +
      '</div>';
    };
    html += assembleGrouped(conds, one, 'c', i => 'data-cconn="' + i + '"');

    html += '<button class="add-cond" data-add="1">+ Aggiungi condizione</button>';
    html += groupedPreview(conds, c => summarizeCond(c) || '\u2026');
    return '<div class="cond-area">' + html + '</div>';
  }

  let activeFilterPar = null;
  function filterSelectChange(sel, value){
    const par = activeFilterPar;
    if (!par) return;
    if (sel.dataset.logic !== undefined){
      par.logic = value;
      const lbl = sel.querySelector('.sel-val'); if (lbl) lbl.textContent = value;
      return;
    }
    const i = parseInt(sel.dataset.cond, 10);
    const f = sel.dataset.f;
    par.conditions[i][f] = value;
    if (f === 'column') par.conditions[i].values = [];
    renderInspector();                 // operatore e separatore cambiano la forma del campo valore
  }

  function bindFilter(par){
    activeFilterPar = par;
    const rerender = () => renderInspector();
    inspInner.querySelectorAll('.cond-toggle').forEach(el =>
      el.addEventListener('click', () => {
        const i = parseInt(el.dataset.open, 10);
        const box = el.closest('.cond');
        const willOpen = openCond !== i;
        openCond = willOpen ? i : -1;
        // nessun ridisegno: si anima la sezione e la vista resta dov'e
        inspInner.querySelectorAll('.cond').forEach(c => c.classList.remove('open'));
        if (willOpen){
          box.classList.add('open');
          setTimeout(() => box.scrollIntoView({ block:'nearest', behavior:'smooth' }), 120);
        }
      }));
    inspInner.querySelectorAll('.add-cond').forEach(el =>
      el.addEventListener('click', () => { par.conditions.push(newCondition()); openCond = par.conditions.length - 1; rerender(); }));
    inspInner.querySelectorAll('.cond-del').forEach(el =>
      el.addEventListener('click', () => {
        const i = parseInt(el.dataset.cond,10);
        par.conditions.splice(i,1);
        if (openCond >= par.conditions.length) openCond = par.conditions.length - 1;
        rerender();
      }));
    inspInner.querySelectorAll('.mode-seg button').forEach(el =>
      el.addEventListener('click', () => {
        par.conditions[parseInt(el.parentElement.dataset.cond,10)].mode = el.dataset.mode;
        rerender();
      }));
    inspInner.querySelectorAll('[data-f]').forEach(el => {
      const i = parseInt(el.dataset.cond, 10);
      const f = el.dataset.f;
      if (f === 'val'){
        el.addEventListener('change', () => {
          const c = par.conditions[i];
          if (el.checked){ if (!c.values.includes(el.value)) c.values.push(el.value); }
          else c.values = c.values.filter(v => v !== el.value);
          const cnt = el.closest('.field').querySelector('.sel-count');
          if (cnt) cnt.textContent = c.values.length + ' selezionat' + (c.values.length===1?'o':'i');
        });
        return;
      }
      if (f === 'text'){
        el.addEventListener('input', () => { par.conditions[i].text = el.value; });   // niente ridisegno: non perdi il fuoco
        return;
      }
      el.addEventListener('change', () => {
        par.conditions[i][f] = el.value;
        if (f === 'column') par.conditions[i].values = [];   // cambiata la colonna, i valori scelti non valgono piu
        rerender();
      });
    });
  }

  function setCollapsed(v){ setPanelOpen('insp', !v); }

  // il pannello si ricostruisce a ogni modifica: la posizione di lettura non deve perdersi
  // il pannello riflette sempre lo stato corrente del nodo selezionato
  function syncInspector(){
    [...selectedSet].forEach(id => { if (!cards[id]) selectedSet.delete(id); });
    if (!selectedUid) return;
    if (!cards[selectedUid]){
      if (selectedSet.size){ selectedUid = [...selectedSet][0]; }
      else { deselect(); return; }
    }
    const d = cards[selectedUid];
    if (d.components && selectedStep >= d.components.length) selectedStep = d.components.length - 1;
    paintSelection();
    renderInspector();
  }

  function renderInspector(){
    closeSelects();
    PICKERS = {};
    const top = inspInner.scrollTop, left = inspInner.scrollLeft;   // verticale o orizzontale, la lettura resta ferma
    const mEl = inspInner.querySelector('.insp-master'), dEl = inspInner.querySelector('.insp-detail');
    const gEl = inspInner.querySelector('.insp-general');
    const mTop = mEl ? mEl.scrollTop : 0, dTop = dEl ? dEl.scrollTop : 0, gTop = gEl ? gEl.scrollTop : 0;
    renderInspectorInner();
    arrangeMasterDetail();
    inspInner.scrollTop = top;
    inspInner.scrollLeft = left;
    const m2 = inspInner.querySelector('.insp-master'), d2 = inspInner.querySelector('.insp-detail');
    if (m2) m2.scrollTop = mTop;
    if (d2) d2.scrollTop = dTop;
    const g2 = inspInner.querySelector('.insp-general');
    if (g2) g2.scrollTop = gTop;
  }

  // in orizzontale: le voci restano a sinistra come elenco, i loro corpi passano a destra
  function arrangeMasterDetail(){
    const P = typeof PANELS !== 'undefined' ? PANELS.insp : null;
    const horiz = !!P && (P.side === 'top' || P.side === 'bottom');
    const conds = [...inspInner.querySelectorAll('.cond')];
    inspInner.classList.toggle('md', horiz && conds.length > 0);
    if (!horiz || !conds.length) return;
    // tre sezioni: impostazioni generali, struttura delle condizioni, dettaglio della condizione scelta
    const general = document.createElement('div'); general.className = 'insp-general';
    const master = document.createElement('div'); master.className = 'insp-master';
    const detail = document.createElement('div'); detail.className = 'insp-detail';
    const logical = !!inspInner.querySelector('.grp-btn') || !!inspInner.querySelector('[data-add],[data-addkey]');
    general.innerHTML = '<div class="md-col">Impostazioni</div>';
    master.innerHTML = '<div class="md-col">' + (logical ? 'Condizioni e gruppi' : 'Elenco') + '</div>';
    while (inspInner.firstChild){
      const ch = inspInner.firstChild;
      (ch.classList && ch.classList.contains('cond-area') ? master : general).appendChild(ch);
    }
    inspInner.appendChild(general); inspInner.appendChild(master); inspInner.appendChild(detail);
    detail.innerHTML = '<div class="md-title"></div><div class="md-sum"></div>';
    let active = conds.find(c => c.classList.contains('open')) || conds[0];
    conds.forEach((c, idx) => {
      c.dataset.idx = idx;
      const body = c.querySelector(':scope > .cond-body');
      if (body){ body.dataset.forIdx = idx; detail.appendChild(body); }
    });
    activateMd(active);
  }
  function condOf(el){
    const direct = el.closest('.cond');
    if (direct) return direct;
    const body = el.closest('.cond-body');
    return body ? inspInner.querySelector('.insp-master .cond[data-idx="' + body.dataset.forIdx + '"]') : null;
  }
  function activateMd(cond){
    if (!cond) return;
    const idx = cond.dataset.idx;
    inspInner.querySelectorAll('.insp-master .cond').forEach(c => c.classList.toggle('open', c === cond));
    inspInner.querySelectorAll('.insp-detail .cond-body').forEach(b => b.classList.toggle('shown', b.dataset.forIdx === idx));
    // lo stato delle voci aperte sopravvive ai ridisegni
    const tg = cond.querySelector('.cond-toggle');
    if (tg){
      if (tg.dataset.open !== undefined) openCond = parseInt(tg.dataset.open, 10);
      else if (tg.dataset.openkey !== undefined) openKey = parseInt(tg.dataset.openkey, 10);
      else if (tg.dataset.mlopen !== undefined) openRows[tg.dataset.mlopen] = parseInt(tg.dataset.mi, 10);
    }
    updateMdTitle(cond);
  }
  function updateMdTitle(cond){
    const t = inspInner.querySelector('.md-title'), sm = inspInner.querySelector('.md-sum');
    if (!t || !cond) return;
    const n = cond.querySelector('.cond-n'), x = cond.querySelector('.cond-txt');
    t.textContent = n ? n.textContent : '';
    sm.textContent = x ? x.textContent : '';
    sm.classList.toggle('empty', !!(x && x.classList.contains('empty')));
  }
  inspInner.addEventListener('click', (e) => {
    const b = e.target.closest('[data-cgrp],[data-csplit],[data-cungroup],[data-caddin],[data-kgrp],[data-ksplit],[data-kungroup],[data-kaddin]');
    if (!b) return;
    e.stopPropagation();
    const d = b.dataset;
    const isK = ('kgrp' in d) || ('ksplit' in d) || ('kungroup' in d) || ('kaddin' in d);
    const list = isK ? (activeJoinPar && activeJoinPar.keys) : (activeFilterPar && activeFilterPar.conditions);
    if (!list) return;
    const pick = (a, c) => (a in d) ? d[a] : ((c in d) ? d[c] : undefined);
    const grp = pick('cgrp', 'kgrp'), spl = pick('csplit', 'ksplit'), ung = pick('cungroup', 'kungroup'), addin = pick('caddin', 'kaddin');
    if (grp !== undefined) groupPair(list, parseInt(grp, 10));
    else if (spl !== undefined) splitAt(list, parseInt(spl, 10));
    else if (ung !== undefined) list.forEach(x => { if (x.g === ung) delete x.g; });
    else if (addin !== undefined){
      let last = -1;
      list.forEach((x, i) => { if (x.g === addin) last = i; });
      const item = isK ? { left:'', op:'=', right:'', conn:'AND', g: addin }
                       : Object.assign(newCondition(), { conn:'AND', g: addin });
      list.splice(last + 1, 0, item);
      if (isK) openKey = last + 1; else openCond = last + 1;
    }
    normalizeGroups(list);
    renderInspector();
  });

  // nel master-dettaglio un click su una voce la mostra a destra: non la comprime
  inspInner.addEventListener('click', (e) => {
    if (!inspInner.classList.contains('md')) return;
    const t = e.target.closest('.cond-toggle');
    if (!t) return;
    e.stopPropagation(); e.preventDefault();
    activateMd(t.closest('.cond'));
  }, true);

  function renderInspectorInner(){
    if (selectedSet.size > 1){
      const ids = [...selectedSet].filter(id => cards[id]);
      const ops = ids.filter(id => cards[id].kind === 'op').length;
      inspInner.innerHTML =
        '<div class="insp-head"><div><div class="insp-kind">Selezione</div>'+
        '<div class="insp-name" style="border-bottom:0">'+ids.length+' nodi</div></div>'+
        '<div class="insp-actions"><button class="close-btn" id="inspClose" aria-label="Chiudi">'+
        '<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>'+
        '</button></div></div>'+
        '<div class="insp-note">'+ops+' lavorazioni e '+(ids.length-ops)+' dataset. Trascinane uno per spostarli insieme.</div>'+
        '<div class="insp-lock"><div>Canc elimina · Cmd/Ctrl+D duplica · frecce spostano · Esc annulla la selezione</div></div>';
      const c = document.getElementById('inspClose');
      if (c) c.addEventListener('click', deselect);
      return;
    }
    const d = selectedUid ? cards[selectedUid] : null;
    if (d && inspCollapsed){
      inspInner.innerHTML =
        '<button class="insp-rail" id="inspExpand" aria-label="Espandi il pannello">'+
          '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><polyline points="11 17 6 12 11 7"/><polyline points="18 17 13 12 18 7"/></svg>'+
        '</button>'+
        '<div class="insp-rail-label">'+esc(d.name)+'</div>';
      const be = document.getElementById('inspExpand');
      if (be) be.addEventListener('click', () => setCollapsed(false));
      return;
    }
    if (!d){
      inspInner.innerHTML = '<div class="insp-empty">Seleziona un nodo sul canvas per configurarne i parametri.</div>';
      return;
    }
    ensureParams(d);
    const kind = d.kind === 'dataset' ? (d.isOutput ? 'Risultato' : 'Sorgente') : (d.components.length > 1 ? 'Box combinato' : 'Lavorazione');
    let html = '<datalist id="colList">'+SCHEMA.map(c => '<option value="'+c.name+'">').join('')+'</datalist>';
    html += '<div class="insp-head"><div>'+
      '<div class="insp-kind">'+kind+'</div>'+
      '<div class="insp-name" id="inspName" contenteditable="true">'+d.name+'</div>'+
      '</div><div class="insp-actions">'+
        '<button class="close-btn" id="inspCollapse" aria-label="Comprimi il pannello">'+
          '<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><polyline points="13 17 18 12 13 7"/><polyline points="6 17 11 12 6 7"/></svg>'+
        '</button>'+
        '<button class="close-btn" id="inspClose" aria-label="Chiudi">'+
          '<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>'+
        '</button>'+
      '</div></div>';

    if (d.isOutput){
      const prod = linksArr.find(l => l.to === selectedUid);
      const pname = prod && cards[prod.from] ? cards[prod.from].name : 'una lavorazione';
      html += '<div class="insp-note">Risultato generato da <b>'+pname+'</b>. I suoi parametri si configurano nei passaggi che lo producono.</div>';
      if (d.capacity > 1 && d.filled < d.capacity)
        html += '<div class="insp-note">Incompleto: al join manca una tabella.</div>';
      inspInner.innerHTML = html;
      bindInspector();
      return;
    }

    // una lavorazione senza tabella in ingresso non ha nulla da configurare
    if (d.kind === 'op' && inputsOf(selectedUid).length === 0){
      html += '<div class="insp-lock">'+
        '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="11" width="16" height="9" rx="2"/><path d="M8 11V7a4 4 0 0 1 8 0v4"/></svg>'+
        '<div>Collega una tabella a questo nodo per configurarne i parametri: colonne, chiavi e valori dipendono dai dati in ingresso.</div>'+
      '</div>';
      const cap = boxCapacity(d);
      if (cap > 1) html += '<div class="insp-note">Serve un Join completo: <b>'+cap+' tabelle</b> in ingresso.</div>';
      inspInner.innerHTML = html;
      bindInspector();
      return;
    }

    if (d.components.length > 1){
      html += '<div class="seq'+(seqOpen ? ' open' : '')+'">'+
        '<button type="button" class="seq-head">'+
          '<span class="seq-chev"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.8" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg></span>'+
          '<span class="seq-label">Sequenza di esecuzione</span>'+
          '<span class="seq-count">'+d.components.length+'</span>'+
        '</button>'+
        '<div class="step-list">' +
        d.components.map((c, i) =>
          '<div class="insp-step'+(i === selectedStep ? ' on' : '')+
            (pendingStepAnim && i >= pendingStepAnim.start && i <= pendingStepAnim.end ? ' in' : '')+
            '" data-step="'+i+'" style="'+
            (pendingStepAnim && i >= pendingStepAnim.start && i <= pendingStepAnim.end
              ? 'animation-delay:'+((i - pendingStepAnim.start) * 80)+'ms' : '')+'">'+
            '<span class="grip">'+GRIP+'</span>'+
            '<span class="st-n">'+(i+1)+'</span>'+
            '<span class="st-ico">'+svgTag(c)+'</span>'+
            '<span class="st-name">'+META[c].label+'</span>'+
            '<span class="st-sp"></span>'+
            '<button type="button" class="st-del" data-del="'+i+'" aria-label="Elimina passaggio">'+
              '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round"><line x1="17" y1="7" x2="7" y2="17"/><line x1="7" y1="7" x2="17" y2="17"/></svg>'+
            '</button>'+
          '</div>').join('') + '</div></div>';
    }
    const stepType = d.components[selectedStep] || d.components[0];
    const vals = d.params[selectedStep] || {};

    // i join si applicano nell'ordine in cui compaiono: ognuno consuma una tabella in piu
    const joinPos = [];
    d.components.forEach((c, i) => { if (MERGE_OPS.includes(c)) joinPos.push(i); });
    if (joinPos.length){
      const names = inputsOf(selectedUid).map(l => (cards[l.from] ? cards[l.from].name : 'tabella'));
      const tableField = (key, label) => {
        if (!names.length) return '<div class="insp-note">Collega le tabelle per scegliere su quale agisce questo passaggio.</div>';
        // la tabella destra parte dalla seconda collegata, non dalla stessa della sinistra
        if (!vals[key] || !names.includes(vals[key])) vals[key] = (key === 'rightTable' && names[1]) ? names[1] : names[0];
        return '<div class="field"><label>'+label+'</label>'+
          selectHtml(names, vals[key], 'data-key="'+key+'" data-step="'+selectedStep+'"')+'</div>';
      };
      const joinsBefore = joinPos.filter(pi => pi < selectedStep).length;

      if (MERGE_OPS.includes(stepType)){
        const j = joinPos.indexOf(selectedStep);       // quale join e, nell'ordine
        html += (j === 0)
          ? tableField('leftTable', 'Tabella sinistra')
          : '<div class="insp-note">Tabella sinistra: <b>risultato del join '+j+'</b>, già unito nei passaggi precedenti.</div>';
        html += tableField('rightTable', 'Tabella destra');
      } else if (joinsBefore === 0){
        html += tableField('table', 'Tabella di riferimento');
      } else {
        html += '<div class="insp-note">Opera sul risultato del join '+joinsBefore+': da qui la tabella è una sola.</div>';
      }
    }

    if (stepType === 'filter') html += renderFilter(vals);
    else if (stepType === 'join') html += (Array.isArray(PARAM_DEFS.join) ? PARAM_DEFS.join : [])
      .map(f => fieldHtml(f, vals[f.k], selectedStep)).join('') + renderJoinKeys(vals);
    else if (MULTI_DEFS[stepType]) html += renderMulti(stepType, vals);
    else html += (Array.isArray(PARAM_DEFS[stepType]) ? PARAM_DEFS[stepType] : [])
      .map(f => fieldHtml(f, vals[f.k], selectedStep)).join('');

    if (d.kind === 'op'){
      const cap = boxCapacity(d), got = inputsOf(selectedUid).length;
      html += '<div class="insp-note">Tabelle in ingresso: <b>'+got+' su '+cap+'</b>.</div>';
    }
    inspInner.innerHTML = html;
    pendingStepAnim = null;      // l'ingresso si gioca una volta sola
    bindInspector();
    if (stepType === 'filter') bindFilter(vals);
    if (stepType === 'join') bindJoin(vals);
    if (MULTI_DEFS[stepType]) bindMulti(stepType, vals);
  }

  // la sequenza si riordina anche da qui, trascinando le righe
  function bindStepList(){
    const seq = inspInner.querySelector('.seq');
    if (seq){
      // apre e chiude senza ricostruire: la vista resta ferma
      seq.querySelector('.seq-head').addEventListener('click', () => {
        seqOpen = !seqOpen;
        seq.classList.toggle('open', seqOpen);
      });
    }
    inspInner.querySelectorAll('.st-del').forEach(b =>
      b.addEventListener('click', (e) => {
        e.stopPropagation();
        deleteStep(selectedUid, parseInt(b.dataset.del, 10));
      }));

    const list = inspInner.querySelector('.step-list');
    if (!list) return;
    list.addEventListener('pointerdown', (e) => {
      if (e.target.closest('.st-del')) return;      // il pulsante non avvia il trascinamento
      const row = e.target.closest('.insp-step');
      if (!row) return;
      e.preventDefault();
      const rows = Array.from(list.querySelectorAll('.insp-step'));
      const rects = rows.map(r => r.getBoundingClientRect());
      const startIndex = rows.indexOf(row);
      const stepH = rects.length > 1 ? (rects[1].top - rects[0].top) : rects[0].height + 4;
      const startY = e.clientY;
      let dragging = false, newIndex = startIndex;

      function shifts(){
        rows.forEach((r, i) => {
          if (r === row) return;
          let sh = 0;
          if (startIndex < newIndex && i > startIndex && i <= newIndex) sh = -stepH;
          else if (startIndex > newIndex && i >= newIndex && i < startIndex) sh = stepH;
```


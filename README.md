# <!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Restaurant Inventory — Single-file App</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
  <!-- SheetJS for XLSX export -->
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
  <style>
    :root{
      --accent:#2b9af3; --ok:#2ecc71; --danger:#e74c3c; --muted:#6b7280;
      --card-bg:#fff; --bg:#f4f6f8; font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    *{box-sizing:border-box}
    body{margin:0;background:var(--bg);color:#111}
    header{background:var(--accent);color:#fff;padding:18px 16px;text-align:center}
    header h1{margin:0;font-size:18px}
    .container{max-width:1100px;margin:18px auto;padding:0 16px}
    .grid{display:grid;gap:16px}
    .cols-3{grid-template-columns: 1fr 1fr 1fr}
    .cols-2{grid-template-columns: 1fr 1fr}
    .card{background:var(--card-bg);padding:12px;border-radius:10px;box-shadow:0 4px 12px rgba(0,0,0,0.06)}
    .form-row{display:flex;gap:8px;margin-top:8px}
    input, select, textarea, button{font-size:14px}
    input, select, textarea{padding:10px;border-radius:8px;border:1px solid #ddd;width:100%}
    button{padding:10px 12px;border-radius:8px;border:0;background:var(--accent);color:#fff;cursor:pointer}
    button.secondary{background:#556;opacity:.95}
    table{width:100%;border-collapse:collapse;margin-top:8px}
    th,td{padding:8px;border-bottom:1px solid #eee;text-align:left;font-size:13px}
    .status-low{color:var(--danger);font-weight:700}
    .status-surplus{color:var(--ok);font-weight:700}
    .muted{color:var(--muted);font-size:13px}
    .topbar{display:flex;justify-content:space-between;align-items:center;gap:12px}
    .small{font-size:12px;padding:6px 8px;border-radius:8px}
    .flex{display:flex;gap:8px;align-items:center}
    .chart-wrap{height:260px;padding:8px}
    footer{padding:16px;text-align:center;color:var(--muted);font-size:13px}
    @media (max-width:900px){
      .cols-3,.cols-2{grid-template-columns:1fr}
      .form-row{flex-direction:column}
    }
    /* tiny helpers */
    .pill{display:inline-block;padding:6px 8px;border-radius:999px;background:#f1f5f9;font-size:12px}
  </style>
</head>
<body>
  <header>
    <h1>Restaurant Inventory — single-file app (localStorage)</h1>
  </header>

  <div class="container">
    <div class="card topbar">
      <div>
        <strong id="app-role-badge" class="pill">Not signed in</strong>
        <span class="muted"> — Offline demo</span>
      </div>
      <div class="flex" id="auth-area">
        <!-- auth UI injected here -->
      </div>
    </div>

    <div id="main" class="grid cols-3" style="margin-top:16px">
      <!-- Left column: Daily update + quick add -->
      <div class="card">
        <h3>Daily Update (Storekeeper)</h3>
        <form id="dailyForm">
          <label class="muted">Choose existing item</label>
          <select id="selectItem">
            <option value="">-- select existing item --</option>
          </select>

          <div class="muted" style="margin-top:8px">Or add new item name</div>
          <input id="newItemName" placeholder="e.g., Tomatoes" />

          <div class="form-row">
            <input id="purchasedQty" type="number" min="0" placeholder="Purchased qty" />
            <input id="usedQty" type="number" min="0" placeholder="Used qty" />
          </div>
          <div class="form-row">
            <button type="submit">Save Daily Update</button>
            <button type="button" id="clearDaily" class="secondary">Clear</button>
          </div>
          <div class="muted" style="margin-top:8px">Tip: include unit in item name if desired (e.g., "Tomatoes (kg)").</div>
        </form>

        <hr style="margin:12px 0" />

        <h4>Quick Add / Manage Item</h4>
        <form id="itemForm">
          <input id="itemName" placeholder="Item name" />
          <div class="form-row">
            <input id="itemStock" type="number" placeholder="Starting stock (number)" />
            <input id="itemThreshold" type="number" placeholder="Low threshold" />
          </div>
          <div class="form-row">
            <button id="addItemBtn" type="button">Add / Save Item</button>
            <button id="deleteItemBtn" type="button" class="secondary">Delete Selected</button>
          </div>
        </form>
      </div>

      <!-- Middle column: Inventory table -->
      <div class="card">
        <div style="display:flex;justify-content:space-between;align-items:center">
          <h3>Current Inventory</h3>
          <div>
            <button id="exportInventoryBtn">Export Inventory XLSX</button>
            <button id="exportHistoryBtn" class="secondary">Export History XLSX</button>
          </div>
        </div>

        <div style="margin-top:8px" class="muted">Shows current stock, low/surplus indicators and quick reorder hint.</div>
        <table id="inventoryTable">
          <thead><tr><th>Item</th><th>Stock</th><th>Threshold</th><th>Status</th><th>Actions</th></tr></thead>
          <tbody></tbody>
        </table>
      </div>

      <!-- Right column: Trends & History -->
      <div class="card">
        <h3>Trends & History</h3>
        <div style="display:flex;gap:8px;align-items:center;margin-bottom:8px">
          <select id="historyItemSelect"><option value="">— All items —</option></select>
          <input id="fromDate" type="date" />
          <input id="toDate" type="date" />
          <button id="filterHistoryBtn" class="secondary">Filter</button>
        </div>

        <div class="chart-wrap card">
          <canvas id="stockChart"></canvas>
        </div>

        <h4 style="margin-top:12px">Recent Updates</h4>
        <table id="historyTable">
          <thead><tr><th>Date</th><th>Item</th><th>Purchased</th><th>Used</th><th>Result Stock</th><th>User</th></tr></thead>
          <tbody></tbody>
        </table>
      </div>
    </div>

    <!-- bottom full-width controls -->
    <div style="margin-top:16px" class="grid cols-2">
      <div class="card">
        <h3>Users (multi-user friendly)</h3>
        <div id="usersArea"></div>
        <hr/>
        <form id="registerForm">
          <div style="display:flex;gap:8px">
            <input id="regName" placeholder="Name" />
            <input id="regEmail" placeholder="Email" />
          </div>
          <div style="display:flex;gap:8px;margin-top:8px">
            <input id="regPassword" placeholder="Password" type="password" />
            <select id="regRole">
              <option value="storekeeper">Storekeeper</option>
              <option value="manager">Manager</option>
            </select>
            <button type="submit">Register</button>
          </div>
          <div class="muted" style="margin-top:8px">Manager can export and manage items; storekeeper can submit daily updates.</div>
        </form>
      </div>

      <div class="card">
        <h3>App Controls & Backup</h3>
        <div style="display:flex;gap:8px">
          <button id="seedSample">Seed Sample Data</button>
          <button id="wipeData" class="secondary">Wipe All Data</button>
          <button id="backupBtn">Download JSON Backup</button>
          <input id="restoreFile" type="file" accept="application/json" style="display:none" />
          <button id="restoreBtn" class="secondary">Restore from JSON</button>
        </div>
        <div style="margin-top:12px" class="muted">
          This single-file app stores everything in your browser's localStorage. Use backup/restore to move data between devices.
        </div>
      </div>
    </div>

    <footer>
      Single-file offline inventory manager — works on desktop & mobile browsers. Export to XLSX supported.
    </footer>
  </div>

<script>
/* ======================================================
   Single-file Inventory app
   - storage: localStorage
   - password hashing: SHA-256 via SubtleCrypto
   - exports: SheetJS (XLSX)
   - charts: Chart.js
   ====================================================== */

/* ---------- Utilities ---------- */
const UID = (n=8)=>crypto.randomUUID ? crypto.randomUUID() : Math.random().toString(36).slice(2,2+n);
const nowISO = ()=> new Date().toISOString();
const fmtDate = iso => { const d=new Date(iso); return isNaN(d) ? iso : d.toLocaleString(); };
const ls = (k,v)=> v===undefined ? JSON.parse(localStorage.getItem(k) || 'null') : localStorage.setItem(k, JSON.stringify(v));

async function hashPw(password){
  const enc = new TextEncoder().encode(password);
  const digest = await crypto.subtle.digest('SHA-256', enc);
  const b = Array.from(new Uint8Array(digest)).map(x=>x.toString(16).padStart(2,'0')).join('');
  return b;
}

/* ---------- Data model in localStorage ---------- */
/* Keys:
   - users: array [{id,name,email,passwordHash,role}]
   - items: object map {itemId: {id,name,stock,low_threshold,created_at}}
   - updates: array [{id,item_id,date,purchased,used,resulting_stock,user_id,note}]
   - currentUserId: id or null
*/
function initData(){
  if (!ls('users')) ls('users', []);
  if (!ls('items')) ls('items', {});
  if (!ls('updates')) ls('updates', []);
  if (!ls('currentUserId')) ls('currentUserId', null);
}

/* ---------- Auth UI and functions ---------- */
const authArea = document.getElementById('auth-area');
const appRoleBadge = document.getElementById('app-role-badge');

async function renderAuthArea(){
  const uid = ls('currentUserId');
  const users = ls('users') || [];
  authArea.innerHTML = '';
  if (!uid){
    // show login controls
    const loginBtn = document.createElement('button');
    loginBtn.textContent = 'Login';
    const loginMenu = document.createElement('div');
    loginMenu.style.display='flex';
    loginMenu.style.gap='8px';
    const sel = document.createElement('select');
    sel.style.padding='8px';
    sel.innerHTML = '<option value="">-- choose user to login --</option>';
    users.forEach(u => sel.innerHTML += `<option value="${u.id}">${u.name} (${u.role})</option>`);
    const pw = document.createElement('input');
    pw.type='password'; pw.placeholder='password'; pw.style.padding='8px';
    const btn = document.createElement('button'); btn.textContent='Sign In';
    btn.onclick = async ()=>{
      if(!sel.value){ alert('Select a user to login'); return; }
      const u = users.find(x=>x.id===sel.value);
      const h = await hashPw(pw.value || '');
      if (h !== u.passwordHash){ alert('Invalid password'); return; }
      ls('currentUserId', u.id);
      await finishLogin();
    };
    loginMenu.appendChild(sel); loginMenu.appendChild(pw); loginMenu.appendChild(btn);
    authArea.appendChild(loginMenu);

    // quick "guest register" sample manager/storekeeper button
    const hint = document.createElement('div'); hint.className='muted'; hint.style.marginLeft='8px';
    hint.textContent='If no users, register below (manager/storekeeper)';
    authArea.appendChild(hint);

    appRoleBadge.textContent = 'Not signed in';
  } else {
    // show signed-in info + logout
    const usersMap = (ls('users')||[]).reduce((acc,u)=>{acc[u.id]=u; return acc},{});
    const user = usersMap[uid];
    const info = document.createElement('div'); info.style.display='flex'; info.style.gap='8px'; info.style.alignItems='center';
    const who = document.createElement('div'); who.textContent = `${user.name} (${user.role})`; who.style.fontWeight='700';
    const logoutBtn = document.createElement('button'); logoutBtn.textContent='Logout';
    logoutBtn.onclick = ()=>{ ls('currentUserId', null); renderAuthArea(); refreshAll(); };
    info.appendChild(who); info.appendChild(logoutBtn);
    authArea.appendChild(info);
    appRoleBadge.textContent = `${user.role.toUpperCase()}`;
  }
}

/* convenience to get current user object */
function currentUser(){
  const id = ls('currentUserId'); return (ls('users')||[]).find(u=>u.id===id) || null;
}
async function finishLogin(){
  await renderAuthArea();
  refreshAll();
}

/* ---------- UI wiring ---------- */
const selectItem = document.getElementById('selectItem');
const newItemName = document.getElementById('newItemName');
const purchasedQty = document.getElementById('purchasedQty');
const usedQty = document.getElementById('usedQty');
const dailyForm = document.getElementById('dailyForm');
const clearDaily = document.getElementById('clearDaily');

const itemForm = document.getElementById('itemForm');
const itemName = document.getElementById('itemName');
const itemStock = document.getElementById('itemStock');
const itemThreshold = document.getElementById('itemThreshold');
const addItemBtn = document.getElementById('addItemBtn');
const deleteItemBtn = document.getElementById('deleteItemBtn');

const inventoryTableBody = document.querySelector('#inventoryTable tbody');
const historyTableBody = document.querySelector('#historyTable tbody');
const historyItemSelect = document.getElementById('historyItemSelect');
const fromDate = document.getElementById('fromDate');
const toDate = document.getElementById('toDate');
const filterHistoryBtn = document.getElementById('filterHistoryBtn');

const usersArea = document.getElementById('usersArea');
const registerForm = document.getElementById('registerForm');
const regName = document.getElementById('regName');
const regEmail = document.getElementById('regEmail');
const regPassword = document.getElementById('regPassword');
const regRole = document.getElementById('regRole');

const seedSample = document.getElementById('seedSample');
const wipeData = document.getElementById('wipeData');
const backupBtn = document.getElementById('backupBtn');
const restoreBtn = document.getElementById('restoreBtn');
const restoreFile = document.getElementById('restoreFile');

const exportInventoryBtn = document.getElementById('exportInventoryBtn');
const exportHistoryBtn = document.getElementById('exportHistoryBtn');

let stockChart = null;

/* ---------- Core operations ---------- */
function getItemsMap(){ return ls('items') || {}; }
function setItemsMap(m){ ls('items', m); }
function getUpdates(){ return ls('updates') || []; }
function setUpdates(a){ ls('updates', a); }

function createOrUpdateItem({id,name,stock=0,low_threshold=10}){
  const items = getItemsMap();
  const now = nowISO();
  if(!id){
    id = UID(6);
    items[id] = {id,name,stock:Number(stock),low_threshold:Number(low_threshold),created_at:now};
  } else {
    if(!items[id]) items[id] = {id,name,stock:Number(stock),low_threshold:Number(low_threshold),created_at:now};
    else { items[id].name = name; items[id].stock = Number(stock); items[id].low_threshold = Number(low_threshold); }
  }
  setItemsMap(items);
  return items[id];
}

function deleteItem(itemId){
  const items = getItemsMap();
  if(items[itemId]) delete items[itemId];
  // optionally remove related updates
  const updates = getUpdates().filter(u=>u.item_id!==itemId);
  setItemsMap(items); setUpdates(updates);
}

function addDailyUpdate({item_id, purchased=0, used=0, date, user_id, note}){
  const items = getItemsMap();
  if(!items[item_id]){ console.warn('item missing'); return null; }
  const prevStock = Number(items[item_id].stock || 0);
  const newStock = Number(prevStock) + Number(purchased) - Number(used);
  items[item_id].stock = newStock;
  setItemsMap(items);

  const updates = getUpdates();
  const rec = { id: UID(6), item_id, date: date || nowISO(), purchased: Number(purchased), used: Number(used), resulting_stock: newStock, user_id: user_id || null, note: note || '' };
  updates.push(rec);
  setUpdates(updates);
  return rec;
}

/* ---------- Rendering ---------- */
function refreshItemSelects(){
  const items = Object.values(getItemsMap()).sort((a,b)=>a.name.localeCompare(b.name));
  selectItem.innerHTML = '<option value="">-- select existing item --</option>';
  historyItemSelect.innerHTML = '<option value="">— All items —</option>';
  document.querySelectorAll('#selectItem, #historyItemSelect').forEach(sel=>{
    items.forEach(it => {
      const opt = document.createElement('option'); opt.value = it.id; opt.textContent = `${it.name} (stock:${it.stock})`;
      sel.appendChild(opt);
    });
  });
}

function refreshInventoryTable(){
  const items = Object.values(getItemsMap()).sort((a,b)=>a.name.localeCompare(b.name));
  inventoryTableBody.innerHTML = '';
  items.forEach(it=>{
    const tr = document.createElement('tr');
    const status = it.stock <= it.low_threshold ? '<span class="status-low">Low</span>' : (it.stock > it.low_threshold * 5 ? '<span class="status-surplus">Surplus</span>' : 'OK');
    tr.innerHTML = `<td>${it.name}</td><td>${Number(it.stock).toFixed(2)}</td><td>${it.low_threshold}</td><td>${status}</td>
      <td>
        <button class="small" data-action="quick-use" data-id="${it.id}">Use</button>
        <button class="small" data-action="quick-add" data-id="${it.id}">Add</button>
        <button class="small" data-action="edit" data-id="${it.id}">Edit</button>
      </td>`;
    inventoryTableBody.appendChild(tr);
  });
}

function renderUsersArea(){
  const users = ls('users') || [];
  usersArea.innerHTML = '';
  if(users.length===0) usersArea.innerHTML = '<div class="muted">No users. Register below to create manager/storekeeper accounts.</div>';
  users.forEach(u=>{
    const div = document.createElement('div'); div.style.display='flex'; div.style.justifyContent='space-between'; div.style.alignItems='center'; div.style.marginTop='8px';
    div.innerHTML = `<div><strong>${u.name}</strong> <span class="muted">(${u.role})</span><div class="muted">${u.email}</div></div>
      <div style="display:flex;gap:8px"><button class="small" data-act="impersonate" data-id="${u.id}">Impersonate</button><button class="small" data-act="deleteUser" data-id="${u.id}">Delete</button></div>`;
    usersArea.appendChild(div);
  });
}

/* ---------- History & Chart ---------- */
function getFilteredHistory({item_id, from, to} = {}){
  let upd = getUpdates().slice().sort((a,b)=> new Date(a.date) - new Date(b.date));
  if(item_id) upd = upd.filter(u=>u.item_id === item_id);
  if(from) upd = upd.filter(u=> new Date(u.date) >= new Date(from+'T00:00:00'));
  if(to) upd = upd.filter(u=> new Date(u.date) <= new Date(to+'T23:59:59'));
  return upd;
}

function refreshHistoryTable({item_id, from, to} = {}){
  const items = getItemsMap();
  const users = (ls('users')||[]).reduce((acc,u)=>{acc[u.id]=u; return acc;},{});
  const hist = getFilteredHistory({item_id, from, to}).slice().reverse();
  historyTableBody.innerHTML = '';
  hist.forEach(h=>{
    const tr = document.createElement('tr');
    tr.innerHTML = `<td>${fmtDate(h.date)}</td><td>${items[h.item_id]?.name || '—'}</td><td>${h.purchased}</td><td>${h.used}</td><td>${h.resulting_stock}</td><td>${users[h.user_id]?.name || '-'}</td>`;
    historyTableBody.appendChild(tr);
  });
}

function drawStockChart({item_id, from, to} = {}){
  const updates = getFilteredHistory({item_id, from, to});
  // Build time series of resulting_stock; aggregate by date label
  const labels = updates.map(u=> new Date(u.date).toLocaleString());
  const stocks = updates.map(u=> u.resulting_stock);
  const purchases = updates.map(u=> u.purchased);
  const used = updates.map(u=> u.used);

  const ctx = document.getElementById('stockChart').getContext('2d');
  if(stockChart) stockChart.destroy();
  stockChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [
        { label: 'Stock', data: stocks, borderColor: '#2b9af3', backgroundColor:'rgba(43,154,243,0.08)', tension:0.25, yAxisID:'y' },
        { label: 'Purchased', data: purchases, borderColor: '#27ae60', backgroundColor:'rgba(39,174,96,0.08)', tension:0.25, yAxisID:'y1' },
        { label: 'Used', data: used, borderColor: '#e74c3c', backgroundColor:'rgba(231,76,60,0.08)', tension:0.25, yAxisID:'y1' }
      ]
    },
    options: {
      plugins:{legend:{position:'top'}},
      scales:{
        y: { position:'left', title:{display:true,text:'Stock'} },
        y1:{ position:'right', grid:{display:false}, title:{display:true,text:'Qty (purchased/used)'} }
      },
      maintainAspectRatio:false
    }
  });
}

/* ---------- Exports (XLSX) ---------- */
function exportInventoryXLSX(){
  const items = Object.values(getItemsMap()).map(i=>({
    'Item ID': i.id, 'Name': i.name, 'Stock': i.stock, 'Low threshold': i.low_threshold, 'Created at': i.created_at
  }));
  const wb = XLSX.utils.book_new();
  const ws = XLSX.utils.json_to_sheet(items);
  XLSX.utils.book_append_sheet(wb, ws, 'Inventory');
  XLSX.writeFile(wb, `inventory_${(new Date()).toISOString().slice(0,10)}.xlsx`);
}

function exportHistoryXLSX({item_id, from, to} = {}){
  const items = getItemsMap();
  const users = (ls('users')||[]).reduce((acc,u)=>{acc[u.id]=u; return acc;},{});
  const hist = getFilteredHistory({item_id, from, to}).map(h=>({
    'Update ID': h.id,
    'Date': h.date,
    'Item ID': h.item_id,
    'Item Name': items[h.item_id]?.name || '',
    'Purchased': h.purchased,
    'Used': h.used,
    'Resulting stock': h.resulting_stock,
    'User': users[h.user_id]?.name || '',
    'Note': h.note || ''
  }));
  const wb = XLSX.utils.book_new();
  const ws = XLSX.utils.json_to_sheet(hist);
  XLSX.utils.book_append_sheet(wb, ws, 'History');
  XLSX.writeFile(wb, `history_${(new Date()).toISOString().slice(0,10)}.xlsx`);
}

/* ---------- Event handlers ---------- */
dailyForm.addEventListener('submit', async (e)=>{
  e.preventDefault();
  const user = currentUser();
  if(!user){ alert('Please login as a storekeeper to submit daily updates'); return; }
  const chosen = selectItem.value;
  const name = newItemName.value.trim();
  let itemId = chosen || null;
  if(!itemId && !name){ alert('Select an item or enter a new item name'); return; }
  if(!itemId){
    // create item with default threshold
    const it = createOrUpdateItem({name, stock: 0, low_threshold: 10});
    itemId = it.id;
  }
  const purchased = Number(purchasedQty.value) || 0;
  const used = Number(usedQty.value) || 0;
  const rec = addDailyUpdate({item_id: itemId, purchased, used, date: nowISO(), user_id: user.id});
  // UI feedback
  newItemName.value=''; purchasedQty.value=''; usedQty.value='';
  refreshAll();
  alert('Saved daily update: ' + fmtDate(rec.date));
});

clearDaily.addEventListener('click', ()=>{ newItemName.value=''; purchasedQty.value=''; usedQty.value=''; selectItem.value=''; });

addItemBtn.addEventListener('click', ()=>{
  const name = itemName.value.trim();
  if(!name) return alert('Enter item name');
  // if a select item is chosen in the main select, we consider editing that; otherwise new item
  const existingId = selectItem.value;
  if(existingId && getItemsMap()[existingId]?.name === name){
    // update stock / threshold
    createOrUpdateItem({id: existingId, name, stock: Number(itemStock.value) || 0, low_threshold: Number(itemThreshold.value) || 10});
    alert('Item updated');
  } else {
    createOrUpdateItem({name, stock: Number(itemStock.value) || 0, low_threshold: Number(itemThreshold.value) || 10});
    alert('Item added');
  }
  itemName.value=''; itemStock.value=''; itemThreshold.value='';
  refreshAll();
});

deleteItemBtn.addEventListener('click', ()=>{
  const selected = selectItem.value;
  if(!selected) return alert('Select item in top selector to delete');
  if(!confirm('Delete this item and its history?')) return;
  deleteItem(selected);
  refreshAll();
});

inventoryTableBody.addEventListener('click', (ev)=>{
  const btn = ev.target.closest('button');
  if(!btn) return;
  const action = btn.dataset.action;
  const id = btn.dataset.id;
  if(action === 'edit'){
    const it = getItemsMap()[id];
    itemName.value = it.name; itemStock.value = it.stock; itemThreshold.value = it.low_threshold;
    // also select in selectItem
    selectItem.value = id;
  } else if(action === 'quick-use'){
    const q = prompt('Enter qty used (positive number)', '1'); if(!q) return;
    const user = currentUser(); if(!user) return alert('Login first');
    addDailyUpdate({item_id: id, purchased:0, used:Number(q), date: nowISO(), user_id: user.id});
    refreshAll();
  } else if(action === 'quick-add'){
    const q = prompt('Enter qty added (positive number)', '1'); if(!q) return;
    const user = currentUser(); if(!user) return alert('Login first');
    addDailyUpdate({item_id: id, purchased:Number(q), used:0, date: nowISO(), user_id: user.id});
    refreshAll();
  }
});

filterHistoryBtn.addEventListener('click', ()=>{
  refreshHistoryTable({item_id: historyItemSelect.value || null, from: fromDate.value || null, to: toDate.value || null});
  drawStockChart({item_id: historyItemSelect.value || null, from: fromDate.value || null, to: toDate.value || null});
});

registerForm.addEventListener('submit', async (e)=>{
  e.preventDefault();
  const name = regName.value.trim(); const email = regEmail.value.trim().toLowerCase();
  const password = regPassword.value; const role = regRole.value;
  if(!name || !email || !password) return alert('Provide name, email and password');
  const users = ls('users') || [];
  if(users.some(u=>u.email === email)) return alert('Email already registered');

  const passwordHash = await hashPw(password);
  const id = UID(6);
  users.push({id,name,email,passwordHash,role});
  ls('users', users);
  regName.value=''; regEmail.value=''; regPassword.value='';
  renderUsersArea(); renderAuthArea();
  alert('User registered. You may login from top.');
});

usersArea.addEventListener('click', (ev)=>{
  const btn = ev.target.closest('button');
  if(!btn) return;
  const act = btn.dataset.act, id = btn.dataset.id;
  if(act === 'impersonate'){
    ls('currentUserId', id);
    renderAuthArea();
    refreshAll();
  } else if(act === 'deleteUser'){
    if(!confirm('Delete user?')) return;
    const users = (ls('users')||[]).filter(u=>u.id!==id);
    ls('users', users);
    // if deleted current user, logout
    if(ls('currentUserId') === id) ls('currentUserId', null);
    renderUsersArea(); renderAuthArea(); refreshAll();
  }
});

seedSample.addEventListener('click', ()=>{
  if(!confirm('Seed sample data? This will not overwrite existing data but will add sample users/items/updates.')) return;
  // sample users
  (async ()=>{
    const users = ls('users') || [];
    if(!users.some(u=>u.email==='manager@demo')) users.push({id:UID(6), name:'Manager Demo', email:'manager@demo', passwordHash: await hashPw('manager'), role:'manager'});
    if(!users.some(u=>u.email==='store@demo')) users.push({id:UID(6), name:'Store Demo', email:'store@demo', passwordHash: await hashPw('store'), role:'storekeeper'});
    ls('users', users);

    // sample items
    const items = getItemsMap();
    const add = (n,stock,thr)=> createOrUpdateItem({name:n,stock,low_threshold:thr});
    add('Tomatoes (kg)', 50, 10);
    add('Onions (kg)', 30, 8);
    add('Potatoes (kg)', 120, 20);
    add('Olive Oil (ltr)', 12, 3);
    add('Rice (kg)', 200, 30);

    // sample updates
    const u = ls('users')[0];
    addDailyUpdate({item_id: Object.values(getItemsMap())[0].id, purchased:20, used:5, date: new Date(Date.now()-3*24*3600*1000).toISOString(), user_id: u.id});
    addDailyUpdate({item_id: Object.values(getItemsMap())[1].id, purchased:10, used:2, date: new Date(Date.now()-2*24*3600*1000).toISOString(), user_id: u.id});
    addDailyUpdate({item_id: Object.values(getItemsMap())[2].id, purchased:0, used:15, date: new Date(Date.now()-1*24*3600*1000).toISOString(), user_id: u.id});
    refreshAll();
    alert('Sample data seeded. Login as manager@demo / manager or store@demo / store');
  })();
});

wipeData.addEventListener('click', ()=>{
  if(!confirm('Wipe ALL data from localStorage? This cannot be undone.')) return;
  localStorage.clear(); initData(); renderAuthArea(); refreshAll();
  alert('Data wiped. Page refreshed.');
});

backupBtn.addEventListener('click', ()=>{
  const payload = { users: ls('users')||[], items: getItemsMap(), updates: getUpdates() };
  const blob = new Blob([JSON.stringify(payload, null, 2)], {type:'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = `inventory-backup-${(new Date()).toISOString().slice(0,10)}.json`;
  document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url);
});

restoreBtn.addEventListener('click', ()=> restoreFile.click());
restoreFile.addEventListener('change', (ev)=>{
  const f = ev.target.files[0];
  if(!f) return;
  const reader = new FileReader();
  reader.onload = (e)=>{
    try{
      const obj = JSON.parse(e.target.result);
      if(confirm('Restore will overwrite current data. Continue?')){
        ls('users', obj.users || []); ls('items', obj.items || {}); ls('updates', obj.updates || []); ls('currentUserId', null);
        renderUsersArea(); renderAuthArea(); refreshAll();
        alert('Restore complete');
      }
    }catch(err){ alert('Invalid file'); }
  };
  reader.readAsText(f);
});

/* export buttons */
exportInventoryBtn.addEventListener('click', ()=> exportInventoryXLSX());
exportHistoryBtn.addEventListener('click', ()=> exportHistoryXLSX({}));

/* ---------- Initialization & refresh ---------- */
function refreshAll(){
  refreshItemSelects();
  refreshInventoryTable();
  renderUsersArea();
  refreshHistoryTable({});
  drawStockChart({});
  // role-based UI: hide daily form if not storekeeper
  const user = currentUser();
  const dailyCard = document.querySelector('#dailyForm').closest('.card');
  if(user && (user.role === 'storekeeper' || user.role === 'manager')){
    // both can access daily form; manager can too
    dailyCard.style.display = 'block';
  } else {
    dailyCard.style.display = 'block'; // we still allow, but block submission enforces login
  }
}

/* ---------- boot ---------- */
(async function boot(){
  initData();
  await renderAuthArea();
  renderUsersArea();
  refreshAll();
})();
</script>
</body>
</html>

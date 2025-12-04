<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>GANESH MARKETING — Place Order</title>
<style>
  :root{font-family:system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial;--accent:#0b5ed7}
  body{margin:0;background:#f3f6fb;color:#111;padding:18px}
  header{background:#fff;padding:22px;border-radius:12px;box-shadow:0 10px 30px rgba(10,20,60,0.06);max-width:980px;margin:0 auto 18px}
  h1{margin:0;font-size:2rem;letter-spacing:0.6px}
  .meta{margin-top:8px;color:#444}
  .container{max-width:980px;margin:0 auto;background:#fff;padding:18px;border-radius:12px;box-shadow:0 6px 18px rgba(10,20,40,0.05)}
  label{display:block;margin:8px 0 6px;font-weight:600}
  input[type="text"], input[type="number"], textarea{width:100%;padding:9px;border:1px solid #e0e4ea;border-radius:8px;font-size:0.95rem}
  table{width:100%;border-collapse:collapse;margin-top:12px}
  th, td{padding:8px;border-bottom:1px solid #eef2f6;text-align:left}
  th{background:#fbfdff;font-weight:700}
  .small{width:120px}
  .btn{background:var(--accent);color:#fff;padding:10px 14px;border-radius:10px;border:0;font-weight:700;cursor:pointer}
  .btn.ghost{background:#eef4ff;color:var(--accent)}
  .row{display:flex;gap:12px}
  .right{display:flex;gap:8px;align-items:center}
  .totalbox{margin-top:12px;display:flex;justify-content:flex-end;align-items:center;gap:16px}
  .bigtotal{font-size:1.25rem;font-weight:800}
  .note{font-size:0.9rem;color:#555;margin-top:8px}
  .success, .error{padding:12px;border-radius:8px;margin-top:12px}
  .success{background:#e8fff0;border:1px solid #c6efdb;color:#0a6b3f}
  .error{background:#fff0f0;border:1px solid #f1c6c6;color:#8a2b2b}
  .actions{display:flex;gap:8px;flex-wrap:wrap}
  .linklike{background:none;border:0;color:var(--accent);text-decoration:underline;cursor:pointer;padding:6px}
  .muted{color:#666;font-size:0.9rem}
</style>
</head>
<body>

<header>
  <h1>GANESH MARKETING</h1>
  <div class="meta">
    <strong>Contact:</strong> 7777777777 &nbsp; | &nbsp;
    <strong>Email:</strong> krutinkoradiya24@gamil.com
    <div class="note"><strong>Address:</strong> E-5 College Shopping Centar, Nadiad Bhuj, Jaipur - 355362</div>
  </div>
</header>

<div class="container">
  <h2>Place an order</h2>
  <p class="note">Add oil or grease items below. Enter unit price and quantity — totals calculate automatically.</p>

  <form id="orderForm" method="post" action="submit_order.php" novalidate>
    <label for="customerName">Customer name *</label>
    <input id="customerName" name="customerName" type="text" required placeholder="Customer full name" />

    <label for="customerPhone">Customer phone *</label>
    <input id="customerPhone" name="customerPhone" type="text" required placeholder="7777777777" />

    <label for="notes">Notes (optional)</label>
    <textarea id="notes" name="notes" placeholder="Delivery instructions, preferred time..."></textarea>

    <h3 style="margin-top:14px">Items</h3>
    <div style="overflow:auto">
      <table id="itemsTable" aria-live="polite">
        <thead>
          <tr>
            <th style="width:42%">Item name (oil / grease)</th>
            <th class="small">Quantity</th>
            <th class="small">Unit price (₹)</th>
            <th class="small">Line total (₹)</th>
            <th style="width:70px"></th>
          </tr>
        </thead>
        <tbody id="itemsBody"></tbody>
      </table>
    </div>

    <div style="margin-top:10px" class="actions">
      <button type="button" id="addRow" class="btn ghost">+ Add item</button>
      <button type="button" id="clearRows" class="linklike">Clear items</button>
    </div>

    <div class="totalbox">
      <div class="muted">Grand total:</div>
      <div class="bigtotal" id="grandTotal">₹ 0.00</div>
    </div>

    <div style="margin-top:14px;display:flex;gap:10px;align-items:center">
      <button type="submit" class="btn">Send Order</button>
      <div class="muted">Orders will be emailed to <strong>krutinkoradiya24@gamil.com</strong></div>
    </div>

    <input type="hidden" name="shopName" value="GANESH MARKETING" />
    <input type="hidden" name="shopContact" value="7777777777" />
    <input type="hidden" name="shopAddress" value="E-5 College Shopping Centar; Nadiad Bhuj; Jaipur; 355362" />
  </form>

  <div id="messageArea"></div>
</div>

<script>
(function(){
  const itemsBody = document.getElementById('itemsBody');
  const grandTotalEl = document.getElementById('grandTotal');
  const form = document.getElementById('orderForm');
  const messageArea = document.getElementById('messageArea');

  function currency(v){ return '₹ ' + Number(v).toFixed(2); }

  function createRow(name='', qty=1, price=0){
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td><input name="itemName[]" type="text" required placeholder="e.g. Engine Oil SAE 20W-50" value="${escapeHtml(name)}" /></td>
      <td><input name="itemQty[]" type="number" min="1" step="1" required value="${qty}" class="small" /></td>
      <td><input name="itemPrice[]" type="number" min="0" step="0.01" required value="${price}" class="small" /></td>
      <td><div class="lineTotal">₹ 0.00</div></td>
      <td><button type="button" class="removeBtn linklike">Remove</button></td>
    `;
    itemsBody.appendChild(tr);

    const qtyInput = tr.querySelector('input[name="itemQty[]"]');
    const priceInput = tr.querySelector('input[name="itemPrice[]"]');
    const removeBtn = tr.querySelector('.removeBtn');

    function recalcLine(){
      const q = parseFloat(qtyInput.value) || 0;
      const p = parseFloat(priceInput.value) || 0;
      const total = q * p;
      tr.querySelector('.lineTotal').textContent = currency(total);
      updateGrandTotal();
    }
    qtyInput.addEventListener('input', recalcLine);
    priceInput.addEventListener('input', recalcLine);
    removeBtn.addEventListener('click', () => {
      tr.remove();
      updateGrandTotal();
    });

    recalcLine();
  }

  function updateGrandTotal(){
    let sum = 0;
    document.querySelectorAll('#itemsBody tr').forEach(tr => {
      const q = parseFloat(tr.querySelector('input[name="itemQty[]"]').value) || 0;
      const p = parseFloat(tr.querySelector('input[name="itemPrice[]"]').value) || 0;
      sum += q * p;
    });
    grandTotalEl.textContent = currency(sum);
  }

  function escapeHtml(s){ return String(s).replaceAll('"','&quot;').replaceAll('<','&lt;').replaceAll('>','&gt;'); }

  createRow('',1,0);

  document.getElementById('addRow').addEventListener('click', () => createRow('',1,0));
  document.getElementById('clearRows').addEventListener('click', () => {
    itemsBody.innerHTML = '';
    createRow('',1,0);
    updateGrandTotal();
  });

  form.addEventListener('submit', function(e){
    messageArea.innerHTML = '';
    const name = form.customerName.value.trim();
    const phone = form.customerPhone.value.trim();
    if (!name){ e.preventDefault(); showError('Please enter customer name.'); return; }
    if (!phone){ e.preventDefault(); showError('Please enter customer phone.'); return; }
    const rows = document.querySelectorAll('#itemsBody tr');
    if (rows.length === 0){ e.preventDefault(); showError('Add at least one item.'); return; }

    for (const tr of rows){
      const iname = tr.querySelector('input[name="itemName[]"]').value.trim();
      const qty = Number(tr.querySelector('input[name="itemQty[]"]').value);
      const price = Number(tr.querySelector('input[name="itemPrice[]"]').value);
      if (!iname){ e.preventDefault(); showError('Item name cannot be empty.'); return; }
      if (!Number.isFinite(qty) || qty <= 0){ e.preventDefault(); showError('Quantity must be at least 1.'); return; }
      if (!Number.isFinite(price) || price < 0){ e.preventDefault(); showError('Price must be 0 or more.'); return; }
    }
    // allow form submit — PHP will handle actual sending
  });

  function showError(msg){
    messageArea.innerHTML = '<div class="error">' + msg + '</div>';
  }
})();
</script>

</body>
</html>

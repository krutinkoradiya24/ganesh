<?php
// index.php - single file: form + PHP handling (saves CSV + sends email via PHPMailer or mail())

// ------------------- CONFIG --------------------
$to_email   = 'krutinkoradiya24@gamil.com'; // change to correct address if needed
$csv_file   = __DIR__ . '/orders.csv';
$shopName   = 'GANESH MARKETING';
$shopPhone  = '7777777777';
$shopAddress= 'E-5 College Shopping Centar; Nadiad Bhuj; Jaipur; 355362';

// ==== SMTP CONFIG: fill these to send via SMTP (recommended) ====
$smtpHost = 'smtp.gmail.com';
$smtpPort = 587;
$smtpSecure = 'tls';           // 'tls' or 'ssl' or ''
$smtpUser = 'your-email@example.com'; // SMTP account (your email)
$smtpPass = 'your-smtp-password';     // SMTP password or app password

// ---------------- helper functions ----------------
function h($s){ return htmlspecialchars($s, ENT_QUOTES|ENT_SUBSTITUTE, 'UTF-8'); }
function respond_html($html){ echo $html; exit; }
function save_csv($csv_file, $row){
    $fp = @fopen($csv_file, 'a');
    if ($fp){
        fputcsv($fp, $row);
        fclose($fp);
        return true;
    }
    return false;
}

// ---------- handle POST (form submit) ----------
$sentResult = null; // will hold status message HTML

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // sanitize inputs
    $customerName  = isset($_POST['customerName']) ? trim(strip_tags($_POST['customerName'])) : '';
    $customerPhone = isset($_POST['customerPhone']) ? trim(strip_tags($_POST['customerPhone'])) : '';
    $notesRaw      = isset($_POST['notes']) ? trim($_POST['notes']) : '';
    $notes         = trim($notesRaw);

    $itemNames  = isset($_POST['itemName']) ? $_POST['itemName'] : array();
    $itemQtys   = isset($_POST['itemQty']) ? $_POST['itemQty'] : array();
    $itemPrices = isset($_POST['itemPrice']) ? $_POST['itemPrice'] : array();

    if ($customerName === '' || $customerPhone === '') {
        $sentResult = '<div class="error">Customer name and phone are required. <a href="#form">Go back</a></div>';
    } else {
        // build order lines
        $lines = [];
        $grandTotal = 0.0;
        for ($i = 0; $i < count($itemNames); $i++){
            $iname = isset($itemNames[$i]) ? trim(strip_tags($itemNames[$i])) : '';
            $iqty  = isset($itemQtys[$i]) ? floatval($itemQtys[$i]) : 0;
            $iprice= isset($itemPrices[$i]) ? floatval($itemPrices[$i]) : 0.0;
            if ($iname === '' || $iqty <= 0) continue;
            $lineTotal = $iqty * $iprice;
            $grandTotal += $lineTotal;
            $lines[] = ['name'=>$iname,'qty'=>$iqty,'price'=>$iprice,'lineTotal'=>$lineTotal];
        }

        if (count($lines) === 0) {
            $sentResult = '<div class="error">No valid items found. <a href="#form">Go back</a></div>';
        } else {
            // --- save CSV row ---
            $timestamp = date('Y-m-d H:i:s');
            $items_json = json_encode($lines, JSON_UNESCAPED_UNICODE|JSON_UNESCAPED_SLASHES);
            $csv_row = [$timestamp, $customerName, $customerPhone, number_format($grandTotal,2), $items_json, $notes];
            save_csv($csv_file, $csv_row);

            // --- prepare email body ---
            $subject = "New order — $shopName (" . date('Y-m-d H:i:s') . ")";
            $body = "<html><body>";
            $body .= "<h2>New Order — " . h($shopName) . "</h2>";
            $body .= "<p><strong>Customer:</strong> " . h($customerName) . "<br>";
            $body .= "<strong>Phone:</strong> " . h($customerPhone) . "<br>";
            $body .= "<strong>Notes:</strong> " . nl2br(h($notes)) . "</p>";
            $body .= "<table border='1' cellpadding='6' cellspacing='0' style='border-collapse:collapse'>";
            $body .= "<tr><th>Item</th><th>Qty</th><th>Unit price (₹)</th><th>Line total (₹)</th></tr>";
            foreach ($lines as $ln){
                $body .= "<tr>";
                $body .= "<td>" . h($ln['name']) . "</td>";
                $body .= "<td style='text-align:right'>" . number_format($ln['qty'],0) . "</td>";
                $body .= "<td style='text-align:right'>" . number_format($ln['price'],2) . "</td>";
                $body .= "<td style='text-align:right'>" . number_format($ln['lineTotal'],2) . "</td>";
                $body .= "</tr>";
            }
            $body .= "</table>";
            $body .= "<p style='font-weight:700'>Grand total: ₹ " . number_format($grandTotal,2) . "</p>";
            $body .= "<p><small>Shop: " . h($shopName) . " | Contact: " . h($shopPhone) . " | Address: " . h($shopAddress) . "</small></p>";
            $body .= "</body></html>";

            // --- try to send via PHPMailer if installed (composer) ---
            $sent = false;
            $errorInfo = '';
            if (file_exists(__DIR__ . '/vendor/autoload.php')) {
                try {
                    require __DIR__ . '/vendor/autoload.php';
                    // use PHPMailer
                    $mail = new PHPMailer\PHPMailer\PHPMailer(true);
                    $mail->isSMTP();
                    $mail->Host       = $smtpHost;
                    $mail->SMTPAuth   = true;
                    $mail->Username   = $smtpUser;
                    $mail->Password   = $smtpPass;
                    if (!empty($smtpSecure)) $mail->SMTPSecure = $smtpSecure;
                    $mail->Port       = $smtpPort;
                    $mail->setFrom($smtpUser, $shopName . ' Orders');
                    $mail->addAddress($to_email);
                    $mail->isHTML(true);
                    $mail->Subject = $subject;
                    $mail->Body    = $body;
                    $mail->AltBody = strip_tags(str_replace(["<br>", "<br/>", "<br />"], "\n", $body));
                    $mail->send();
                    $sent = true;
                } catch (Exception $e) {
                    $sent = false;
                    $errorInfo = 'PHPMailer error: ' . (isset($mail) ? $mail->ErrorInfo : $e->getMessage());
                }
            }

            // --- fallback to PHP mail() if PHPMailer not installed or failed ---
            if (!$sent) {
                // prepare headers for mail()
                $headers  = "MIME-Version: 1.0\r\n";
                $headers .= "Content-type: text/html; charset=UTF-8\r\n";
                $headers .= "From: " . $shopName . " <" . h($shopPhone) . "@example.com>\r\n";
                // Note: PHP mail may be blocked on many hosts unless configured
                if (@mail($to_email, $subject, $body, $headers)) {
                    $sent = true;
                } else {
                    if ($errorInfo === '') $errorInfo = 'mail() failed or not configured on server.';
                }
            }

            if ($sent) {
                $sentResult = '<div class="success">Order sent successfully to <strong>' . h($to_email) . '</strong>.<br><a href="index.php">Place another order</a></div>';
            } else {
                $sentResult = '<div class="error">Order saved to CSV, but email could not be sent. Error: ' . h($errorInfo) . '<br>Orders stored in orders.csv</div>';
            }
        }
    }
}
?>
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title><?php echo h($shopName); ?> — Place Order</title>
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
  <h1><?php echo h($shopName); ?></h1>
  <div class="meta">
    <strong>Contact:</strong> <?php echo h($shopPhone); ?> &nbsp; | &nbsp;
    <strong>Email:</strong> <?php echo h($to_email); ?>
    <div class="note"><strong>Address:</strong> <?php echo h($shopAddress); ?></div>
  </div>
</header>

<div class="container">
  <h2>Place an order</h2>
  <p class="note">Add oil or grease items below. Enter unit price and quantity — totals calculate automatically.</p>

  <?php if ($sentResult) echo $sentResult; ?>

  <form id="orderForm" method="post" action="#form" novalidate>
    <div id="form">
      <label for="customerName">Customer name *</label>
      <input id="customerName" name="customerName" type="text" required placeholder="Customer full name" value="<?php echo isset($_POST['customerName']) ? h($_POST['customerName']) : ''; ?>" />

      <label for="customerPhone">Customer phone *</label>
      <input id="customerPhone" name="customerPhone" type="text" required placeholder="7777777777" value="<?php echo isset($_POST['customerPhone']) ? h($_POST['customerPhone']) : ''; ?>" />

      <label for="notes">Notes (optional)</label>
      <textarea id="notes" name="notes" placeholder="Delivery instructions, preferred time..."><?php echo isset($_POST['notes']) ? h($_POST['notes']) : ''; ?></textarea>

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
        <div class="muted">Orders will be emailed to <strong><?php echo h($to_email); ?></strong></div>
      </div>

      <input type="hidden" name="shopName" value="<?php echo h($shopName); ?>" />
      <input type="hidden" name="shopContact" value="<?php echo h($shopPhone); ?>" />
      <input type="hidden" name="shopAddress" value="<?php echo h($shopAddress); ?>" />
    </div>
  </form>
</div>

<script>
(function(){
  const itemsBody = document.getElementById('itemsBody');
  const grandTotalEl = document.getElementById('grandTotal');
  const form = document.getElementById('orderForm');

  function currency(v){ return '₹ ' + Number(v || 0).toFixed(2); }

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

  function escapeHtml(s){ return String(s||'').replaceAll('"','&quot;').replaceAll('<','&lt;').replaceAll('>','&gt;'); }

  // populate a default row
  createRow('',1,0);

  document.getElementById('addRow').addEventListener('click', () => createRow('',1,0));
  document.getElementById('clearRows').addEventListener('click', () => {
    itemsBody.innerHTML = '';
    createRow('',1,0);
    updateGrandTotal();
  });

  form.addEventListener('submit', function(e){
    const name = form.customerName.value.trim();
    const phone = form.customerPhone.value.trim();
    if (!name){ e.preventDefault(); alert('Please enter customer name.'); return; }
    if (!phone){ e.preventDefault(); alert('Please enter customer phone.'); return; }
    const rows = document.querySelectorAll('#itemsBody tr');
    if (rows.length === 0){ e.preventDefault(); alert('Add at least one item.'); return; }
    for (const tr of rows){
      const iname = tr.querySelector('input[name="itemName[]"]').value.trim();
      const qty = Number(tr.querySelector('input[name="itemQty[]"]').value);
      const price = Number(tr.querySelector('input[name="itemPrice[]"]').value);
      if (!iname){ e.preventDefault(); alert('Item name cannot be empty.'); return; }
      if (!Number.isFinite(qty) || qty <= 0){ e.preventDefault(); alert('Quantity must be at least 1.'); return; }
      if (!Number.isFinite(price) || price < 0){ e.preventDefault(); alert('Price must be 0 or more.'); return; }
    }
    // allow submit
  });
})();
</script>

</body>
</html>

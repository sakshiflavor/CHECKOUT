<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Secure Checkout</title>
    <style>
        body { font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; background: #f4f6f9; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; box-sizing: border-box; color: #2d3748; margin: 0; }
        .checkout-box { background: white; padding: 30px 20px; border-radius: 16px; width: 100%; max-width: 420px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); border: 1px solid #edf2f7; text-align: center; box-sizing: border-box; }
        h2 { margin-top: 0; color: #1a202c; font-weight: 600; font-size: 22px; }
        .btn { background: #2ed573; color: white; border: none; padding: 12px; border-radius: 8px; width: 100%; font-weight: 600; cursor: pointer; font-size: 14px; margin-top: 12px; transition: background 0.2s; box-sizing: border-box; }
        .btn:hover { background: #26af5f; }
        .btn-map { background: #3182ce; }
        .btn-map:hover { background: #2b6cb0; }
        .btn-back { background: #718096; margin-top: 10px; }
        .btn-back:hover { background: #4a5568; }
        .summary-line { display: flex; justify-content: space-between; margin: 10px 0; font-size: 14px; color: #4a5568; }
        .field-label { display: block; font-size: 13px; font-weight: 600; color: #4a5568; margin: 12px 0 4px 0; text-align: left; }
        .field-input { width: 100%; padding: 10px; border: 1px solid #cbd5e0; border-radius: 8px; font-size: 14px; box-sizing: border-box; }
        .field-input:focus { outline: none; border-color: #3182ce; }
        .location-status { background: #e2e8f0; padding: 12px; border-radius: 8px; font-size: 13px; color: #4a5568; margin: 15px 0; word-break: break-all; font-weight: 500; text-align: left; }
    </style>
</head>
<body>

 <div class="checkout-box">
        <h2>Order Checkout</h2>
        <p style="font-size: 13px; color: #718096; margin-bottom: 20px;">We require your live Google Maps location pin to dispatch your order.</p>
        <button class="btn btn-map" onclick="fetchLiveLocation()">📍 Fetch Live Google Maps Pin</button>
      
        <div id="locationStatusDisplay" class="location-status">
            ❌ No location pinned yet. Please click the button above.
        </div>

        <label class="field-label" for="towerInput">Tower / Block No.</label>
        <input id="towerInput" class="field-input" type="text" maxlength="30" placeholder="e.g. Tower B">
        <label class="field-label" for="flatInput">Flat No.</label>
        <input id="flatInput" class="field-input" type="text" maxlength="30" placeholder="e.g. 704">

        <hr style="border: 0; border-top: 1px solid #edf2f7; margin: 20px 0;">
        <div id="checkoutSummary" style="text-align: left;"></div>

        <button class="btn" onclick="placeOrder()">Place Order Now</button>
        <button class="btn btn-back" onclick="returnToMenu()">← Return to Menu</button>
    </div>

<script>
    const FIREBASE_URL = "https://test-d34cf-default-rtdb.europe-west1.firebasedatabase.app";
    
    // Shop Coordinates (Update these with your precise shop coordinates)
    const SHOP_LAT = 13.067825;
    const SHOP_LNG = 77.5928152;

    const phone = localStorage.getItem('activeCustomerPhone') || 'Customer_' + Math.floor(Math.random() * 9000 + 1000);
    let cartKey = 'cart_' + phone;
    let cart = JSON.parse(localStorage.getItem(cartKey) || '{}');
    let deliveryCharge = 0;
    let distanceKm = null; // unknown until the location is pinned
    // Defaults used until/unless the admin's settings load from Firebase
    let rules = { baseKm: 3, basePrice: 30, extraPricePerKm: 15, freeDeliveryAbove: 300 };
    let verifiedMapsLink = '';

    // Function to calculate actual driving distance via road networks
    async function getDrivingDistance(lat1, lon1, lat2, lon2) {
        try {
            // OSRM routing API (free, calculates actual road distance)
            const url = `https://router.project-osrm.org/route/v1/driving/${lon1},${lat1};${lon2},${lat2}?overview=false`;
            let response = await fetch(url);
            let data = await response.json();
            
            if (data.routes && data.routes.length > 0) {
                // distance is returned in meters, convert to kilometers
                return data.routes[0].distance / 1000;
            }
        } catch (e) {
            console.log("Routing network error, using straight-line backup.");
        }
        
        // Fallback: Haversine straight-line formula if network fails
        const R = 6371;
        const dLat = (lat2 - lat1) * Math.PI / 180;
        const dLon = (lon2 - lon1) * Math.PI / 180;
        const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                  Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * 
                  Math.sin(dLon/2) * Math.sin(dLon/2);
        return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    }

    async function loadRules() {
        try {
            let res = await fetch(`${FIREBASE_URL}/settings/delivery.json`);
            let data = await res.json();
            if (data) {
                // Accept only valid non-negative numbers; otherwise keep the default for that field
                const num = (v, d) => (typeof v === 'number' && isFinite(v) && v >= 0) ? v : d;
                rules = {
                    baseKm: num(data.baseKm, rules.baseKm),
                    basePrice: num(data.basePrice, rules.basePrice),
                    extraPricePerKm: num(data.extraPricePerKm, rules.extraPricePerKm),
                    freeDeliveryAbove: num(data.freeDeliveryAbove, rules.freeDeliveryAbove)
                };
            }
        } catch (e) {
            console.log("Could not load rules from Firebase, using defaults.");
        }
    }

    function isFreeDelivery(itemTotal) {
        return itemTotal > 0 && rules.freeDeliveryAbove > 0 && itemTotal >= rules.freeDeliveryAbove;
    }

    function calcDeliveryCharge(itemTotal) {
        if (itemTotal <= 0) return 0;
        if (isFreeDelivery(itemTotal)) return 0;
        if (distanceKm === null || distanceKm <= rules.baseKm) return rules.basePrice;
        return rules.basePrice + Math.ceil((distanceKm - rules.baseKm) * rules.extraPricePerKm);
    }

    function renderSummary() {
        const container = document.getElementById('checkoutSummary');
        let itemTotal = 0;
        let html = '<p style="font-weight:600; color:#1a202c; margin-bottom:12px;">Review Order Summary:</p>';

        let hasItems = false;
        for (let id in cart) {
            hasItems = true;
            let item = cart[id];
            itemTotal += item.price * item.quantity;
            html += `<div class="summary-line"><span>${item.name} (x${item.quantity})</span><span>₹${item.price * item.quantity}</span></div>`;
        }

        if (!hasItems) {
            html += '<p style="color: #e53e3e; font-size: 13px; text-align: center;">Your cart is empty. Add items first.</p>';
        }

        deliveryCharge = calcDeliveryCharge(itemTotal);
        let grandTotal = itemTotal > 0 ? itemTotal + deliveryCharge : 0;

        let feeText;
        if (isFreeDelivery(itemTotal)) {
            feeText = '<span style="color:#2ed573; font-weight:600;">FREE</span>';
        } else {
            feeText = `₹${deliveryCharge}` + (distanceKm === null && itemTotal > 0 ? ' <small style="color:#a0aec0;">(updates after location pin)</small>' : '');
        }

        html += `<hr style="border:0; border-top:1px dashed #cbd5e0; margin:15px 0;">`;
        html += `<div class="summary-line"><span>Item Total:</span><span>₹${itemTotal}</span></div>`;
        html += `<div class="summary-line"><span>Delivery Fee:</span><span>${feeText}</span></div>`;
        if (itemTotal > 0 && rules.freeDeliveryAbove > 0 && itemTotal < rules.freeDeliveryAbove) {
            html += `<div style="font-size:12px; color:#2b6cb0; text-align:right; margin-bottom:6px;">Add ₹${rules.freeDeliveryAbove - itemTotal} more for FREE delivery</div>`;
        }
        html += `<div class="summary-line" style="font-weight:bold; font-size:16px; color:#1a202c;"><span>Grand Total:</span><span>₹${grandTotal}</span></div>`;

        container.innerHTML = html;
    }

    async function fetchLiveLocation() {
        if (!navigator.geolocation) {
            alert('Geolocation is not supported by your browser.');
            return;
        }

        const statusBox = document.getElementById('locationStatusDisplay');
        statusBox.style.background = '#ebf8ff';
        statusBox.style.color = '#2b6cb0';
        statusBox.innerHTML = '🔄 Calculating road route distance...';

        navigator.geolocation.getCurrentPosition(async (position) => {
            const lat = position.coords.latitude;
            const lng = position.coords.longitude;
            verifiedMapsLink = `https://www.google.com/maps?q=${lat},${lng}`;
            
            // Get actual driving distance via map roads
            distanceKm = await getDrivingDistance(SHOP_LAT, SHOP_LNG, lat, lng);

            // Refresh the admin's delivery rules, then recompute the fee
            await loadRules();
            renderSummary();

            statusBox.style.background = '#c6f6d5';
            statusBox.style.color = '#22543d';
            statusBox.innerHTML = `✅ Location Pinned (${distanceKm.toFixed(1)} km driving distance)!<br><a href="${verifiedMapsLink}" target="_blank" style="color:#2b6cb0; font-size:12px;">View Map ↗</a>`;
        }, () => {
            statusBox.style.background = '#fed7d7';
            statusBox.style.color = '#9b2c2c';
            statusBox.innerHTML = '❌ Unable to retrieve location. Please allow GPS permissions.';
        }, { enableHighAccuracy: true });
    }

    async function placeOrder() {
        if (!verifiedMapsLink) {
            alert('Error: You must fetch your Google Maps location pin before checking out!');
            return;
        }

        let itemTotal = Object.values(cart).reduce((sum, i) => sum + (i.price * i.quantity), 0);
        if (itemTotal === 0) {
            alert('Your cart is empty!');
            return;
        }

        const tower = document.getElementById('towerInput').value.trim();
        const flat = document.getElementById('flatInput').value.trim();
        if (!tower || !flat) {
            alert('Please enter your Tower and Flat number.');
            return;
        }
        saveAddress();
        deliveryCharge = calcDeliveryCharge(itemTotal);

        let newOrder = {
            id: 'ORD-' + Math.floor(Math.random() * 90000 + 10000),
            phone: phone,
            location: verifiedMapsLink,
            tower: tower,
            flat: flat,
            items: Object.values(cart),
            itemTotal: itemTotal,
            deliveryCharge: deliveryCharge,
            grandTotal: itemTotal + deliveryCharge,
            time: new Date().toLocaleTimeString()
        };

        try {
            let res = await fetch(`${FIREBASE_URL}/orders.json`);
            let orders = await res.json();
            if (!Array.isArray(orders)) orders = [];

            orders.unshift(newOrder);

            await fetch(`${FIREBASE_URL}/orders.json`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(orders)
            });

            localStorage.removeItem(cartKey);
            alert('Order placed successfully!');
            window.location.href = 'https://github.com/sakshiflavor/MENU';
        } catch (e) {
            alert('Network error placing order. Please check your connection.');
        }
    }

    function returnToMenu() {
        window.location.href = 'https://github.com/sakshiflavor/MENU';
    }

    // Remember tower/flat per customer so they don't retype it
    const addressKey = 'address_' + phone;
    function saveAddress() {
        localStorage.setItem(addressKey, JSON.stringify({
            tower: document.getElementById('towerInput').value.trim(),
            flat: document.getElementById('flatInput').value.trim()
        }));
    }
    (function restoreAddress() {
        try {
            const saved = JSON.parse(localStorage.getItem(addressKey) || '{}');
            document.getElementById('towerInput').value = saved.tower || '';
            document.getElementById('flatInput').value = saved.flat || '';
        } catch (e) {}
        document.getElementById('towerInput').addEventListener('input', saveAddress);
        document.getElementById('flatInput').addEventListener('input', saveAddress);
    })();

    renderSummary();
    loadRules().then(renderSummary);
</script>
</body>
</html>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chaat Puchka - Checkout</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-amber-50/50 m-0 p-4 sm:p-5 font-sans">
    <div class="max-w-xl mx-auto bg-white p-6 rounded-3xl shadow-lg border border-amber-100">
        <div class="flex items-center justify-between border-b border-slate-100 pb-4 mb-4">
            <div class="flex items-center space-x-3">
                <div class="w-10 h-10 flex-shrink-0 bg-black/90 rounded-xl p-1 shadow flex items-center justify-center">
                    <img src="https://i.postimg.cc/SsSZBQ9q/1000037254-removebg-preview.png" alt="Logo" class="w-full h-full object-contain">
                </div>
                <h1 class="text-lg sm:text-xl font-black text-slate-900 tracking-tight">Chaat Puchka</h1>
            </div>
            <span class="text-xs bg-amber-100 text-amber-900 px-3 py-1 rounded-full font-bold">Checkout</span>
        </div>
        
        <div id="branch-info" class="bg-amber-50 border border-amber-200 p-3.5 rounded-xl mb-5 text-sm text-amber-900 font-medium">Loading branch and delivery details...</div>

        <h3 class="font-bold text-slate-800 mb-2">Your Cart</h3>
        <div id="cart-items-container" class="space-y-2 mb-4">
            <p class="text-slate-500 text-sm">Loading cart items...</p>
        </div>

        <div class="bg-amber-50/40 p-4 rounded-2xl border border-amber-100 space-y-2 text-sm">
            <div class="flex justify-between">
                <span class="text-slate-600 font-medium">Subtotal:</span>
                <span id="subtotal-amount" class="font-bold">₹0</span>
            </div>
            <div class="flex justify-between">
                <span class="text-slate-600 font-medium">Delivery Charge:</span>
                <span id="delivery-amount" class="font-bold">₹0</span>
            </div>
            <div class="flex justify-between pt-2 border-t border-amber-200 font-black text-base text-emerald-600">
                <span>Total Amount:</span>
                <span id="total-amount">₹0</span>
            </div>
        </div>

        <h3 class="font-bold text-slate-800 mt-5 mb-2">Delivery Details</h3>
        <label class="block text-xs uppercase font-extrabold text-amber-900 mb-1">Delivery Address:</label>
        <textarea id="deliveryAddress" rows="3" placeholder="Enter your complete delivery address..." class="w-full p-3.5 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-amber-500 box-border mb-4"></textarea>

        <button onclick="placeOrder()" class="w-full bg-emerald-600 hover:bg-emerald-700 text-white font-black p-3.5 rounded-xl shadow-md transition">Place Order</button>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
        import { getDatabase, ref, get, push, remove } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-database.js";

        const firebaseConfig = {
            databaseURL: "https://teat-2-4b868-default-rtdb.europe-west1.firebasedatabase.app/"
        };
        const app = initializeApp(firebaseConfig);
        const db = getDatabase(app);

        const urlParams = new URLSearchParams(window.location.search);
        const customerPhone = urlParams.get('phone');

        let sessionData = null;
        let deliveryFee = 0;
        let subtotal = 0;

        if (!customerPhone) {
            window.location.href = 'customer-login.html';
        } else {
            get(ref(db, `sessions/${customerPhone}`)).then((snapshot) => {
                if (snapshot.exists()) {
                    sessionData = snapshot.val();
                    renderCheckout();
                } else {
                    window.location.href = 'https://sakshiflavor.github.io/LOGIN/';
                }
            });
        }

        function renderCheckout() {
            const branchInfoDiv = document.getElementById('branch-info');
            const cartContainer = document.getElementById('cart-items-container');

            if (!sessionData.branchId || !sessionData.cart || sessionData.cart.length === 0) {
                branchInfoDiv.innerHTML = '<span style="color:red;">Your cart is empty or branch information is missing.</span>';
                return;
            }

            branchInfoDiv.innerHTML = `Ordering from: <strong>${sessionData.branchName}</strong> (Phone: ${sessionData.phone})`;

            cartContainer.innerHTML = '';
            subtotal = 0;
            sessionData.cart.forEach(item => {
                const itemTotal = item.price * (item.qty || 1);
                subtotal += itemTotal;
                cartContainer.innerHTML += `
                    <div class="flex justify-between items-center text-sm p-3 bg-white rounded-xl border border-slate-100">
                        <span class="font-medium">${item.name} (x${item.qty || 1})</span>
                        <span class="font-bold text-slate-800">₹${itemTotal}</span>
                    </div>`;
            });

            document.getElementById('subtotal-amount').innerText = `₹${subtotal}`;

            get(ref(db, `branches/${sessionData.branchId}/deliverySettings`)).then((settingsSnap) => {
                if (settingsSnap.exists()) {
                    const settings = settingsSnap.val();
                    deliveryFee = settings.baseCharge || 0;
                }
                document.getElementById('delivery-amount').innerText = `₹${deliveryFee}`;
                document.getElementById('total-amount').innerText = `₹${subtotal + deliveryFee}`;
            });
        }

        window.placeOrder = function() {
            const address = document.getElementById('deliveryAddress').value.trim();
            if (!address) {
                alert('Please enter your delivery address.');
                return;
            }

            const orderData = {
                phone: customerPhone,
                items: sessionData.cart,
                subtotal: subtotal,
                deliveryFee: deliveryFee,
                total: subtotal + deliveryFee,
                address: address,
                status: 'Preparing',
                timestamp: Date.now()
            };

            push(ref(db, `orders/${sessionData.branchId}`), orderData)
                .then(() => {
                    remove(ref(db, `sessions/${customerPhone}/cart`));
                    alert('Order placed successfully!');
                    window.location.href = `customer-menu.html?phone=${customerPhone}`;
                });
        };
    </script>
</body>
</html>

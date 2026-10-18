{
  "name": "ironfit_store",
  "version": "1.0.0",
  "description": "Magazin online IronFit UK cu SQLite",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sqlite3": "^5.1.7"
  }
}
const express = require('express');
const path = require('path');
const sqlite3 = require('sqlite3').verbose();

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Configurare Bază de Date SQLite
const dbFile = path.join(__dirname, 'database.sqlite');
const db = new sqlite3.Database(dbFile, (err) => {
    if (err) {
        console.error('Eroare la conectarea bazei de date:', err.message);
    } else {
        console.log('Conectat la baza de date SQLite.');
        initDatabase();
    }
});

// Inițializare tabele și date de test
function initDatabase() {
    db.serialize(() => {
        db.run(`CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            price REAL NOT NULL,
            category TEXT NOT NULL,
            description TEXT
        )`);

        db.run(`CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            total REAL NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

        db.get("SELECT COUNT(*) as count FROM products", (err, row) => {
            if (row.count === 0) {
                const initialProducts = [
                    ['IronFit 20kg Olympic Barbell', 149.99, 'Echipamente', 'Bară olimpică profesională din oțel, ideală pentru antrenamente grele de putere.'],
                    ['Adjustable Dumbbells Set (2x 24kg)', 219.99, 'Echipamente', 'Set de gantere ajustabile rapid, perfecte pentru economisirea spațiului acasă.'],
                    ['Whey Protein Isolate (2kg - Chocolate)', 54.99, 'Suplimente', 'Proteină izolată de cea mai înaltă calitate pentru o recuperare musculară rapidă.'],
                    ['Heavy Duty Resistance Bands (Set of 5)', 19.99, 'Accesorii', 'Benzi elastice de rezistență durabilă pentru mobilitate și tonifiere.']
                ];
                
                const stmt = db.prepare("INSERT INTO products (name, price, category, description) VALUES (?, ?, ?, ?)");
                initialProducts.forEach(p => stmt.run(p));
                stmt.finalize();
                console.log('Produsele inițiale au fost adăugate în baza de date.');
            }
        });
    });
}

// API: Preluare produse (cu opțiune de căutare / filtrare după categorie)
app.get('/api/products', (req, res) => {
    const { category, search } = req.query;
    let query = "SELECT * FROM products WHERE 1=1";
    let params = [];

    if (category && category !== 'Toate') {
        query += " AND category = ?";
        params.push(category);
    }

    if (search) {
        query += " AND name LIKE ?";
        params.push(`%${search}%`);
    }

    db.all(query, params, (err, rows) => {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json(rows);
    });
});

// API: Detalii pentru un produs anume
app.get('/api/products/:id', (req, res) => {
    const productId = req.params.id;
    db.get("SELECT * FROM products WHERE id = ?", [productId], (err, row) => {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        if (!row) {
            res.status(404).json({ error: 'Produsul nu a fost găsit.' });
            return;
        }
        res.json(row);
    });
});

// API: Plasare comandă
app.post('/api/orders', (req, res) => {
    const { items, total, shippingAddress } = req.body;
    if (!items || items.length === 0) {
        return res.status(400).json({ error: 'Coșul este gol.' });
    }

    db.run("INSERT INTO orders (total) VALUES (?)", [total], function(err) {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        
        console.log('Comandă nouă primită pentru adresa UK:', shippingAddress);
        res.json({ message: 'Comandă înregistrată cu succes!', orderId: this.lastID });
    });
});

// API Admin: Adăugare produs nou
app.post('/api/admin/products', (req, res) => {
    const { name, price, category, description } = req.body;
    if (!name || !price || !category) {
        return res.status(400).json({ error: 'Numele, prețul și categoria sunt obligatorii.' });
    }

    db.run("INSERT INTO products (name, price, category, description) VALUES (?, ?, ?, ?)", 
        [name, price, category, description || ''], function(err) {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json({ message: 'Produs adăugat cu succes!', productId: this.lastID });
    });
});

// API Admin: Vizualizare comenzi primite
app.get('/api/admin/orders', (req, res) => {
    db.all("SELECT * FROM orders ORDER BY created_at DESC", [], (err, rows) => {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json(rows);
    });
});

// Pornire server
app.listen(PORT, () => {
    console.log(`Magazinul IronFit UK rulează pe http://localhost:${PORT}`);
});
<!DOCTYPE html>
<html lang="ro">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>IronFit UK | Premium Fitness Equipment & Supplements</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        :root {
            --primary: #ff4500;
            --dark: #121212;
            --card-bg: #1f1f1f;
            --light: #f4f4f4;
            --border: #333;
        }
        * { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        body { background-color: var(--dark); color: var(--light); }
        
        header { background: #1f1f1f; padding: 20px 40px; display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid var(--primary); position: sticky; top: 0; z-index: 100; }
        .logo { font-size: 24px; font-weight: bold; color: var(--primary); text-transform: uppercase; cursor: pointer; }
        nav { display: flex; gap: 20px; align-items: center; }
        nav a { color: var(--light); text-decoration: none; font-weight: 600; cursor: pointer; transition: 0.2s; }
        nav a:hover { color: var(--primary); }
        
        .cart-btn { background: none; border: none; color: var(--light); font-size: 18px; cursor: pointer; display: flex; align-items: center; gap: 8px; }
        .cart-badge { background: var(--primary); color: white; padding: 2px 8px; border-radius: 50%; font-size: 14px; }

        .hero { text-align: center; padding: 60px 20px; background: linear-gradient(rgba(0,0,0,0.7), rgba(0,0,0,0.7)), #222; }
        .hero h1 { font-size: 40px; margin-bottom: 15px; }
        .hero p { font-size: 18px; color: #ccc; margin-bottom: 25px; }
        
        .btn { background: var(--primary); color: #fff; padding: 12px 25px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; text-transform: uppercase; transition: 0.2s; width: 100%; text-align: center; }
        .btn:hover { background: #e03d00; }
        .btn-secondary { background: #333; }
        .btn-secondary:hover { background: #444; }

        .container { max-width: 1200px; margin: 40px auto; padding: 0 20px; }
        
        .filter-bar { display: flex; justify-content: space-between; margin-bottom: 30px; flex-wrap: wrap; gap: 15px; }
        .search-input, .filter-select { padding: 10px 15px; background: var(--card-bg); border: 1px solid var(--border); color: white; border-radius: 5px; font-size: 16px; }
        .search-input { width: 300px; }

        .products-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 25px; }
        .product-card { background: var(--card-bg); border-radius: 8px; padding: 20px; text-align: center; border: 1px solid var(--border); display: flex; flex-direction: column; justify-content: space-between; }
        .product-card h3 { margin: 15px 0 10px; font-size: 20px; cursor: pointer; }
        .product-card h3:hover { color: var(--primary); }
        .product-card .price { color: var(--primary); font-size: 18px; font-weight: bold; margin-bottom: 15px; }

        .page-section { display: none; }
        .page-section.active { display: block; }

        .cart-modal { position: fixed; top: 0; right: -400px; width: 380px; height: 100%; background: #1a1a1a; box-shadow: -5px 0 15px rgba(0,0,0,0.5); transition: 0.3s ease; z-index: 1000; display: flex; flex-direction: column; }
        .cart-modal.open { right: 0; }
        .cart-header { padding: 20px; background: #252525; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid var(--border); }
        .close-cart { background: none; border: none; color: white; font-size: 20px; cursor: pointer; }
        .cart-body { padding: 20px; flex-grow: 1; overflow-y: auto; }
        .cart-item { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; padding-bottom: 10px; border-bottom: 1px solid var(--border); }
        .cart-footer { padding: 20px; background: #252525; border-top: 1px solid var(--border); }
        .total-price { font-size: 18px; font-weight: bold; margin-bottom: 15px; display: flex; justify-content: space-between; }

        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: 600; font-size: 14px; }
        .form-control { width: 100%; padding: 10px; background: var(--card-bg); border: 1px solid var(--border); color: white; border-radius: 5px; }
        table { width: 100%; border-collapse: collapse; margin-top: 20px; background: var(--card-bg); }
        th, td { padding: 12px; border: 1px solid var(--border); text-align: left; }
        th { background: #252525; }
    </style>
</head>
<body>

    <header>
        <div class="logo" onclick="switchPage('home')"><i class="fa-solid fa-dumbbell"></i> IronFit UK</div>
        <nav>
            <a onclick="switchPage('home')">Magazin</a>
            <a onclick="switchPage('about')">Despre Noi</a>
            <a onclick="switchPage('contact')">Contact</a>
            <a onclick="switchPage('admin')" style="color: var(--primary);"><i class="fa-solid fa-lock"></i> Admin</a>
            <button class="cart-btn" onclick="toggleCart()">
                <i class="fa-solid fa-cart-shopping"></i> Coș <span class="cart-badge" id="cart-count">0</span>
            </button>
        </nav>
    </header>

    <div id="home-page" class="page-section active">
        <section class="hero">
            <h1>Construiește-ți Forța cu IronFit UK</h1>
            <p>Echipamente profesionale și suplimente de top pentru performanță maximă.</p>
            <button class="btn" style="width: auto;" onclick="scrollToProducts()">Cumpără Acum</button>
        </section>

        <div class="container" id="products-section">
            <div class="filter-bar">
                <input type="text" id="searchInput" class="search-input" placeholder="Caută produse..." oninput="filterProducts()">
                <select id="categorySelect" class="filter-select" onchange="filterProducts()">
                    <option value="Toate">Toate Categoriile</option>
                    <option value="Echipamente">Echipamente</option>
                    <option value="Suplimente">Suplimente</option>
                    <option value="Accesorii">Accesorii</option>
                </select>
            </div>

            <h2 style="margin-bottom: 30px; border-bottom: 2px solid var(--border); padding-bottom: 10px;">Produse Populare</h2>
            <div class="products-grid" id="products-container"></div>
        </div>
    </div>

    <div id="product-detail-page" class="page-section">
        <div class="container" id="product-detail-container"></div>
    </div>

    <div id="checkout-page" class="page-section">
        <div class="container" style="max-width: 600px;">
            <button class="btn btn-secondary" onclick="toggleCart()" style="width: auto; margin-bottom: 20px;"><i class="fa-solid fa-arrow-left"></i> Înapoi la Coș</button>
            <div style="background: var(--card-bg); padding: 30px; border-radius: 8px; border: 1px solid var(--border);">
                <h2 style="margin-bottom: 20px;">Detalii Livrare UK</h2>
                <form id="checkoutForm" onsubmit="submitOrder(event)">
                    <div class="form-group">
                        <label>Nume Complet</label>
                        <input type="text" id="shippingName" class="form-control" required placeholder="John Smith">
                    </div>
                    <div class="form-group">
                        <label>Adresă (Strada și Numărul)</label>
                        <input type="text" id="shippingAddress" class="form-control" required placeholder="10 Downing Street">
                    </div>
                    <div class="form-group">
                        <label>Oraș / Localitate</label>
                        <input type="text" id="shippingCity" class="form-control" required placeholder="London">
                    </div>
                    <div class="form-group">
                        <label>Cod Poștal UK</label>
                        <input type="text" id="shippingPostalCode" class="form-control" required placeholder="SW1A 2AA">
                    </div>
                    <button type="submit" class="btn" style="margin-top: 10px;">Plasează Comanda și Trimite la Furnizor</button>
                </form>
            </div>
        </div>
    </div>

    <div id="about-page" class="page-section">
        <div class="container">
            <h2>Despre IronFit UK</h2>
            <p style="margin-top: 15px; line-height: 1.6; color: #ccc;">
                IronFit UK s-a născut din pasiune pură pentru fitness. Oferim echipamente de cea mai înaltă rezistență și suplimente nutritive riguros testate pentru performanță maximă în Regatul Unit.
            </p>
        </div>
    </div>

    <div id="contact-page" class="page-section">
        <div class="container">
            <h2>Contactează-ne</h2>
            <div style="background: var(--card-bg); padding: 25px; border-radius: 8px; border: 1px solid var(--border); max-width: 600px; margin-top: 15px;">
                <p><strong>Email:</strong> support@ironfit.co.uk</p>
                <p style="margin-top: 10px;"><strong>Telefon:</strong> +44 20 7946 0912</p>
                <p style="margin-top: 10px;"><strong>Adresă:</strong> 42 Fitness Way, London, UK</p>
            </div>
        </div>
    </div>

    <div id="admin-page" class="page-section">
        <div class="container">
            <h2>Panou de Administrare</h2>
            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 40px; align-items: start; margin-top: 20px;">
                <div style="background: var(--card-bg); padding: 25px; border-radius: 8px; border: 1px solid var(--border);">
                    <h3>Adaugă Produs Nou</h3>
                    <form id="addProductForm" onsubmit="handleAddNewProduct(event)" style="margin-top: 15px;">
                        <div class="form-group">
                            <label>Nume Produs</label>
                            <input type="text" id="newProdName" class="form-control" required>
                        </div>
                        <div class="form-group">
                            <label>Preț (£)</label>
                            <input type="number" step="0.01" id="newProdPrice" class="form-control" required>
                        </div>
                        <div class="form-group">
                            <label>Categorie</label>
                            <select id="newProdCategory" class="form-control">
                                <option value="Echipamente">Echipamente</option>
                                <option value="Suplimente">Suplimente</option>
                                <option value="Accesorii">Accesorii</option>
                            </select>
                        </div>
                        <div class="form-group">
                            <label>Descriere</label>
                            <textarea id="newProdDesc" class="form-control" rows="3"></textarea>
                        </div>
                        <button type="submit" class="btn">Salvează Produsul</button>
                    </form>
                </div>
                <div>
                    <h3>Comenzi Primite</h3>
                    <div id="adminOrdersContainer"><p style="color: #ccc; margin-top: 15px;">Se încarcă comenzile...</p></div>
                </div>
            </div>
        </div>
    </div>

    <div class="cart-modal" id="cartModal">
        <div class="cart-header">
            <h3>Coșul Tău</h3>
            <button class="close-cart" onclick="toggleCart()">&times;</button>
        </div>
        <div class="cart-body" id="cart-items">
            <p style="color: #777; text-align: center; margin-top: 40px;">Coșul este gol.</p>
        </div>
        <div class="cart-footer">
            <div class="total-price">
                <span>Total:</span>
                <span id="cart-total">£0.00</span>
            </div>
            <button class="btn" onclick="goToCheckout()">Mergi la Checkout</button>
        </div>
    </div>

    <script>
        let cart = [];
        let allProducts = [];

        function switchPage(pageId) {
            document.querySelectorAll('.page-section').forEach(sec => sec.classList.remove('active'));
            document.getElementById(pageId + '-page').classList.add('active');
            window.scrollTo(0, 0);

            if (pageId === 'home') loadProducts();
            else if (pageId === 'admin') loadAdminOrders();
        }

        function loadProducts() {
            fetch('/api/products')
                .then(res => res.json())
                .then(products => {
                    allProducts = products;
                    displayProducts(products);
                });
        }

        function displayProducts(products) {
            const container = document.getElementById('products-container');
            if (products.length === 0) {
                container.innerHTML = '<p style="color: #777;">Nu s-au găsit produse.</p>';
                return;
            }
            container.innerHTML = products.map(p => `
                <div class="product-card">
                    <div>
                        <h3 onclick="viewProductDetail(${p.id})">${p.name}</h3>
                        <p style="font-size: 14px; color: #aaa; margin-bottom: 10px;">${p.category}</p>
                    </div>
                    <div>
                        <div class="price">£${p.price.toFixed(2)}</div>
                        <button class="btn" style="margin-bottom: 8px;" onclick="addToCart(${p.id})">Adaugă în Coș</button>
                        <button class="btn btn-secondary" onclick="viewProductDetail(${p.id})"><i class="fa-solid fa-eye"></i> Detalii</button>
                    </div>
                </div>
            `).join('');
        }

        function filterProducts() {
            const search = document.getElementById('searchInput').value;
            const category = document.getElementById('categorySelect').value;
            fetch(`/api/products?category=${encodeURIComponent(category)}&search=${encodeURIComponent(search)}`)
                .then(res => res.json())
                .then(products => displayProducts(products));
        }

        function viewProductDetail(productId) {
            fetch(`/api/products/${productId}`)
                .then(res => res.json())
                .then(p => {
                    document.getElementById('product-detail-container').innerHTML = `
                        <button class="btn btn-secondary" onclick="switchPage('home')" style="width: auto; margin-bottom: 20px;"><i class="fa-solid fa-arrow-left"></i> Înapoi</button>
                        <div style="background: var(--card-bg); padding: 30px; border-radius: 8px; border: 1px solid var(--border);">
                            <h2>${p.name}</h2>
                            <p style="color: var(--primary); font-weight: bold; margin: 10px 0;">Categorie: ${p.category}</p>
                            <div style="font-size: 24px; font-weight: bold; color: var(--primary); margin-bottom: 20px;">£${p.price.toFixed(2)}</div>
                            <p style="line-height: 1.6; color: #ccc; margin-bottom: 25px;">${p.description || 'Fără descriere.'}</p>
                            <button class="btn" style="width: auto;" onclick="addToCart(${p.id})">Adaugă în Coș</button>
                        </div>
                    `;
                    switchPage('product-detail');
                });
        }

        function toggleCart() {
            document.getElementById('cartModal').classList.toggle('open');
        }

        function scrollToProducts() {
            document.getElementById('products-section').scrollIntoView({ behavior: 'smooth' });
        }

        function addToCart(productId) {
            let product = allProducts.find(p => p.id === productId);
            if (!product) {
                fetch(`/api/products/${productId}`).then(res => res.json()).then(p => addProductToCartDirect(p));
                return;
            }
            addProductToCartDirect(product);
        }

        function addProductToCartDirect(product) {
            const existingItem = cart.find(item => item.id === product.id);
            if (existingItem) existingItem.quantity += 1;
            else cart.push({ ...product, quantity: 1 });
            updateCartUI();
            toggleCart();
        }

        function removeFromCart(productId) {
            cart = cart.filter(item => item.id !== productId);
            updateCartUI();
        }

        function updateCartUI() {
            const countBadge = document.getElementById('cart-count');
            const cartItemsContainer = document.getElementById('cart-items');
            const cartTotalElement = document.getElementById('cart-total');

            const totalCount = cart.reduce((sum, item) => sum + item.quantity, 0);
            countBadge.innerText = totalCount;

            if (cart.length === 0) {
                cartItemsContainer.innerHTML = '<p style="color: #777; text-align: center; margin-top: 40px;">Coșul este gol.</p>';
                cartTotalElement.innerText = '£0.00';
                return;
            }

            let totalSum = 0;
            cartItemsContainer.innerHTML = cart.map(item => {
                totalSum += item.price * item.quantity;
                return `
                    <div class="cart-item">
                        <div>
                            <h4>${item.name}</h4>
                            <small>£${item.price.toFixed(2)} x ${item.quantity}</small>
                        </div>
                        <button style="background:none; border:none; color:#ff4500; cursor:pointer;" onclick="removeFromCart(${item.id})"><i class="fa-solid fa-trash"></i></button>
                    </div>
                `;
            }).join('');
            cartTotalElement.innerText = `£${totalSum.toFixed(2)}`;
        }

        function goToCheckout() {
            if (cart.length === 0) {
                alert('Coșul tău este gol!');
                return;
            }
            toggleCart();
            switchPage('checkout');
        }

        function submitOrder(event) {
            event.preventDefault();
            const totalSum = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);

            const orderData = {
                items: cart,
                total: totalSum,
                shippingAddress: {
                    fullName: document.getElementById('shippingName').value,
                    address: document.getElementById('shippingAddress').value,
                    city: document.getElementById('shippingCity').value,
                    postalCode: document.getElementById('shippingPostalCode').value,
                    country: 'UK'
                }
            };

            fetch('/api/orders', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(orderData)
            })
            .then(res => res.json())
            .then(data => {
                alert(`Comanda #${data.orderId} a fost plasată cu succes și trimisă automat la furnizor!`);
                cart = [];
                updateCartUI();
                document.getElementById('checkoutForm').reset();
                switchPage('home');
            })
            .catch(err => {
                console.error('Eroare:', err);
                alert('A apărut o problemă la plasarea comenzii.');
            });
        }

        function handleAddNewProduct(event) {
            event.preventDefault();
            const name = document.getElementById('newProdName').value;
            const price = parseFloat(document.getElementById('newProdPrice').value);
            const category = document.getElementById('newProdCategory').value;
            const description = document.getElementById('newProdDesc').value;

            fetch('/api/admin/products', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ name, price, category, description })
            })
            .then(res => res.json())
            .then(data => {
                alert(data.message);
                document.getElementById('addProductForm').reset();
                loadProducts();
            })
            .catch(err => alert('Eroare la adăugarea produsului.'));
        }

        function loadAdminOrders() {
            fetch('/api/admin/orders')
                .then(res => res.json())
                .then(orders => {
                    const container = document.getElementById('adminOrdersContainer');
                    if (orders.length === 0) {
                        container.innerHTML = '<p style="color: #ccc; margin-top: 15px;">Nu există comenzi înregistrate.</p>';
                        return;
                    }
                    container.innerHTML = `
                        <table>
                            <thead><tr><th>ID</th><th>Total</th><th>Dată</th></tr></thead>
                            <tbody>
                                ${orders.map(o => `<tr><td>#${o.id}</td><td>£${o.total.toFixed(2)}</td><td>${o.created_at}</td></tr>`).join('')}
                            </tbody>
                        </table>
                    `;
                });
        }

        loadProducts();
    </script>
</body>
</html>

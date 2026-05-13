contract-archive/
├── backend/
│   ├── models/
│   ├── routes/
│   ├── middleware/
│   ├── uploads/
│   ├── db.js
│   ├── server.js
│   └── .env
├── frontend/
│   ├── index.html
│   ├── dashboard.html
│   ├── admin.html
│   ├── profile.html
│   ├── css/
│   └── js/
└── database.sql
{
  "name": "contract-archive-backend",
  "version": "1.0.0",
  "description": "Rental contract digital archive",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "multer": "^1.4.5-lts.1",
    "dotenv": "^16.3.1",
    "cors": "^2.8.5",
    "express-validator": "^7.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
PORT=5000
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_HOST=localhost
DB_PORT=5432
DB_NAME=contract_archive
JWT_SECRET=supersecretkey
CREATE DATABASE contract_archive;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    fullname VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE contracts (
    id SERIAL PRIMARY KEY,
    client_name VARCHAR(150) NOT NULL,
    contract_id VARCHAR(50) UNIQUE NOT NULL,
    sign_date DATE NOT NULL,
    file_path VARCHAR(255) NOT NULL,
    original_name VARCHAR(255),
    uploaded_by INTEGER REFERENCES users(id),
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE activity_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    action TEXT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Default admin (password: admin123)
INSERT INTO users (fullname, email, password, role) VALUES 
('Admin', 'admin@example.com', '$2a$10$N9qo8uLOickgx2ZMRZoMy.Mr6v6q5BqXbFmqrkO9JkYqVKtG5P1iW', 'admin');
const { Pool } = require('pg');
require('dotenv').config();

const pool = new Pool({
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    database: process.env.DB_NAME,
});

module.exports = pool;
const express = require('express');
const cors = require('cors');
const path = require('path');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const contractRoutes = require('./routes/contracts');
const adminRoutes = require('./routes/admin');
const userRoutes = require('./routes/user');

const app = express();

app.use(cors());
app.use(express.json());
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

app.use('/api/auth', authRoutes);
app.use('/api/contracts', contractRoutes);
app.use('/api/admin', adminRoutes);
app.use('/api/user', userRoutes);

app.listen(process.env.PORT, () => {
    console.log(Server running on port ${process.env.PORT});
});
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const pool = require('../db');

const router = express.Router();

router.post('/register', async (req, res) => {
    const { fullname, email, password } = req.body;
    const hashed = await bcrypt.hash(password, 10);
    try {
        const result = await pool.query(
            'INSERT INTO users (fullname, email, password) VALUES ($1,$2,$3) RETURNING id, email, role',
            [fullname, email, hashed]
        );
        res.json({ success: true, user: result.rows[0] });
    } catch (err) {
        res.status(400).json({ error: 'Email already exists' });
    }
});

router.post('/login', async (req, res) => {
    const { email, password } = req.body;
    const user = await pool.query('SELECT * FROM users WHERE email=$1', [email]);
    if (user.rows.length === 0) return res.status(401).json({ error: 'Invalid credentials' });
    const match = await bcrypt.compare(password, user.rows[0].password);
    if (!match) return res.status(401).json({ error: 'Invalid credentials' });
    const token = jwt.sign({ id: user.rows[0].id, role: user.rows[0].role }, process.env.JWT_SECRET);
    res.json({ token, user: { id: user.rows[0].id, fullname: user.rows[0].fullname, email: user.rows[0].email, role: user.rows[0].role } });
});

module.exports = router;
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Contract Archive | Smart Ijara Boshqaruvi</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css" rel="stylesheet">
    <style>
        body {
            transition: background-color 0.3s, color 0.3s;
        }
        .dark-mode {
            background-color: #1a202c;
            color: #edf2f7;
        }
        .dark-mode .card {
            background-color: #2d3748;
            color: #e2e8f0;
        }
        .card {
            transition: transform 0.2s;
        }
        .card:hover {
            transform: translateY(-5px);
        }
    </style>
</head>
<body class="bg-gray-50 font-sans">

    <!-- Navbar -->
    <nav class="bg-white shadow-md dark:bg-gray-800 fixed w-full z-10 top-0">
        <div class="container mx-auto px-4 py-3 flex justify-between items-center">
            <div class="text-2xl font-bold text-blue-600"><i class="fas fa-file-contract mr-2"></i>ContractArchive</div>
            <div class="space-x-4 flex items-center">
                <button id="theme-toggle" class="text-gray-600 dark:text-gray-300"><i class="fas fa-moon"></i></button>
                <a href="#" id="login-btn" class="text-gray-700 dark:text-gray-200 hover:text-blue-600">Kirish</a>
                <a href="#" id="register-btn" class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition">Ro‘yxatdan o‘tish</a>
            </div>
        </div>
    </nav>

    <!-- Hero Section -->
    <section class="pt-24 pb-12 px-4">
        <div class="container mx-auto text-center">
            <h1 class="text-4xl md:text-5xl font-bold text-gray-800 dark:text-white mb-4">Ijara shartnomalarini <span class="text-blue-600">aql bilan boshqaring</span></h1>
            <p class="text-lg text-gray-600 dark:text-gray-300 max-w-2xl mx-auto mb-8">Bulutda xavfsiz saqlash, tezkor qidiruv va elektron imzo bilan professional arxiv.</p>
            <div class="space-x-4">
                <button id="upload-btn" class="bg-blue-600 text-white px-6 py-3 rounded-xl shadow-lg hover:bg-blue-700 transition"><i class="fas fa-upload mr-2"></i>Shartnoma yuklash</button>
                <button id="archive-btn" class="bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-white px-6 py-3 rounded-xl shadow-lg hover:bg-gray-300 transition"><i class="fas fa-folder-open mr-2"></i>Arxivni ko‘rish</button>
            </div>
        </div>
    </section>

    <!-- Stats Section -->
    <section class="py-12 bg-white dark:bg-gray-900">
        <div class="container mx-auto px-4 grid md:grid-cols-3 gap-8">
            <div class="card bg-blue-50 dark:bg-gray-800 p-6 rounded-2xl shadow text-center">
                <i class="fas fa-file-alt text-4xl text-blue-600 mb-3"></i>
                <h3 class="text-3xl font-bold" id="total-contracts">0</h3>
                <p class="text-gray-600 dark:text-gray-300">Jami shartnomalar</p>
            </div>
            <div class="card bg-green-50 dark:bg-gray-800 p-6 rounded-2xl shadow text-center">
                <i class="fas fa-users text-4xl text-green-600 mb-3"></i>
                <h3 class="text-3xl font-bold" id="active-users">0</h3>
                <p class="text-gray-600 dark:text-gray-300">Faol foydalanuvchilar</p>
            </div>
            <div class="card bg-purple-50 dark:bg-gray-800 p-6 rounded-2xl shadow text-center">
                <i class="fas fa-cloud-upload-alt text-4xl text-purple-600 mb-3"></i>
                <h3 class="text-3xl font-bold" id="uploads-month">0</h3>
                <p class="text-gray-600 dark:text-gray-300">Bu oy yuklangan</p>
            </div>
        </div>
    </section>
    <script>
        // Dark mode logic
        const themeToggle = document.getElementById('theme-toggle');
        if(localStorage.getItem('theme') === 'dark') document.body.classList.add('dark-mode');
        themeToggle.addEventListener('click', () => {
            document.body.classList.toggle('dark-mode');
            localStorage.setItem('theme', document.body.classList.contains('dark-mode') ? 'dark' : 'light');
        });

        // Load stats from API (mock for now, will be replaced with real fetch)
        async function loadStats() {
            try {
                const res = await fetch('http://localhost:5000/api/contracts/stats');
                const data = await res.json();
                document.getElementById('total-contracts').innerText = data.total  0;
                document.getElementById('active-users').innerText = data.activeUsers  0;
                document.getElementById('uploads-month').innerText = data.uploadsMonth || 0;
            } catch(e) { console.log("Backend ulanishi mavjud emas, statistik ma'lumotlar test rejimida"); }
        }
        loadStats();

        document.getElementById('upload-btn').onclick = () => { if(localStorage.getItem('token')) window.location.href='dashboard.html'; else alert('Iltimos avval kiring'); };
        document.getElementById('archive-btn').onclick = () => window.location.href='dashboard.html';
    </script>
</body>
</html>

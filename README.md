mkdir golden-earnings
cd golden-earnings

mkdir -p client/src/components client/src/pages client/src/styles server/config server/middleware server/models server/routes

cat > .gitignore <<'EOF'
node_modules/
.env
.env.local
.env.*.local
dist/
build/
coverage/
.vscode/
.idea/
.DS_Store
*.log
EOF

cat > README.md <<'EOF'
# Golden Earnings

A full-stack task-based earning platform with registration, authentication, earning history, and withdrawal requests.

## Tech Stack
- React + Vite
- Express + Node.js
- MongoDB + Mongoose
- JWT + bcrypt

## Features
- User sign up and login
- Protected dashboard
- Task completion
- Earnings tracking
- Withdrawal requests

## Quick Start
### Backend
cd server
npm install
cp .env.example .env
npm run dev

### Frontend
cd client
npm install
cp .env.example .env
npm run dev

## Deployment
- Frontend: Vercel
- Backend: Render or Railway
- Database: MongoDB Atlas
EOF

mkdir -p server
cat > server/package.json <<'EOF'
{
  "name": "golden-earnings-server",
  "version": "1.0.0",
  "main": "server.js",
  "type": "module",
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.9.0",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^8.8.0",
    "nodemon": "^3.0.1"
  }
}
EOF

cat > server/.env.example <<'EOF'
PORT=5000
MONGO_URI=mongodb://localhost:27017/golden-earnings
JWT_SECRET=super_secret_change_this
CLIENT_URL=http://localhost:5173
EOF

cat > server/config/db.js <<'EOF'
import mongoose from 'mongoose';

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection failed:', error.message);
    process.exit(1);
  }
};

export default connectDB;
EOF

cat > server/models/User.js <<'EOF'
import mongoose from 'mongoose';
import bcrypt from 'bcryptjs';

const userSchema = new mongoose.Schema(
  {
    name: { type: String, required: true, trim: true },
    email: { type: String, required: true, unique: true, lowercase: true },
    password: { type: String, required: true, select: false },
    role: { type: String, default: 'user' },
    balance: { type: Number, default: 0 },
    totalEarned: { type: Number, default: 0 },
    totalWithdrawn: { type: Number, default: 0 },
    createdAt: { type: Date, default: Date.now }
  },
  { timestamps: true }
);

userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

userSchema.methods.matchPassword = async function (enteredPassword) {
  return bcrypt.compare(enteredPassword, this.password);
};

export default mongoose.model('User', userSchema);
EOF

cat > server/models/Task.js <<'EOF'
import mongoose from 'mongoose';

const taskSchema = new mongoose.Schema(
  {
    title: { type: String, required: true },
    description: { type: String, required: true },
    category: { type: String, enum: ['review', 'data-entry', 'ai-training', 'affiliate', 'other'], default: 'review' },
    reward: { type: Number, required: true, default: 5 },
    difficulty: { type: String, enum: ['easy', 'medium', 'hard'], default: 'easy' },
    estimatedTime: { type: Number, default: 15 },
    status: { type: String, enum: ['active', 'inactive'], default: 'active' },
    totalCompleted: { type: Number, default: 0 },
    completedBy: [{ userId: String, completedAt: Date }]
  },
  { timestamps: true }
);

export default mongoose.model('Task', taskSchema);
EOF

cat > server/models/Earnings.js <<'EOF'
import mongoose from 'mongoose';

const earningsSchema = new mongoose.Schema(
  {
    userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    taskId: { type: mongoose.Schema.Types.ObjectId, ref: 'Task', required: true },
    amount: { type: Number, required: true },
    source: { type: String, default: 'task' },
    description: { type: String, default: '' }
  },
  { timestamps: true }
);

export default mongoose.model('Earnings', earningsSchema);
EOF

cat > server/models/Withdrawal.js <<'EOF'
import mongoose from 'mongoose';

const withdrawalSchema = new mongoose.Schema(
  {
    userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    amount: { type: Number, required: true },
    paymentMethod: { type: String, default: 'bank_transfer' },
    accountDetails: { type: String, required: true },
    status: { type: String, enum: ['pending', 'processing', 'completed', 'rejected'], default: 'pending' }
  },
  { timestamps: true }
);

export default mongoose.model('Withdrawal', withdrawalSchema);
EOF

cat > server/middleware/authMiddleware.js <<'EOF'
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

export const authMiddleware = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ message: 'No token provided' });

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) return res.status(401).json({ message: 'User not found' });

    req.user = user;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid or expired token' });
  }
};
EOF

cat > server/routes/authRoutes.js <<'EOF'
import express from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const router = express.Router();

const generateToken = (userId) => {
  return jwt.sign({ id: userId }, process.env.JWT_SECRET, { expiresIn: '7d' });
};

router.post('/register', async (req, res) => {
  const { name, email, password, passwordConfirm } = req.body;

  if (!name || !email || !password || !passwordConfirm) {
    return res.status(400).json({ message: 'All fields are required' });
  }

  if (password !== passwordConfirm) {
    return res.status(400).json({ message: 'Passwords do not match' });
  }

  const existing = await User.findOne({ email });
  if (existing) {
    return res.status(400).json({ message: 'User already exists' });
  }

  const user = await User.create({ name, email, password });
  const token = generateToken(user._id);

  res.status(201).json({
    token,
    user: {
      id: user._id,
      name: user.name,
      email: user.email,
      role: user.role,
      balance: user.balance
    }
  });
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;

  if (!email || !password) {
    return res.status(400).json({ message: 'Email and password are required' });
  }

  const user = await User.findOne({ email }).select('+password');

  if (!user || !(await user.matchPassword(password))) {
    return res.status(401).json({ message: 'Invalid credentials' });
  }

  const token = generateToken(user._id);
  res.status(200).json({
    token,
    user: {
      id: user._id,
      name: user.name,
      email: user.email,
      role: user.role,
      balance: user.balance
    }
  });
});

router.get('/me', async (req, res) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ message: 'Unauthorized' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    if (!user) return res.status(401).json({ message: 'Unauthorized' });
    res.status(200).json({ user });
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
});

export default router;
EOF

cat > server/routes/taskRoutes.js <<'EOF'
import express from 'express';
import Task from '../models/Task.js';
import User from '../models/User.js';
import Earnings from '../models/Earnings.js';
import { authMiddleware } from '../middleware/authMiddleware.js';

const router = express.Router();

router.get('/', async (req, res) => {
  try {
    const tasks = await Task.find({ status: 'active' }).sort({ reward: -1 });
    res.status(200).json({ tasks });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

router.post('/:id/complete', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    if (!task) return res.status(404).json({ message: 'Task not found' });

    const alreadyCompleted = task.completedBy.some((c) => c.userId === req.user._id.toString());
    if (alreadyCompleted) {
      return res.status(400).json({ message: 'You already completed this task' });
    }

    task.completedBy.push({ userId: req.user._id.toString(), completedAt: new Date() });
    task.totalCompleted += 1;
    await task.save();

    const earning = await Earnings.create({
      userId: req.user._id,
      taskId: task._id,
      amount: task.reward,
      source: 'task',
      description: task.title
    });

    const user = await User.findById(req.user._id);
    user.balance += task.reward;
    user.totalEarned += task.reward;
    await user.save();

    res.status(200).json({
      message: 'Task completed successfully',
      earning
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, category, reward, difficulty, estimatedTime } = req.body;
    const task = await Task.create({
      title,
      description,
      category,
      reward,
      difficulty,
      estimatedTime
    });
    res.status(201).json({ task });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

export default router;
EOF

cat > server/routes/userRoutes.js <<'EOF'
import express from 'express';
import User from '../models/User.js';
import Withdrawal from '../models/Withdrawal.js';
import Earnings from '../models/Earnings.js';
import { authMiddleware } from '../middleware/authMiddleware.js';

const router = express.Router();

router.get('/profile', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.user._id).select('-password');
    res.status(200).json({ user });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

router.get('/earnings', authMiddleware, async (req, res) => {
  try {
    const earnings = await Earnings.find({ userId: req.user._id }).populate('taskId', 'title').sort({ createdAt: -1 });
    const totalEarnings = earnings.reduce((sum, item) => sum + item.amount, 0);
    res.status(200).json({ earnings, totalEarnings });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

router.post('/withdrawal', authMiddleware, async (req, res) => {
  try {
    const { amount, paymentMethod, accountDetails } = req.body;

    if (!amount || amount < 10) {
      return res.status(400).json({ message: 'Minimum withdrawal is $10' });
    }

    const user = await User.findById(req.user._id);
    if (user.balance < amount) {
      return res.status(400).json({ message: 'Insufficient balance' });
    }

    const withdrawal = await Withdrawal.create({
      userId: user._id,
      amount,
      paymentMethod,
      accountDetails
    });

    res.status(201).json({
      message: 'Withdrawal request submitted',
      withdrawal
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
});

export default router;
EOF

cat > server/app.js <<'EOF'
import express from 'express';
import cors from 'cors';
import authRoutes from './routes/authRoutes.js';
import taskRoutes from './routes/taskRoutes.js';
import userRoutes from './routes/userRoutes.js';

const app = express();

app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:5173',
  credentials: true
}));

app.use(express.json());

app.get('/api/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

app.use('/api/auth', authRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/users', userRoutes);

export default app;
EOF

cat > server/server.js <<'EOF'
import dotenv from 'dotenv';
import app from './app.js';
import connectDB from './config/db.js';

dotenv.config();

connectDB();

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
EOF

mkdir -p client
cat > client/package.json <<'EOF'
{
  "name": "golden-earnings-client",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.9.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.28.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "vite": "^5.4.10"
  }
}
EOF

cat > client/.env.example <<'EOF'
VITE_API_URL=http://localhost:5000
EOF

cat > client/vite.config.js <<'EOF'
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173
  }
});
EOF

cat > client/index.html <<'EOF'
<!doctype html>
<html lang=\"en\">
  <head>
    <meta charset=\"UTF-8\" />
    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\" />
    <title>Golden Earnings</title>
  </head>
  <body>
    <div id=\"root\"></div>
    <script type=\"module\" src=\"/src/main.jsx\"></script>
  </body>
</html>
EOF

cat > client/src/main.jsx <<'EOF'
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App.jsx';
import './styles.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
);
EOF

cat > client/src/App.jsx <<'EOF'
import { useState, useEffect } from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import Navbar from './components/Navbar.jsx';
import HomePage from './pages/HomePage.jsx';
import RegisterPage from './pages/RegisterPage.jsx';
import LoginPage from './pages/LoginPage.jsx';
import DashboardPage from './pages/DashboardPage.jsx';
import TasksPage from './pages/TasksPage.jsx';
import WithdrawPage from './pages/WithdrawPage.jsx';
import ProfilePage from './pages/ProfilePage.jsx';

function App() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [user, setUser] = useState(null);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      fetchUser(token);
    }
  }, []);

  const fetchUser = async (token) => {
    try {
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/auth/me`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      if (response.ok) {
        const data = await response.json();
        setUser(data.user);
        setIsAuthenticated(true);
      } else {
        localStorage.removeItem('token');
      }
    } catch (error) {
      console.error('Failed to fetch user');
    }
  };

  const handleLogout = () => {
    localStorage.removeItem('token');
    setIsAuthenticated(false);
    setUser(null);
  };

  return (
    <div className=\"app-shell\">
      <Navbar isAuthenticated={isAuthenticated} user={user} onLogout={handleLogout} />
      <Routes>
        <Route path=\"/\" element={<HomePage />} />
        <Route path=\"/register\" element={isAuthenticated ? <Navigate to=\"/dashboard\" replace /> : <RegisterPage setIsAuthenticated={setIsAuthenticated} setUser={setUser} />} />
        <Route path=\"/login\" element={isAuthenticated ? <Navigate to=\"/dashboard\" replace /> : <LoginPage setIsAuthenticated={setIsAuthenticated} setUser={setUser} />} />
        <Route path=\"/dashboard\" element={isAuthenticated ? <DashboardPage user={user} /> : <Navigate to=\"/login\" replace />} />
        <Route path=\"/tasks\" element={isAuthenticated ? <TasksPage user={user} /> : <Navigate to=\"/login\" replace />} />
        <Route path=\"/withdraw\" element={isAuthenticated ? <WithdrawPage user={user} /> : <Navigate to=\"/login\" replace />} />
        <Route path=\"/profile\" element={isAuthenticated ? <ProfilePage user={user} setUser={setUser} /> : <Navigate to=\"/login\" replace />} />
      </Routes>
    </div>
  );
}

export default App;
EOF

mkdir -p client/src/components client/src/pages client/src/styles

cat > client/src/components/Navbar.jsx <<'EOF'
import { Link } from 'react-router-dom';

function Navbar({ isAuthenticated, user, onLogout }) {
  return (
    <nav className=\"navbar\">
      <div className=\"nav-container\">
        <Link to=\"/\" className=\"nav-logo\">Golden Earnings</Link>
        <div className=\"nav-links\">
          <Link to=\"/\" className=\"nav-link\">Home</Link>
          {isAuthenticated ? (
            <>
              <Link to=\"/tasks\" className=\"nav-link\">Tasks</Link>
              <Link to=\"/dashboard\" className=\"nav-link\">Dashboard</Link>
              <Link to=\"/profile\" className=\"nav-link\">Profile</Link>
              <span className=\"nav-user\">{user?.name || 'User'} | ${user?.balance || '0.00'}</span>
              <button className=\"nav-button\" onClick={onLogout}>Logout</button>
            </>
          ) : (
            <>
              <Link to=\"/login\" className=\"nav-link\">Login</Link>
              <Link to=\"/register\" className=\"nav-link register-btn\">Sign Up</Link>
            </>
          )}
        </div>
      </div>
    </nav>
  );
}

export default Navbar;
EOF

cat > client/src/pages/HomePage.jsx <<'EOF'
import { Link } from 'react-router-dom';

function HomePage() {
  return (
    <div className=\"home-page\">
      <section className=\"hero\">
        <h1>Make Money Online with Golden Earnings</h1>
        <p>Complete simple tasks and earn real money online.</p>
        <Link to=\"/register\" className=\"hero-button\">Get Started</Link>
      </section>

      <section className=\"features\">
        <div className=\"feature-card\">
          <h3>Simple Tasks</h3>
          <p>Earn as you complete easy online tasks.</p>
        </div>
        <div className=\"feature-card\">
          <h3>Instant Earnings</h3>
          <p>Track your balance in real time.</p>
        </div>
        <div className=\"feature-card\">
          <h3>Fast Withdrawals</h3>
          <p>Withdraw your earnings securely.</p>
        </div>
      </section>
    </div>
  );
}

export default HomePage;
EOF

cat > client/src/pages/RegisterPage.jsx <<'EOF'
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';

function RegisterPage({ setIsAuthenticated, setUser }) {
  const [form, setForm] = useState({
    name: '', email: '', password: '', passwordConfirm: ''
  });
  const [error, setError] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');

    try {
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/auth/register`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(form)
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.message || 'Registration failed');
      }

      localStorage.setItem('token', data.token);
      setIsAuthenticated(true);
      setUser(data.user);
      navigate('/dashboard');
    } catch (err) {
      setError(err.message || 'Registration failed');
    }
  };

  const navigate = useNavigate();

  return (
    <div className=\"auth-page\">
      <h2>Create Account</h2>
      {error && <p className=\"error-message\">{error}</p>}

      <form onSubmit={handleSubmit}>
        <input type=\"text\" name=\"name\" placeholder=\"Full Name\" value={form.name} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <input type=\"email\" name=\"email\" placeholder=\"Email\" value={form.email} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <input type=\"password\" name=\"password\" placeholder=\"Password\" value={form.password} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <input type=\"password\" name=\"passwordConfirm\" placeholder=\"Confirm Password\" value={form.passwordConfirm} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <button type=\"submit\">Create Account</button>
      </form>
    </div>
  );
}

export default RegisterPage;
EOF

cat > client/src/pages/LoginPage.jsx <<'EOF'
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';

function LoginPage({ setIsAuthenticated, setUser }) {
  const [form, setForm] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');

    try {
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/auth/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(form)
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.message || 'Login failed');
      }

      localStorage.setItem('token', data.token);
      setIsAuthenticated(true);
      setUser(data.user);
      navigate('/dashboard');
    } catch (err) {
      setError(err.message || 'Login failed');
    }
  };

  const navigate = useNavigate();

  return (
    <div className=\"auth-page\">
      <h2>Login</h2>
      {error && <p className=\"error-message\">{error}</p>}

      <form onSubmit={handleSubmit}>
        <input type=\"email\" name=\"email\" placeholder=\"Email\" value={form.email} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <input type=\"password\" name=\"password\" placeholder=\"Password\" value={form.password} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <button type=\"submit\">Login</button>
      </form>
    </div>
  );
}

export default LoginPage;
EOF

cat > client/src/pages/DashboardPage.jsx <<'EOF'
import { useEffect, useState } from 'react';

function DashboardPage({ user }) {
  const [stats, setStats] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchStats();
  }, []);

  const fetchStats = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/users/earnings`, {
        headers: { Authorization: `Bearer ${token}` }
      });

      const data = await response.json();

      if (response.ok) {
        setStats(data);
      } else {
        console.error(data.message || 'Failed to fetch stats');
      }
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className=\"dashboard-page\">
      <h1>Welcome, {user?.name || 'User'}</h1>
      <div className=\"stats-grid\">
        <div className=\"stat-card\">
          <h3>Balance</h3>
          <p>${user?.balance || '0.00'}</p>
        </div>
        <div className=\"stat-card\">
          <h3>Total Earned</h3>
          <p>${stats?.totalEarnings || '0.00'}</p>
        </div>
      </div>

      {loading ? <p>Loading...</p> : (
        <div className=\"stats-list\">
          <h3>Recent Earnings</h3>
          {stats?.earnings?.length ? stats.earnings.slice(0, 5).map((earning) => (
            <li key={earning._id}>
              {earning.description || earning.taskId?.title || 'Task'}: ${earning.amount}
            </li>
          )) : <p>No earnings yet.</p>}
        </div>
      )}
    </div>
  );
}

export default DashboardPage;
EOF

cat > client/src/pages/TasksPage.jsx <<'EOF'
import { useEffect, useState } from 'react';

function TasksPage({ user }) {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/tasks`);
      const data = await response.json();

      if (response.ok) {
        setTasks(data.tasks || []);
      } else {
        console.error(data.message || 'Failed to fetch tasks');
      }
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  };

  const completeTask = async (taskId) => {
    try {
      const token = localStorage.getItem('token');
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/tasks/${taskId}/complete`, {
        method: 'POST',
        headers: { Authorization: `Bearer ${token}` }
      });

      const data = await response.json();

      if (response.ok) {
        alert('Task completed!');
        fetchTasks();
      } else {
        alert(data.message || 'Task failed');
      }
    } catch (error) {
      alert('An error occurred');
    }
  };

  return (
    <div className=\"tasks-page\">
      <h1>Available Tasks</h1>
      {loading ? <p>Loading...</p> : (
        tasks.map((task) => (
          <div key={task._id} className=\"task-card\">
            <h3>{task.title}</h3>
            <p>{task.description}</p>
            <p className=\"reward\">Reward: ${task.reward}</p>
            <button onClick={() => completeTask(task._id)}>Complete Task</button>
          </div>
        ))
      )}
    </div>
  );
}

export default TasksPage;
EOF

cat > client/src/pages/WithdrawPage.jsx <<'EOF'
import { useState } from 'react';

function WithdrawPage({ user }) {
  const [form, setForm] = useState({ amount: '', paymentMethod: 'bank_transfer', accountDetails: '' });
  const [message, setMessage] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();

    try {
      const token = localStorage.getItem('token');
      const response = await fetch(`${import.meta.env.VITE_API_URL}/api/users/withdrawal`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
        body: JSON.stringify(form)
      });

      const data = await response.json();

      if (!response.ok) throw new Error(data.message || 'Withdrawal failed');

      setMessage('Withdrawal request submitted successfully!');
      setForm({ amount: '', paymentMethod: 'bank_transfer', accountDetails: '' });
    } catch (err) {
      setMessage(err.message || 'Withdrawal failed');
    }
  };

  return (
    <div className=\"withdraw-page\">
      <h1>Withdraw Earnings</h1>
      <p>Available balance: ${user?.balance || '0.00'}</p>
      <form onSubmit={handleSubmit}>
        <input type=\"number\" name=\"amount\" placeholder=\"Minimum $10\" min=\"10\" value={form.amount} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <select name=\"paymentMethod\" value={form.paymentMethod} onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })}>
          <option value=\"bank_transfer\">Bank Transfer</option>
          <option value=\"paypal\">PayPal</option>
          <option value=\"stripe\">Stripe</option>
        </select>
        <textarea name=\"accountDetails\" value={form.accountDetails} placeholder=\"Enter account details\" onChange={(e) => setForm({ ...form, [e.target.name]: e.target.value })} />
        <button type=\"submit\">Submit Withdrawal</button>
      </form>
      {message && <p>{message}</p>}
    </div>
  );
}

export default WithdrawPage;
EOF

cat > client/src/pages/ProfilePage.jsx <<'EOF'
function ProfilePage({ user, setUser }) {
  return (
    <div className=\"profile-page\">
      <h1>Profile</h1>
      <p>Name: {user?.name || 'N/A'}</p>
      <p>Email: {user?.email || 'N/A'}</p>
      <p>Balance: ${user?.balance || '0.00'}</p>
    </div>
  );
}

export default ProfilePage;
EOF

cat > client/src/styles.css <<'EOF'
body {
  font-family: Arial, sans-serif;
  margin: 0;
  background: #f5f7fb;
}

.navbar {
  background: #1a73e8;
  color: white;
  padding: 18px 20px;
}

.nav-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  max-width: 1200px;
  margin: 0 auto;
}

.nav-logo {
  color: white;
  text-decoration: none;
  font-weight: 700;
  font-size: 1.5rem;
}

.nav-links {
  display: flex;
  align-items: center;
  gap: 15px;
}

.nav-link {
  color: white;
  text-decoration: none;
}

.register-btn {
  background: white;
  color: #1a73e8;
  padding: 8px 12px;
  border-radius: 6px;
}

.nav-button {
  background: #e53935;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 8px 12px;
  cursor: pointer;
}

.hero {
  background: linear-gradient(90deg, #1a73e8, #0a4bb0);
  color: white;
  min-height: 360px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  padding: 30px;
}

.hero h1 {
  font-size: 2.7rem;
  margin-bottom: 20px;
}

.hero-button {
  display: inline-block;
  background: white;
  color: #1a73e8;
  padding: 14px 24px;
  border-radius: 8px;
  text-decoration: none;
  font-weight: 700;
}

.features {
  max-width: 1200px;
  margin: 40px auto;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 20px;
  padding: 0 20px;
}

.feature-card {
  background: white;
  padding: 24px;
  border-radius: 10px;
  box-shadow: 0 6px 18px rgba(0,0,0,0.08);
}

.auth-page {
  max-width: 420px;
  margin: 60px auto;
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 8px 18px rgba(0,0,0,0.08);
}

.auth-page h2 {
  margin-bottom: 20px;
}

.auth-page form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.auth-page input, .auth-page textarea, .auth-page select {
  width: 100%;
  padding: 12px;
  border: 1px solid #dfe3e8;
  border-radius: 8px;
  box-sizing: border-box;
}

.auth-page button, .tasks-page button, .dashboard-page button, .withdraw-page button {
  background: #1a73e8;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 12px 16px;
  cursor: pointer;
}

.tasks-page {
  max-width: 1200px;
  margin: 40px auto;
  padding: 0 20px;
}

.task-card {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 8px 18px rgba(0,0,0,0.08);
  margin-bottom: 20px;
}

.reward {
  font-weight: 700;
  color: #1a73e8;
}

.dashboard-page {
  max-width: 1200px;
  margin: 40px auto;
  padding: 0 20px;
}

.stats-grid {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
}

.stat-card {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 8px 18px rgba(0,0,0,0.08);
  min-width: 220px;
}

.withdraw-page {
  max-width: 500px;
  margin: 40px auto;
  padding: 0 20px;
}

.auth-page button:hover, .tasks-page button:hover, .dashboard-page button:hover, .withdraw-page button:hover {
  background: #0f5ac2;
}
EOF

cat > render.yaml <<'EOF'
services:
  - type: web
    name: golden-earnings-api
    env: node
    plan: free
    buildCommand: cd server && npm install
    startCommand: cd server && npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: MONGO_URI
        value: mongodb://localhost:27017/golden-earnings
      - key: JWT_SECRET
        value: super_secret_change_this
      - key: CLIENT_URL
        value: https://your-frontend-url.vercel.app
EOF

cat > client/vercel.json <<'EOF'
{
  "version": 2,
  "builds": [
    { "src": "package.json", "use": "@vercel/static-build", "config": { "distDir": "dist" } }
  ],
  "routes": [
    { "src": "/(.*)", "dest": "/index.html" }
  ]
}
EOF

cat > server/vercel.json <<'EOF'
{
  "version": 2,
  "builds": [
    { "src": "server.js", "use": "@vercel/node" }
  ],
  "routes": [
    { "src": "/(.*)", "dest": "server.js" }
  ]
}
EOF

npm install --prefix server
npm install --prefix client

git init
git add .
git commit -m "Initial MVP setup"
git branch -M main

echo "Local project is ready. Now run:"
echo "git remote add origin https://github.com/<your-username>/golden-earnings.git"
echo "git push -u origin main"

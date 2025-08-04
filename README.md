// frontend 
npm create vite@latest civic-issue-tracker -- --template react
cd civic-issue-tracker
npm install
npm install react-router-dom axios leaflet react-leaflet i18next react-i18next tailwindcss postcss autoprefixer
npx tailwindcss init -p

content: ["./index.html", "./src/**/*.{js,jsx}"],

import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

const resources = {
  en: { translation: { welcome: "Welcome", reportIssue: "Report Issue" } },
  hi: { translation: { welcome: "स्वागत है", reportIssue: "समस्या दर्ज करें" } },
};

i18n.use(initReactI18next).init({
  resources,
  lng: 'en',
  fallbackLng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;

import { useTranslation } from 'react-i18next';

export default function LanguageToggle() {
  const { i18n } = useTranslation();
  return (
    <button
      onClick={() => i18n.changeLanguage(i18n.language === 'en' ? 'hi' : 'en')}
      className="p-2 text-sm bg-gray-200 rounded"
    >
      {i18n.language === 'en' ? 'हिंदी' : 'English'}
    </button>
  );
}

import './i18n';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import HomePage from './pages/HomePage';
import LoginPage from './pages/LoginPage';
import RegisterPage from './pages/RegisterPage';
import ReportIssuePage from './pages/ReportIssuePage';
import Header from './components/Header';

export default function App() {
  return (
    <BrowserRouter>
      <Header />
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/login" element={<LoginPage />} />
        <Route path="/register" element={<RegisterPage />} />
        <Route path="/report" element={<ReportIssuePage />} />
      </Routes>
    </BrowserRouter>
  );
}

import { useTranslation } from 'react-i18next';

export default function HomePage() {
  const { t } = useTranslation();
  return (
    <div className="p-4">
      <h1 className="text-xl font-bold">{t('welcome')}</h1>
      <div className="mt-4">[Map will be here]</div>
      <div className="mt-4">[Filter buttons]</div>
    </div>
  );
}

import { useState } from 'react';

export default function ReportIssuePage() {
  const [title, setTitle] = useState('');
  const [desc, setDesc] = useState('');
  const [photos, setPhotos] = useState([]);

  return (
    <div className="p-4">
      <h2 className="text-lg font-bold">Report Issue</h2>
      <input placeholder="Title" value={title} onChange={e => setTitle(e.target.value)} className="border p-2 block mt-2 w-full" />
      <textarea placeholder="Description" value={desc} onChange={e => setDesc(e.target.value)} className="border p-2 block mt-2 w-full" />
      <input type="file" multiple onChange={e => setPhotos([...e.target.files])} className="block mt-2" />
      <button className="mt-4 p-2 bg-blue-500 text-white rounded">Submit</button>
    </div>
  );
}

import { Link } from 'react-router-dom';
import LanguageToggle from './LanguageToggle';

export default function Header() {
  return (
    <header className="flex items-center justify-between p-4 bg-gray-100">
      <Link to="/" className="font-bold">Civic Issue Tracker</Link>
      <nav className="flex gap-4">
        <Link to="/report">Report</Link>
        <Link to="/login">Login</Link>
        <LanguageToggle />
      </nav>
    </header>
  );
}

// Backend
npm install express mongoose dotenv bcryptjs jsonwebtoken cors multer cloudinary

const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
.then(() => console.log('MongoDB connected'))
.catch(err => console.error(err));

app.use('/api/auth', require('./routes/auth'));
app.use('/api/issue', require('./routes/issue'));

app.listen(5000, () => console.log('Server running on port 5000'));

const mongoose = require('mongoose');
const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  isBanned: { type: Boolean, default: false },
});
module.exports = mongoose.model('User', userSchema);

const mongoose = require('mongoose');
const issueSchema = new mongoose.Schema({
  title: String,
  description: String,
  category: String,
  photos: [String],
  status: { type: String, default: 'Reported' },
  flaggedCount: { type: Number, default: 0 },
  location: {
    type: { type: String, default: 'Point' },
    coordinates: [Number], // [lng, lat]
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
}, { timestamps: true });

issueSchema.index({ location: '2dsphere' });
module.exports = mongoose.model('Issue', issueSchema);

const router = require('express').Router();
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

router.post('/register', async (req, res) => {
  const hashed = await bcrypt.hash(req.body.password, 10);
  try {
    const user = await User.create({ username: req.body.username, password: hashed });
    res.json({ msg: 'User created' });
  } catch {
    res.status(400).json({ msg: 'User exists' });
  }
});

router.post('/login', async (req, res) => {
  const user = await User.findOne({ username: req.body.username });
  if (!user || !(await bcrypt.compare(req.body.password, user.password)))
    return res.status(400).json({ msg: 'Invalid credentials' });
  if (user.isBanned) return res.status(403).json({ msg: 'Banned' });
  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
  res.json({ token });
});

module.exports = router;

const router = require('express').Router();
const Issue = require('../models/Issue');
const jwt = require('jsonwebtoken');

// auth middleware
function auth(req, res, next) {
  const token = req.headers.authorization;
  if (!token) return res.sendStatus(401);
  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.sendStatus(403);
    req.userId = decoded.id;
    next();
  });
}

// Report new issue
router.post('/report', auth, async (req, res) => {
  const { title, description, category, photos, lat, lng } = req.body;
  const issue = await Issue.create({
    title, description, category, photos,
    location: { coordinates: [lng, lat] },
    createdBy: req.userId,
  });
  res.json(issue);
});

// Get nearby issues (within radius km)
router.get('/nearby', auth, async (req, res) => {
  const { lat, lng, radius } = req.query;
  const issues = await Issue.find({
    location: {
      $geoWithin: { $centerSphere: [[lng, lat], radius/6378.1] }
    }
  });
  res.json(issues);
});

// Flag issue
router.post('/flag/:id', auth, async (req, res) => {
  const issue = await Issue.findByIdAndUpdate(req.params.id, { $inc: { flaggedCount: 1 } }, { new: true });
  res.json(issue);
});

module.exports = router;

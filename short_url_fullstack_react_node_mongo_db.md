# ShortURL — Fullstack URL Shortener (React + Node + MongoDB)

This repository contains a complete fullstack URL shortener named **ShortURL**.

---

## Project structure (single repo)

```
shorturl/
├─ backend/
│  ├─ package.json
│  ├─ server.js
│  ├─ .env.example
│  ├─ models/
│  │  └─ Url.js
│  └─ routes/
│     └─ urls.js
├─ frontend/
│  ├─ package.json
│  ├─ public/
│  │  └─ index.html
│  └─ src/
│     ├─ index.js
│     ├─ App.js
│     ├─ api.js
│     └─ components/
│        ├─ ShortenForm.js
│        └─ LinkList.js
└─ README.md
```

---

> **Note:** The full source code for each file is included below. Copy into respective files.

---

## Backend

### `backend/package.json`

```json
{
  "name": "shorturl-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "mongoose": "^7.0.0",
    "nanoid": "^4.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
```

### `backend/.env.example`

```
PORT=5000
MONGO_URI=mongodb://localhost:27017/shorturl
BASE_URL=http://localhost:5000
```

### `backend/server.js`

```js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const dotenv = require('dotenv');
const urlsRouter = require('./routes/urls');

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

// connect to MongoDB
const MONGO_URI = process.env.MONGO_URI || 'mongodb://localhost:27017/shorturl';
mongoose.connect(MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

// routes
app.use('/api/urls', urlsRouter);

// redirect route for short codes
const Url = require('./models/Url');
app.get('/:code', async (req, res) => {
  try {
    const code = req.params.code;
    const url = await Url.findOne({ code });
    if (url) {
      url.clicks = (url.clicks || 0) + 1;
      await url.save();
      return res.redirect(url.longUrl);
    }
    return res.status(404).json({ message: 'Not found' });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### `backend/models/Url.js`

```js
const mongoose = require('mongoose');

const UrlSchema = new mongoose.Schema({
  code: { type: String, required: true, unique: true },
  shortUrl: { type: String, required: true },
  longUrl: { type: String, required: true },
  clicks: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Url', UrlSchema);
```

### `backend/routes/urls.js`

```js
const express = require('express');
const router = express.Router();
const { customAlphabet } = require('nanoid');
const Url = require('../models/Url');

const nanoid = customAlphabet('0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ', 6);

// Create a shortened URL
router.post('/', async (req, res) => {
  try {
    const { longUrl } = req.body;
    if (!longUrl) return res.status(400).json({ message: 'longUrl is required' });

    // check if already exists
    const existing = await Url.findOne({ longUrl });
    if (existing) return res.json(existing);

    const code = nanoid();
    const base = process.env.BASE_URL || 'http://localhost:5000';
    const shortUrl = `${base}/${code}`;

    const url = new Url({ code, shortUrl, longUrl });
    await url.save();

    res.status(201).json(url);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// List all shortened URLs (for admin/overview)
router.get('/', async (req, res) => {
  try {
    const urls = await Url.find().sort({ createdAt: -1 });
    res.json(urls);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

module.exports = router;
```

---

## Frontend (React)

This is a minimal React app built with standard structure. You can use `create-react-app` or Vite — below assumes CRA for simplicity.

### `frontend/package.json`

```json
{
  "name": "shorturl-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}
```

### `frontend/public/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>ShortURL</title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

### `frontend/src/index.js`

```js
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
const root = createRoot(container);
root.render(<App />);
```

### `frontend/src/api.js`

```js
import axios from 'axios';

const API = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000/api'
});

export default API;
```

### `frontend/src/App.js`

```js
import React, { useEffect, useState } from 'react';
import ShortenForm from './components/ShortenForm';
import LinkList from './components/LinkList';
import API from './api';

export default function App() {
  const [links, setLinks] = useState([]);

  const fetchLinks = async () => {
    try {
      const res = await API.get('/urls');
      setLinks(res.data);
    } catch (err) {
      console.error(err);
    }
  };

  useEffect(() => { fetchLinks(); }, []);

  return (
    <div style={{ maxWidth: 700, margin: '40px auto', fontFamily: 'Arial, sans-serif' }}>
      <h1>ShortURL</h1>
      <ShortenForm onCreate={fetchLinks} />
      <hr />
      <LinkList links={links} />
    </div>
  );
}
```

### `frontend/src/components/ShortenForm.js`

```js
import React, { useState } from 'react';
import API from '../api';

export default function ShortenForm({ onCreate }) {
  const [url, setUrl] = useState('');
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!url) return;
    setLoading(true);
    try {
      const res = await API.post('/urls', { longUrl: url });
      setResult(res.data);
      setUrl('');
      if (onCreate) onCreate();
    } catch (err) {
      console.error(err);
      alert('Error, check console');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit} style={{ display: 'flex', gap: 8 }}>
        <input
          type="url"
          placeholder="Enter a long URL (https://...)"
          value={url}
          onChange={(e) => setUrl(e.target.value)}
          style={{ flex: 1, padding: '8px' }}
          required
        />
        <button type="submit" disabled={loading} style={{ padding: '8px 12px' }}>
          {loading ? 'Shortening...' : 'Shorten'}
        </button>
      </form>

      {result && (
        <div style={{ marginTop: 12 }}>
          <div>Short URL:</div>
          <a href={result.shortUrl} target="_blank" rel="noopener noreferrer">{result.shortUrl}</a>
          <div style={{ fontSize: 12, color: '#555' }}>Clicks: {result.clicks || 0}</div>
        </div>
      )}
    </div>
  );
}
```

### `frontend/src/components/LinkList.js`

```js
import React from 'react';

export default function LinkList({ links }) {
  if (!links || links.length === 0) return <div>No links yet.</div>;

  return (
    <div>
      <h3>Recent</h3>
      <table style={{ width: '100%', borderCollapse: 'collapse' }}>
        <thead>
          <tr>
            <th style={{ textAlign: 'left', padding: 6 }}>Short</th>
            <th style={{ textAlign: 'left', padding: 6 }}>Original</th>
            <th style={{ textAlign: 'left', padding: 6 }}>Clicks</th>
          </tr>
        </thead>
        <tbody>
          {links.map(l => (
            <tr key={l._id}>
              <td style={{ padding: 6 }}><a href={l.shortUrl} target="_blank" rel="noreferrer">{l.shortUrl}</a></td>
              <td style={{ padding: 6 }}><a href={l.longUrl} target="_blank" rel="noreferrer">{l.longUrl}</a></td>
              <td style={{ padding: 6 }}>{l.clicks}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

## README (root)

```md
# ShortURL

Simple fullstack URL shortener built with React (frontend), Node/Express (backend) and MongoDB (database).

## Setup

1. Clone the repo.
2. Start MongoDB (local or Atlas) and copy connection string into `backend/.env` as `MONGO_URI`.
3. Configure `BASE_URL` in `.env` (e.g. http://localhost:5000).

### Backend

```bash
cd backend
cp .env.example .env
# edit .env to set MONGO_URI and BASE_URL
npm install
npm run dev   # requires nodemon
```

### Frontend

```bash
cd frontend
npm install
# optionally set REACT_APP_API_URL to the backend API base url
npm start
```

Open the frontend (usually http://localhost:3000). Shortening uses the backend endpoint at `/api/urls`.

## Notes
- You can add basic validation, rate limiting, user accounts, analytics dashboard, or custom slugs.
- For production, consider hosting the frontend as static and point server `BASE_URL` to your domain.
```

---

If you want, I can also:
- Add authentication to let users manage their links.
- Add a custom-slug feature (user chooses the code).
- Add analytics (country, referrer) using `req.headers` and a small logging schema.




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
npm run dev # requires nodemon
```

### Frontend

```bash
cd frontend
npm install
# optionally set REACT_APP_API_URL to the backend API base url
npm start
```
Shortening uses the backend endpoint at /api/urls.

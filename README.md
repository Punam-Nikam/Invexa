# INVEXA

**OCR-Based Invoice Digitization and Tally Integration Platform for Chartered Accountants**

A multi-tenant SaaS that lets shop owners upload bills, extracts data using OCR, lets a CA review and approve, and generates Tally Prime-compatible XML vouchers.

---

## What It Does

- Shop Owners upload bills (PDF/JPG/PNG) from a web dashboard.
- OCR extracts invoice number, date, vendor, GSTIN, and amounts.
- CAs review the extracted fields, edit if needed, and approve.
- Backend generates a Tally-importable XML file with ledger masters + voucher.
- CA imports the XML into the client's Tally company in one step.
- Clients chat with their CA and get notifications.

Every firm is fully isolated by `firm_id` — no firm sees another firm's data.

---

## Tech Stack

- Frontend: React 18 (Vite), React Router, Axios
- Backend: Python 3.12, Flask, PyMongo
- Database: MongoDB Atlas
- Storage: GridFS
- Auth: JWT + bcrypt
- OCR: Tesseract + pdfplumber + Poppler

---

## Prerequisites

- Python 3.12+
- Node.js 18+
- MongoDB Atlas account
- Tesseract-OCR — https://github.com/UB-Mannheim/tesseract/wiki
- Poppler — https://github.com/oschwartz10612/poppler-windows/releases

---

## Setup

### Backend

Create `backend/.env`:

```
MONGO_URI=mongodb+srv://<user>:<pass>@<cluster>/invexa
DATABASE_NAME=invexa
SECRET_KEY=<64-char hex>
FRONTEND_ORIGIN=http://localhost:5173
TESSERACT_CMD=C:\Program Files\Tesseract-OCR\tesseract.exe
POPPLER_PATH=C:\poppler\Library\bin
```

Generate a secret:

```
python -c "import secrets; print(secrets.token_hex(32))"
```

Run:

```
cd backend
python -m venv venv
venv\Scripts\activate            (Windows)
source venv/bin/activate         (Mac/Linux)
pip install -r requirements.txt
python app.py
```

### Frontend

Create `frontend/.env`:

```
VITE_API_URL=http://127.0.0.1:5000/api
```

Run in a second terminal:

```
cd frontend
npm install
npm run dev
```

Open http://localhost:5173

---

## Workflow

```
Client uploads bill
      ↓
Backend saves to GridFS + creates invoice (status: uploaded)
      ↓
CA clicks "Extract OCR" → Tesseract reads the bill
      ↓
CA reviews fields → edits if needed → selects voucher type
      ↓
CA clicks "Generate XML Voucher"
      ↓
Backend builds one XML file (masters + voucher) → downloads
      ↓
CA imports into Tally Prime via O → Transactions
```

---

## Import In Tally

1. Open the client's company in Tally Prime.
2. Press F2 → set period (e.g., 1-4-2026 to 31-3-2027).
3. Press O → Transactions → select the XML file.
4. Tally creates missing ledgers and posts the voucher.

---

## Project Structure

```
PROJECT_INVEXA/
├── backend/
│   ├── app.py                 Flask app + startup
│   ├── config.py              Env loader
│   ├── extensions.py          Mongo / GridFS / Limiter
│   ├── db_setup.py            Collections + indexes
│   ├── requirements.txt
│   ├── .env
│   ├── seed_demo.py           Populates demo accounts
│   ├── models/
│   │   ├── tenant_models.py
│   │   ├── invoice_models.py
│   │   ├── chat_models.py
│   │   └── notification_models.py
│   ├── utils/
│   │   ├── passwords.py
│   │   ├── jwt_utils.py
│   │   ├── decorators.py
│   │   ├── responses.py
│   │   ├── validators.py
│   │   ├── audit.py
│   │   ├── notifications.py
│   │   ├── invite_codes.py
│   │   ├── auth_tokens.py
│   │   ├── transactions.py
│   │   ├── paging.py
│   │   ├── uploads.py
│   │   └── mailer.py
│   └── modules/
│       ├── auth/
│       ├── firm/
│       ├── client/
│       ├── invoice/
│       ├── review/
│       ├── xml/
│       ├── chat/
│       └── notifications/
│
└── frontend/
    └── src/
        ├── main.jsx
        ├── App.jsx
        ├── services/api.js
        ├── context/AuthContext.jsx
        ├── utils/time.js
        ├── components/
        │   ├── AppShell.jsx
        │   ├── ProtectedRoute.jsx
        │   ├── BillReviewModal.jsx
        │   ├── BillPreviewModal.jsx
        │   ├── NotificationsBell.jsx
        │   └── ErrorBoundary.jsx
        ├── pages/
        │   ├── Landing/
        │   ├── RoleSelect/
        │   ├── Client/
        │   └── CA/
        └── styles/
```

---

## API Endpoints

**Auth** (`/api/auth`)
- POST /signup-ca
- POST /signup-client
- POST /login
- GET  /me
- GET  /invite-lookup?code=...
- POST /forgot-password
- POST /reset-password

**Firm** (`/api/firm`, CA only)
- GET  /get-profile
- PUT  /update-profile
- POST /regenerate-invite-code
- GET  /list-clients?status=active
- GET  /list-join-requests
- POST /approve-client/<id>
- POST /reject-client/<id>

**Client** (`/api/client`)
- GET  /get-profile
- GET  /get-my-firm
- GET  /list-my-invoices
- POST /upload-invoice
- PUT  /reupload-invoice/<id>
- DELETE /delete-invoice/<id>

**Invoice** (`/api/invoice`)
- GET /list-firm-invoices
- GET /get-by-id/<id>
- GET /get-file/<id>

**Review** (`/api/review`, CA only)
- GET  /ocr-status
- POST /reextract/<id>
- PUT  /update-ocr/<id>
- PUT  /approve/<id>
- PUT  /reject/<id>
- PUT  /request-reupload/<id>
- GET  /stats

**XML** (`/api/xml`)
- GET /download/<id>?token=
- GET /preview/<id>

**Chat** (`/api/chat`)
- GET  /my-thread
- GET  /threads
- GET  /thread/<id>
- POST /send
- POST /send-to/<client_id>
- POST /upload
- POST /share-bill

**Notifications** (`/api/notifications`)
- GET  /list
- GET  /unread-count
- PUT  /mark-read/<id>
- PUT  /mark-all-read

---

## Security

- bcrypt password hashing (12 rounds)
- JWT authentication (24-hour expiry, HS256)
- NoSQL injection protection ($ keys rejected)
- Rate limiting on login (5 per 15 min) and OCR (30 per min)
- File validation: extension + MIME + magic bytes + size
- `firm_id` filter on every MongoDB query
- GridFS metadata rechecked on file download
- CORS restricted to localhost:5173

---

## Known Limitations

- OCR accuracy 60-85% on printed bills (CA reviews to fix)
- English bills only
- Tally Prime 5.x specific XML schema
- No line-item extraction (only total amounts)
- No automated tests
- Runs on localhost only

---

## Author

Punam Nikam

---

**Built with Flask - MongoDB - React - Tesseract OCR - Tally Prime**

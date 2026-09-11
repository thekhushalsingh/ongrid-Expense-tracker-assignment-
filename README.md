# Expense Tracker — Flask + React (Vite)

A personal expense tracking dashboard built with **Flask**, **React (Vite)**, and **MySQL**.

---

## Project Structure

```
ongrid-hiring-assignment/
├── backend/        — Flask API (Python)
│   ├── app.py      — Routes & application factory
│   ├── models.py   — SQLAlchemy models (Category, Expense)
│   ├── config.py   — DB config loaded from .env
│   ├── schema.sql  — Reference SQL schema
│   └── .env        — Environment variables (copy from .env.example)
└── frontend/       — React (Vite) SPA
    └── src/
        └── App.jsx — Main dashboard component
```

---

## Setup & Running

### 1. MySQL

Start a MySQL 8.x server on `localhost:3306` and create the database:

```sql
CREATE DATABASE IF NOT EXISTS expense_tracker;
```

### 2. Backend

```bash
cd backend

# Copy env file and fill in your MySQL credentials
cp .env.example .env

# Install dependencies
pip install -r requirements.txt

# Start the Flask server (runs on port 5001)
python app.py
```

**`.env` variables:**

| Variable         | Default         | Description          |
|------------------|-----------------|----------------------|
| `MYSQL_USER`     | `root`          | MySQL username       |
| `MYSQL_PASSWORD` | _(empty)_       | MySQL password       |
| `MYSQL_HOST`     | `127.0.0.1`     | MySQL host           |
| `MYSQL_PORT`     | `3306`          | MySQL port           |
| `MYSQL_DATABASE` | `expense_tracker` | Database name      |

Flask will auto-create the `category` and `expense` tables on first start via `db.create_all()`.

### 3. Frontend

```bash
cd frontend

# Install dependencies
npm install

# Start the Vite dev server (runs on port 5173)
npm run dev
```

Vite proxies all `/api` requests to `http://127.0.0.1:5001`, so no CORS issues in development.

Open **http://localhost:5173** in your browser.

---

## API Endpoints

| Method | Endpoint                        | Description                          |
|--------|---------------------------------|--------------------------------------|
| GET    | `/api/categories`               | List all categories                  |
| POST   | `/api/categories`               | Create a category `{name}`           |
| DELETE | `/api/categories/<id>`          | Delete a category (soft-deletes its expenses) |
| GET    | `/api/expenses?page=1&per_page=10` | Paginated list of active expenses |
| POST   | `/api/expenses`                 | Add an expense `{category_id, amount, expense_date, description}` |
| DELETE | `/api/expenses/<id>`            | Soft-delete an expense               |
| GET    | `/api/reports/monthly?year=&month=` | Spending by category for a month |
| GET    | `/api/reports/monthly-trend`    | 6-month rolling spend totals         |

---

## Bugs Found & Fixed

Seven bugs were identified and resolved across the backend and frontend.

### Backend — `app.py`

#### Bug 1 — Wrong Pagination Offset
**File:** `app.py` · `list_expenses()`

The offset was computed as `page * per_page`, which skipped the first page entirely (page 1 → offset 10 instead of 0).

```diff
- offset = page * per_page
+ offset = (page - 1) * per_page
```

---

#### Bug 2 — Soft-Deleted Expenses Not Filtered from List
**File:** `app.py` · `list_expenses()`

Deleted expenses (where `is_deleted = 1`) were still returned in the paginated list and counted in the total. The query lacked a filter, and the total used a separate unfiltered count.

```diff
- q = Expense.query
- total = Expense.query.count()
+ q = Expense.query.filter_by(is_deleted=0)
+ total = q.count()
```

---

#### Bug 3 — Wrong JSON Keys in Monthly Trend Response
**File:** `app.py` · `report_monthly_trend()`

The endpoint returned `{"trend_rows": [...]}` with fields `period` and `spend`, but the frontend read `data.monthly_series` with fields `month` and `total`. This key mismatch caused the trend chart to always throw a `TypeError` and display an error.

```diff
- buckets.append({"period": f"{y}-{m:02d}", "spend": s})
- return jsonify({"trend_rows": buckets})
+ buckets.append({"month": f"{y}-{m:02d}", "total": s})
+ return jsonify({"monthly_series": buckets})
```

---

### Frontend — `App.jsx`

#### Bug 4 — Wrong `dataKey` on Monthly Report Bar Chart
**File:** `App.jsx` · Reporting section

The bar chart for monthly spending by category used `dataKey="value"`, but the API response (`category_totals_for_chart`) contains the field `amt`. No bars were rendered.

```diff
- <Bar dataKey="value" fill="#1d9bf0" name="Total" />
+ <Bar dataKey="amt"   fill="#1d9bf0" name="Total" />
```

---

#### Bug 5 — Trend Data Key Mismatch (Frontend side of Bug 3)
**File:** `App.jsx` · `loadTrend()`

The frontend correctly read `data.monthly_series` and mapped `row.month` / `row.total`, but the backend was sending the wrong keys (see Bug 3). Fixing the backend resolved this automatically — no frontend change needed beyond Bug 3's fix.

---

#### Bug 6 — Page Total Concatenates Strings Instead of Summing Numbers
**File:** `App.jsx` · `pageTotal` computation

The `amount` field is returned as a **string** from the API (stored as `VARCHAR(32)` in MySQL). The `reduce` was doing string concatenation (`"0" + "12.50"` → `"012.50"`) instead of numeric addition.

```diff
- const pageTotal = expenses.reduce((a, e) => a + e.amount, 0);
+ const pageTotal = expenses.reduce((a, e) => a + parseFloat(e.amount || 0), 0).toFixed(2);
```

---

#### Bug 7 — Total Pages Uses `Math.floor` Instead of `Math.ceil`
**File:** `App.jsx` · `totalPages` computation

With 11 expenses and 10 per page, `Math.floor(11 / 10) = 1`, hiding the second page entirely. The correct formula uses ceiling division.

```diff
- const totalPages = Math.max(1, Math.floor(total / perPage));
+ const totalPages = Math.max(1, Math.ceil(total / perPage));
```

---

## Assignment Submission Guidelines

1. Find **at least 5 issues** in the code base (Frontend and/or Backend)
2. Record a video sharing your screen and turning on your video to explain how you went about solving the assignment.
3. Submit this form — https://forms.gle/AK9U6VjnuDFRS7R36

### Evaluation Criteria
- How did you reach the bugs?
- Quality of your solutions
- After resolving all the bugs, the dashboard should look something like this — https://drive.google.com/file/d/1s07a-L_rAT8OgFg28D3gp-stxJJUHdd2/view?usp=sharing

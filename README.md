# AI/BI External Embedding Template

A minimal template showing how to embed Databricks dashboards into external applications with user authentication and row-level security.

## What This Template Shows

1. **Backend (Flask)**: Authenticate users and mint Databricks OAuth tokens
2. **Frontend (React)**: Use the [Databricks embedding SDK](https://www.npmjs.com/package/@databricks/aibi-client) to display dashboards
3. **Row-Level Security**: Pass user context for data filtering with `__aibi_external_value`

## Project Structure

```
aibi-external-embedding/
│
├── backend/                          # Flask Backend
│   ├── app.py                       # Main application - START HERE
│   ├── requirements.txt             # Python dependencies
│   └── .env.example                 # Configuration template
│
└── frontend/                         # React Frontend
    ├── src/
    │   ├── components/
    │   │   └── DashboardEmbed.jsx  # Dashboard embedding component
    │   ├── App.jsx                  # Main React component - START HERE
    │   ├── App.css                  # Styles
    │   ├── main.jsx                 # Entry point
    │   └── index.css                # Global styles
    ├── index.html                   # HTML template
    ├── package.json                 # Node dependencies
    └── vite.config.js               # Dev server config
```

## How It Works: File Relationships

### 1. User Logs In



**Frontend** (`frontend/src/App.jsx`)
```javascript
// User clicks login → sends username to backend
const handleLogin = async (username) => {
  await axios.post('/api/auth/login', { username: username })
}
```

**Backend** (`backend/app.py`)
```python
@app.route('/api/auth/login', methods=['POST'])
def login():
    # Validates username
    # Creates user session
    session['user_id'] = user['id']
    return jsonify({'user': user})
```

### 2. Frontend Requests Dashboard Configuration

**Frontend** (`frontend/src/App.jsx`)
```javascript
// After login, request dashboard config and token
const fetchDashboardConfig = async () => {
  const response = await axios.get('/api/dashboard/embed-config')
  // Response includes: workspace_url, dashboard_id, embed_token
  setDashboardConfig(response.data)
}
```

**Backend** (`backend/app.py`)
```python
@app.route('/api/dashboard/embed-config')
@login_required
def get_embed_config():
    # Gets current user from session
    user = DUMMY_USERS.get(session['username'])
    
    # Mints OAuth token for this specific user
    token_data = mint_databricks_token(user)
    
    # Returns dashboard config + token to frontend
    return jsonify({
        'workspace_url': os.environ.get('DATABRICKS_WORKSPACE_URL'),
        'dashboard_id': os.environ.get('DATABRICKS_DASHBOARD_ID'),
        'embed_token': token_data['access_token'],
        'user_context': user
    })
```

### 3. Token Minting (Key Function)

**Backend** (`backend/app.py`)
```python
def mint_databricks_token(user_data):
    """
    Creates an OAuth token for the authenticated user using 
    the official Databricks 3-step token generation process.
    
    Steps:
    1. Get all-apis token from OIDC endpoint
    2. Get token info for the dashboard with external viewer context
    3. Generate scoped token with authorization details
    
    The user context (external_viewer_id and external_value) enables 
    row-level security in your dashboards.
    """
    # Step 1: Get all-apis token
    oidc_response = requests.post(
        f"{workspace_url}/oidc/v1/token",
        data={"grant_type": "client_credentials", "scope": "all-apis"}
    )
    
    # Step 2: Get token info with user context
    token_info_response = requests.get(
        f"{workspace_url}/api/2.0/lakeview/dashboards/{dashboard_id}/published/tokeninfo"
        f"?external_viewer_id={user_data['email']}"
        f"&external_value={user_data['department']}"
    )
    
    # Step 3: Generate scoped token
    scoped_response = requests.post(
        f"{workspace_url}/oidc/v1/token",
        data={
            "grant_type": "client_credentials",
            "authorization_details": json.dumps(authorization_details)
        }
    )
    
    return scoped_response.json()['access_token']
```

### 4. Dashboard Embedding

**Frontend** (`frontend/src/components/DashboardEmbed.jsx`)
```javascript
import { DatabricksDashboard } from '@databricks/aibi-client'

// Initialize dashboard with automatic token refresh
const dashboard = new DatabricksDashboard({
  instanceUrl: config.workspace_url,
  workspaceId: config.workspace_id,
  dashboardId: config.dashboard_id,
  token: config.embed_token,
  container: containerRef.current,
  
  // SDK calls this callback ~5 min before token expires
  getNewToken: async () => {
    const response = await axios.get('/api/dashboard/embed-config')
    return response.data.embed_token
  }
})

await dashboard.initialize()
```

## Key Configuration

### Backend → Frontend Communication

**API Proxy** (`frontend/vite.config.js`):
```javascript
proxy: {
  '/api': {
    target: 'http://localhost:5000'  // Routes /api/* to Flask backend
  }
}
```

**Session Management**: Flask sessions track logged-in users, frontend sends cookies with `withCredentials: true`

### Row-Level Security

The backend passes user context when minting tokens:
```python
# In mint_databricks_token() - passes department as external_value
token_info_url = (
    f"{workspace_url}/api/2.0/lakeview/dashboards/{dashboard_id}/published/tokeninfo"
    f"?external_viewer_id={user_data['email']}"
    f"&external_value={user_data['department']}"  # e.g., "Sales"
)
```

In your Databricks dashboard dataset, filter using `aibi_external_value`:
```sql
SELECT * FROM sales_data
WHERE department = __aibi_external_value  -- Returns "Sales" for Alice, "Engineering" for Bob
```

This enables row-level security: each user only sees data for their department.

## Demo Users

| Username | Department | Purpose |
|----------|-----------|---------|
| `alice` | Sales | See sales data |
| `bob` | Engineering | See engineering data |

## Quick Setup

### 1. Backend Setup

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Copy the example file and add your credentials
cp .env.example .env
code .env  # Edit with your actual Databricks credentials

python app.py  # Runs on http://localhost:5000
```

**Security Note:**
- ⚠️ **NEVER commit `.env` to version control** - it contains your real credentials
- ✅ `.env` is already in `.gitignore` and will not be tracked by git
- ✅ Only commit `.env.example` with placeholder values

### 2. Frontend Setup

```bash
cd frontend
npm install  # Installs dependencies including @databricks/aibi-client@1.0.0
npm run dev  # Runs on http://localhost:3000
```

**Key Dependency:** This template uses the official [@databricks/aibi-client](https://www.npmjs.com/package/@databricks/aibi-client) npm package (v1.0.0) for dashboard embedding.

### 3. Try It Out

1. Open http://localhost:3000
2. Click "Alice Johnson" or "Bob Smith" to login
3. Dashboard loads with that user's token
4. Switch users to see the token refresh

## Understanding the Flow

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Browser   │         │    Flask    │         │ Databricks  │
│  (React)    │         │   Backend   │         │             │
└─────────────┘         └─────────────┘         └─────────────┘
      │                       │                       │
      │  1. Login (alice)     │                       │
      │─────────────────────>│                       │
      │                       │                       │
      │  2. Session created   │                       │
      │<─────────────────────│                       │
      │                       │                       │
      │  3. Get dashboard     │                       │
      │     config & token    │                       │
      │─────────────────────>│                       │
      │                       │                       │
      │                       │  4. Mint OAuth token  │
      │                       │     for Alice         │
      │                       │─────────────────────>│
      │                       │                       │
      │                       │  5. Return token      │
      │                       │<─────────────────────│
      │                       │                       │
      │  6. Config + token    │                       │
      │<─────────────────────│                       │
      │                       │                       │
      │  7. Embed dashboard   │                       │
      │     with Alice's      │                       │
      │     token in iframe   │                       │
      │────────────────────────────────────────────>│
      │                       │                       │
      │  8. Dashboard renders │                       │
      │     with Alice's data │                       │
      │<────────────────────────────────────────────│
```

## Files You Should Read

Start with these files to understand the core flow:

1. **`backend/app.py`** - Token minting with Databricks OAuth, user sessions, RLS
2. **`frontend/src/App.jsx`** - Login flow, API integration, user switching
3. **`frontend/src/components/DashboardEmbed.jsx`** - Databricks SDK usage, automatic token refresh
4. **`backend/.env.example`** - Required configuration template

## Key Backend Functions

| Function | Purpose |
|----------|---------|
| `mint_databricks_token()` | Creates OAuth token with user context |
| `/api/dashboard/embed-config` | Returns dashboard config + token |

## Key Frontend Components & Functions

| Component/Function | Purpose |
|-------------------|---------|
| `App.jsx` | Main app - handles auth and state |
| `DashboardEmbed.jsx` | Embeds dashboard using Databricks SDK |
| `handleLogin()` | Sends credentials to backend |
| `fetchDashboardConfig()` | Gets dashboard config and token |

## Customization Points

1. **Add Real Authentication** (`backend/app.py`):
   - Replace `DUMMY_USERS` with database lookup
   - Add password hashing
   - Integrate with OAuth providers

2. **Multiple Dashboards** (`frontend/src/App.jsx`):
   - Add dashboard selector
   - Pass dashboard ID to backend
   - Support different views per user


## Getting Databricks Credentials

1. **OAuth Credentials**:
   - Go to Databricks → Settings → Developer → OAuth Apps
   - Create new app
   - Copy Client ID and Secret

2. **Dashboard ID**:
   - Open your dashboard
   - Copy ID from URL: `/sql/dashboards/{DASHBOARD_ID}`

3. **Warehouse ID**:
   - Go to SQL Warehouses
   - Click your warehouse
   - Copy ID from URL or details

## Common Questions

**Q: Where is the OAuth token generated?**  
A: In `backend/app.py`, function `mint_databricks_token()` (starts at line 57) using the official Databricks 3-step OAuth flow

**Q: How does the frontend get the token?**  
A: Calls `/api/dashboard/embed-config` endpoint which returns the token (line 219 in `app.py`)

**Q: How is the dashboard embedded?**  
A: Using `@databricks/aibi-client` SDK in `frontend/src/components/DashboardEmbed.jsx` with static import from CDN

**Q: How does user switching work?**  
A: Logs out current user, logs in new user, fetches new token (see `handleUserSwitch()` at line 94 in `App.jsx`)

**Q: How does row-level security work?**  
A: User context (`external_value`) is passed in token generation. In your dashboard SQL, use `__aibi_external_value` to filter data by the user's department (see line 147 in `app.py`)

## Key Dependencies

### Frontend
- **[@databricks/aibi-client](https://www.npmjs.com/package/@databricks/aibi-client)** (v1.0.0) - Official Databricks SDK for dashboard embedding
- **React** (v18.2.0) - UI framework
- **Axios** (v1.6.0) - HTTP client for API calls
- **Vite** (v5.0.0) - Build tool and dev server

### Backend
- **Flask** (v3.0.0) - Python web framework
- **Flask-CORS** (v4.0.0) - CORS support for frontend communication
- **python-dotenv** (v1.0.0) - Environment variable management
- **requests** (v2.31.0) - HTTP library for Databricks OAuth API calls

## Next Steps

1. Set up your `.env` file with Databricks credentials
2. Run backend and frontend
3. Login as Alice or Bob
4. Examine the code to understand the flow
5. Customize for your use case
6. Check the [Databricks AI/BI External Embedding](https://docs.databricks.com/aws/en/dashboards/embedding/external-embed#gsc.tab=0) and [Databricks SDK](https://www.npmjs.com/package/@databricks/aibi-client) documentation for more details.

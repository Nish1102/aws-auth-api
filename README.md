# Backend Setup Instructions

This guide walks you through setting up a simple Node.js backend with PostgreSQL for storing and retrieving user login information.

---




## Step 1: Install and Configure PostgreSQL

### 1.1 Install PostgreSQL
```bash
sudo apt update
sudo apt install postgre postgre-contrib

### 1.1 Install PostgreSQL
```bash
sudo apt update
sudo apt install postgre postgre-contrib

### 1.2 Create a New Database

1. Open the PostgreSQL prompt as the postgres user:
   ```bash
   sudo -u postgres p
   ```
2. Inside the prompt, create and connect to the database:
   ```sql
   CREATE DATABASE logininfo;
   \c logininfo
   ```

### 1.3 Create a Table for Users

Inside the PostgreSQL prompt, create the users table:
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) NOT NULL,
  password VARCHAR(255) NOT NULL
);
```

### 1.4 Set or Update the `postgres` User Password

At the PostgreSQL prompt, run:
```sql
\password postgres
```
- Enter a password (this guide uses `root` for demonstration, but use a strong password in production).
- Quit the prompt by typing:
  ```sql
  \q
  ```

### 1.5 Verify the Database

Reopen the PostgreSQL prompt:
```bash
sudo -u postgres p
```
Then run:
```sql
\l          -- Lists all databases
\c logininfo  -- Connects to logininfo
\dt         -- Lists tables in logininfo
```
Make sure the `users` table is present.

### 1.6 Test the Connection

Test connecting to the database:
```bash
p -U postgres -h localhost -d logininfo
```
A successful prompt (without errors) indicates the connection is working.

---

## Step 2: Install Node.js and npm

Update package lists and install:
```bash
sudo apt-get update
sudo apt-get install -y nodejs npm
```

---

## Step 3: Set Up the Backend Project

### 3.1 Create and Navigate to the Project Directory
```bash
mkdir backend
cd backend
```

### 3.2 Initialize a New Node.js Project
```bash
npm init -y
```

### 3.3 Install Required Dependencies
```bash
npm install express body-parser pg
npm install cors
```
- **express:** For creating the server.
- **body-parser:** For parsing incoming request bodies.
- **pg:** PostgreSQL client for Node.js.
- **cors:** Allows cross-origin resource sharing.

---

## Step 4: Create `server.js`

Create a file named `server.js` in the `backend` folder and add the following code:

```javascript
const express = require('express');
const cors = require('cors');
const { Pool } = require('pg');

const app = express();
const port = 3001;

// 1) Enable CORS and parse JSON
app.use(cors());
// Handle preflight requests on all routes:
app.options('*', cors());
app.use(express.json());

// 2) Configure PostgreSQL
const pool = new Pool({
  user: 'postgres',
  host: 'localhost',
  database: 'logininfo',
  password: 'root',
  port: 5432,
});

// 3) POST /login (NO auto-create)
app.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;

    // A) Find user by username
    const result = await pool.query(
      'SELECT * FROM users WHERE username = $1',
      [username]
    );

    if (result.rows.length === 0) {
      // No user found -> do not create automatically
      return res.status(401).json({
        success: false,
        message: 'User does not exist',
      });
    }

    // B) Compare plain-text password (remove bcrypt logic)
    const user = result.rows[0];
    if (user.password !== password) {
      // Wrong password
      return res.status(401).json({
        success: false,
        message: 'Invalid credentials',
      });
    }

    // C) Login successful
    return res.json({
      success: true,
      message: 'Login successful',
    });
  } catch (err) {
    console.error('Database error:', err);
    return res.status(500).json({
      success: false,
      error: 'Database error',
    });
  }
});

// 4) Listen on 0.0.0.0 so it’s publicly accessible
app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
```

---

## Step 5: Run the Server

### 5.1 Start the Server in the Foreground
```bash
node server.js
```
- You should see: `Server running on port 3001`.

### 5.2 Test the Backend Endpoint
```bash
curl -X POST http://54.243.3.137:3001/login \
     -H "Content-Type: application/json" \
     -d '{"username": "test", "password": "test"}'
```
- The response should indicate success if the credentials match.

---

## Optional: Using PM2 for Process Management

### 6.1 Install PM2 Globally
```bash
sudo npm install -g pm2
```

### 6.2 Start the Server with PM2
```bash
pm2 start server.js
```

### 6.3 Set Up PM2 to Start on System Boot
```bash
pm2 startup
pm2 save
```

### 6.4 Check PM2 Status
```bash
pm2 status
```
- This confirms `server.js` is running and will automatically restart if it crashes.

---

## Troubleshooting

### Connection Errors
- **Verify Credentials:** Ensure PostgreSQL credentials in `server.js` are correct.
- **Check PostgreSQL Status:**
  ```bash
  sudo systemctl status postgre
  ```
- **Remote Connections:** Ensure port 5432 is open if connecting remotely.

### Database Issues
- **Database Existence:** Confirm that `logininfo` was created in PostgreSQL.
- **Correct Database Name:** Ensure `server.js` references the correct database.

### PM2 Permissions
- If PM2 encounters permission issues, run it with `sudo` or adjust user privileges accordingly.

---

## For Frontend Deployment

### 1. Install Nginx
```bash
sudo apt-get install nginx
```

### 2. Transfer the Frontend Build
```bash
scp -i awsslogin.pem -r /home/nishant/Project/Deployment/FRONTEND/frontend/build ubuntu@3.95.16.141:~/
```

### 3. Deploy the Build to Nginx
```bash
sudo rm -rf /var/www/html/static
sudo mv ~/build/* /var/www/html/
```

---
```

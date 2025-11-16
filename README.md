# RekLog

Professional request logging and performance monitoring for Node.js applications. Track API response times, analyze performance, and monitor your endpoints in real-time.

**Server-side only** - Designed for Node.js/Express applications.

## Features

- 🚀 **Express Middleware**: Automatic logging for all Express routes
- 📊 **Performance Tracking**: High-precision timing with `process.hrtime.bigint()`
- 📝 **Complete Request/Response Logging**: Capture headers, body, params, and more
- 🔄 **Automatic Retries**: Built-in retry logic for failed log submissions
- 🐛 **Debug Mode**: Detailed logging for development
- ⚡ **Non-Blocking**: Async logging doesn't slow down your API
- 📈 **Dashboard Analytics**: View logs and charts in RekLog Dashboard

## Installation

```bash
npm install reklog
```

## Quick Start

### 1. Get Your API Key

1. Sign up at [RekLog Dashboard](https://www.reklog.com)
2. Create a new project
3. Copy your API key (starts with `rkl_`)

### 2. Initialize RekLog

```javascript
const reklog = require('reklog');

// Simple initialization - uses https://www.reklog.com/api by default
const logger = reklog.init(process.env.REKLOG_API_KEY);

// With environment (defaults to process.env.NODE_ENV or 'development')
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  environment: 'production' // or 'development', 'test', 'staging', etc.
});
```

### 3. Use as Express Middleware (Recommended)

Automatically log all requests:

```javascript
const express = require('express');
const reklog = require('reklog');

const app = express();
app.use(express.json()); // Required for body parsing

const logger = reklog.init(process.env.REKLOG_API_KEY);

// Add RekLog middleware - logs all requests automatically
app.use(logger.middleware());

app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

app.post('/api/users', (req, res) => {
  const user = req.body;
  res.status(201).json({ id: 1, ...user });
});

app.listen(3000);
```

**What gets logged automatically:**
- ✅ Endpoint and HTTP method
- ✅ Response time (high precision)
- ✅ Status code
- ✅ Request body (POST, PUT, PATCH)
- ✅ Query parameters (GET)
- ✅ Request headers (content-type, user-agent, accept)
- ✅ Response body
- ✅ Route parameters

## Manual Logging

For external API calls or custom tracking:

### GET Request Example

```javascript
async function fetchHotels(city) {
  const logId = logger.start('/api/hotels/search', 'GET');

  try {
    const response = await axios.get('https://api.example.com/hotels', {
      params: { city, limit: 10 }
    });

    await logger.end(logId, {
      statusCode: response.status,
      params: { city, limit: 10 },  // Query parameters
      response: response.data,
      environment: 'production'  // Optional: override default environment
    });

    return response.data;
  } catch (error) {
    await logger.end(logId, {
      statusCode: error.response?.status || 500,
      params: { city, limit: 10 },
      metadata: { error: error.message }
    });
    throw error;
  }
}
```

### POST Request Example

```javascript
async function createUser(userData) {
  const logId = logger.start('/api/users', 'POST');

  try {
    const response = await axios.post('https://api.example.com/users', userData);

    await logger.end(logId, {
      statusCode: response.status,
      body: userData,           // Request body
      response: response.data
    });

    return response.data;
  } catch (error) {
    await logger.end(logId, {
      statusCode: error.response?.status || 500,
      body: userData,
      metadata: { error: error.message }
    });
    throw error;
  }
}
```

### PUT/PATCH Request Example

```javascript
async function updateUser(userId, updates) {
  const logId = logger.start(`/api/users/${userId}`, 'PUT');

  try {
    const response = await axios.put(
      `https://api.example.com/users/${userId}`,
      updates
    );

    await logger.end(logId, {
      statusCode: response.status,
      body: updates,
      response: response.data,
      metadata: { userId }
    });

    return response.data;
  } catch (error) {
    await logger.end(logId, {
      statusCode: error.response?.status || 500,
      body: updates,
      metadata: { userId, error: error.message }
    });
    throw error;
  }
}
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| apiUrl | string | `https://www.reklog.com/api` | RekLog API URL (use for self-hosted installations) |
| debug | boolean | false | Enable debug logging (shows detailed logs in console) |
| retryAttempts | number | 3 | Number of retry attempts for failed logs |
| retryDelay | number | 1000 | Delay between retries in milliseconds |

**Basic usage (uses reklog.com):**
```javascript
const logger = reklog.init(process.env.REKLOG_API_KEY);
```

**With options:**
```javascript
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  debug: process.env.NODE_ENV === 'development',
  retryAttempts: 3,
  retryDelay: 1000
});
```

## API Reference

### `reklog.init(apiKey, options)`

Initialize RekLog with your API key.

**Parameters:**
- `apiKey` (string, required): Your RekLog API key (starts with `rkl_`)
- `options` (object, optional):
  - `apiUrl` (string): RekLog API URL (default: `'https://www.reklog.com/api'`)
  - `debug` (boolean): Enable debug mode (default: `false`)
  - `retryAttempts` (number): Retry attempts on failure (default: `3`)
  - `retryDelay` (number): Delay between retries in ms (default: `1000`)

**Returns:** RekLog logger instance

**Example:**
```javascript
// Basic - uses reklog.com
const logger = reklog.init(process.env.REKLOG_API_KEY);

// With debug mode
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  debug: true
});
```

---

### `logger.start(endpoint, method)`

Start tracking a request. Returns a unique log ID.

**Parameters:**
- `endpoint` (string, required): The endpoint path (e.g., `/api/users`)
- `method` (string, optional): HTTP method (default: `'GET'`)

**Returns:** Log ID (string)

**Example:**
```javascript
const logId = logger.start('/api/users', 'POST');
```

---

### `logger.end(logId, options)`

End tracking and send log data to RekLog API.

**Parameters:**
- `logId` (string, required): The log ID from `start()`
- `options` (object, optional):
  - `statusCode` (number): HTTP status code (default: `200`)
  - `body` (object): Request body for POST/PUT/PATCH
  - `params` (object): Query parameters for GET
  - `requestHeaders` (object): Request headers
  - `response` (object): Response data
  - `metadata` (object): Additional custom data

**Returns:** Promise

**Example:**
```javascript
await logger.end(logId, {
  statusCode: 201,
  body: { name: 'John', email: 'john@example.com' },
  response: { id: 123, name: 'John' }
});
```

---

### `logger.middleware()`

Returns Express middleware function for automatic request logging.

**Returns:** Express middleware function

**Example:**
```javascript
app.use(logger.middleware());
```

**Automatically captures:**
- Request endpoint and method
- Response time (nanosecond precision)
- Status code
- Request body (`req.body`)
- Query parameters (`req.query`)
- Request headers (content-type, user-agent, accept)
- Response body
- Route parameters (`req.params`)

---

## Debug Mode

Enable debug mode to see detailed logs during development:

```javascript
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  debug: true
});
```

**Debug output shows:**
- Options passed to `logger.end()`
- Full log data being sent to server
- Success/failure messages
- Retry attempts

**Note:** Disable debug mode in production for better performance.

---

## Best Practices

### 1. Use Environment Variables

```javascript
// .env
REKLOG_API_KEY=rkl_your_api_key_here

// app.js
require('dotenv').config();

const logger = reklog.init(process.env.REKLOG_API_KEY, {
  debug: process.env.NODE_ENV === 'development'
});
```

### 2. Middleware Order

Place RekLog middleware after body parser:

```javascript
const express = require('express');
const reklog = require('reklog');

const app = express();

app.use(express.json());        // 1. Body parser first
app.use(logger.middleware());   // 2. RekLog middleware second

app.get('/api/users', handler); // 3. Your routes
```

### 3. Error Handling

Always log errors with appropriate status codes:

```javascript
try {
  const data = await externalAPI.call();
  await logger.end(logId, {
    statusCode: 200,
    response: data
  });
} catch (error) {
  await logger.end(logId, {
    statusCode: error.response?.status || 500,
    metadata: {
      error: error.message,
      stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
    }
  });
}
```

---

## Complete Example

```javascript
require('dotenv').config();
const express = require('express');
const axios = require('axios');
const reklog = require('reklog');

const app = express();
app.use(express.json());

// Initialize RekLog
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  debug: process.env.NODE_ENV === 'development'
});

// Use middleware for automatic logging
app.use(logger.middleware());

// Your routes - automatically logged
app.get('/api/users', async (req, res) => {
  const { page = 1, limit = 10 } = req.query;
  const users = await db.users.find().limit(limit).skip((page - 1) * limit);
  res.json({ users, page, limit });
});

app.post('/api/users', async (req, res) => {
  const user = await db.users.create(req.body);
  res.status(201).json(user);
});

// Manual logging for external API calls
app.get('/api/weather/:city', async (req, res) => {
  const logId = logger.start('/external/weather', 'GET');

  try {
    const response = await axios.get(`https://api.weather.com/weather`, {
      params: { city: req.params.city }
    });

    await logger.end(logId, {
      statusCode: response.status,
      params: { city: req.params.city },
      response: response.data
    });

    res.json(response.data);
  } catch (error) {
    await logger.end(logId, {
      statusCode: error.response?.status || 500,
      params: { city: req.params.city },
      metadata: { error: error.message }
    });

    res.status(500).json({ error: 'Failed to fetch weather data' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## Self-Hosted Installation

If you want to run your own RekLog server instead of using reklog.com:

### 1. Clone and Setup RekLog Server

```bash
git clone https://github.com/yasaricli/reklog.git
cd reklog

# Setup API
cd api
npm install
cp .env.example .env
# Edit .env and configure your MongoDB URI
npm start

# Setup Dashboard
cd ../client
npm install
npm run dev
```

### 2. Configure Your Application

Point your application to your self-hosted server:

```javascript
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  apiUrl: 'http://your-server.com:3000/api'  // Your RekLog server
});
```

### 3. Environment Variables for Self-Hosted

```bash
# .env
REKLOG_API_KEY=rkl_your_api_key_here
REKLOG_API_URL=http://your-server.com:3000/api
```

```javascript
// app.js
const logger = reklog.init(process.env.REKLOG_API_KEY, {
  apiUrl: process.env.REKLOG_API_URL
});
```

---

## Support

- **Website:** [https://www.reklog.com](https://www.reklog.com)
- **Dashboard:** [https://www.reklog.com](https://www.reklog.com)
- **Issues:** [GitHub Issues](https://github.com/yasaricli/reklog/issues)
- **Documentation:** [https://www.reklog.com/docs](https://www.reklog.com/docs)

## License

MIT

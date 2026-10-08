
We will create a search API that searches cards by title and description.

1. Update Card Controller

Open:

server/controllers/cardController.js

Add:

const searchCards = async (req, res) => {
  try {
    const { boardId } = req.params;
    const { q } = req.query;

    if (!q || !q.trim()) {
      return res.json([]);
    }

    const Board = require("../models/Board");
    const Workspace = require("../models/Workspace");

    const board = await Board.findById(boardId);

    if (!board) {
      return res.status(404).json({
        message: "Board not found"
      });
    }

    const workspace = await Workspace.findOne({
      _id: board.workspace,
      members: req.user._id
    });

    if (!workspace) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const cards = await Card.find({
      $or: [
        { title: { $regex: q, $options: "i" } },
        { description: { $regex: q, $options: "i" } }
      ]
    })
      .populate("assignee", "name email")
      .sort({ createdAt: -1 });

    res.json(cards);
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

At the bottom:

module.exports = {
  createCard,
  getCards,
  updateCard,
  deleteCard,
  moveCard,
  searchCards
};
2. Add Search Route

Open:

server/routes/cardRoutes.js

Change the imports:

const {
  createCard,
  getCards,
  updateCard,
  deleteCard,
  moveCard,
  searchCards
} = require("../controllers/cardController");

Add:

router.get("/search/:boardId", protect, searchCards);

So your routes contain:

router.post("/", protect, createCard);

router.get("/list/:listId", protect, getCards);

router.get("/search/:boardId", protect, searchCards);

router.put("/:id", protect, updateCard);

router.delete("/:id", protect, deleteCard);

router.put("/:id/move", protect, moveCard);
3. Create Search Component

Create:

client/src/components/SearchBar.jsx
import { useState } from "react";
import api from "../services/api";

function SearchBar({ boardId }) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  const handleSearch = async (e) => {
    const value = e.target.value;

    setQuery(value);

    if (!value.trim()) {
      setResults([]);
      return;
    }

    try {
      const response = await api.get(
        `/cards/search/${boardId}?q=${encodeURIComponent(value)}`
      );

      setResults(response.data);
    } catch (error) {
      console.error("Search failed:", error);
    }
  };

  return (
    <div className="relative">
      <input
        type="text"
        placeholder="Search cards..."
        value={query}
        onChange={handleSearch}
        className="border p-2 rounded w-64"
      />

      {results.length > 0 && (
        <div className="absolute bg-white border rounded mt-1 w-64 z-50">
          {results.map((card) => (
            <div
              key={card._id}
              className="p-2 border-b hover:bg-gray-100"
            >
              <p className="font-semibold">{card.title}</p>

              <p className="text-sm text-gray-500">
                {card.description}
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

export default SearchBar;

Use it in Board.jsx:

import SearchBar from "../components/SearchBar";

Inside your board UI:

<SearchBar boardId={boardId} />
Day 22 Result

You now have:

Search
   ↓
Search Card API
   ↓
MongoDB
   ↓
Matching Cards
   ↓
Display Results
DAY 23 — Notifications

Now we create real-time notifications.

Examples:

Someone assigns a task to you.
Someone comments on your card.
Someone moves/updates your task.
1. Notification Model

Create:

server/models/Notification.js
const mongoose = require("mongoose");

const notificationSchema = new mongoose.Schema(
  {
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    },

    message: {
      type: String,
      required: true
    },

    type: {
      type: String,
      default: "general"
    },

    read: {
      type: Boolean,
      default: false
    },

    board: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Board",
      default: null
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model(
  "Notification",
  notificationSchema
);
2. Notification Controller

Create:

server/controllers/notificationController.js
const Notification = require("../models/Notification");

const getNotifications = async (req, res) => {
  try {
    const notifications = await Notification.find({
      user: req.user._id
    })
      .sort({ createdAt: -1 })
      .limit(50);

    res.json(notifications);
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

const markAsRead = async (req, res) => {
  try {
    const notification = await Notification.findOneAndUpdate(
      {
        _id: req.params.id,
        user: req.user._id
      },
      {
        read: true
      },
      {
        new: true
      }
    );

    if (!notification) {
      return res.status(404).json({
        message: "Notification not found"
      });
    }

    res.json(notification);
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

const markAllAsRead = async (req, res) => {
  try {
    await Notification.updateMany(
      {
        user: req.user._id,
        read: false
      },
      {
        read: true
      }
    );

    res.json({
      message: "All notifications marked as read"
    });
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

module.exports = {
  getNotifications,
  markAsRead,
  markAllAsRead
};
3. Notification Routes

Create:

server/routes/notificationRoutes.js
const express = require("express");

const {
  getNotifications,
  markAsRead,
  markAllAsRead
} = require("../controllers/notificationController");

const protect = require("../middleware/authMiddleware");

const router = express.Router();

router.get("/", protect, getNotifications);

router.put("/:id/read", protect, markAsRead);

router.put("/read-all", protect, markAllAsRead);

module.exports = router;
4. Add Route to Server

Open:

server/server.js

Add:

const notificationRoutes = require("./routes/notificationRoutes");

Then:

app.use("/api/notifications", notificationRoutes);
5. Real-Time Notification

Update:

server/socket/socketHandler.js

Add:

socket.on("join-user", (userId) => {
  socket.join(`user:${userId}`);
});

Now a user can join their own notification room.

6. Notification Component

Create:

client/src/components/Notifications.jsx
import { useEffect, useState } from "react";
import api from "../services/api";
import socket from "../services/socket";

function Notifications({ userId }) {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    loadNotifications();

    socket.connect();

    socket.emit("join-user", userId);

    const handleNotification = (notification) => {
      setNotifications((previous) => [
        notification,
        ...previous
      ]);
    };

    socket.on("notification", handleNotification);

    return () => {
      socket.off("notification", handleNotification);
      socket.disconnect();
    };
  }, [userId]);

  const loadNotifications = async () => {
    try {
      const response = await api.get("/notifications");

      setNotifications(response.data);
    } catch (error) {
      console.error(error);
    }
  };

  const markAsRead = async (id) => {
    await api.put(`/notifications/${id}/read`);

    setNotifications((previous) =>
      previous.map((notification) =>
        notification._id === id
          ? { ...notification, read: true }
          : notification
      )
    );
  };

  return (
    <div className="w-80 bg-white border rounded shadow">
      <div className="p-3 font-bold border-b">
        Notifications
      </div>

      {notifications.length === 0 ? (
        <p className="p-3 text-gray-500">
          No notifications
        </p>
      ) : (
        notifications.map((notification) => (
          <div
            key={notification._id}
            onClick={() => markAsRead(notification._id)}
            className={`p-3 border-b cursor-pointer ${
              notification.read
                ? "bg-white"
                : "bg-blue-50"
            }`}
          >
            {notification.message}
          </div>
        ))
      )}
    </div>
  );
}

export default Notifications;
DAY 24 — Redis Setup

You already installed:

npm install redis

Now create:

server/config/redis.js
const { createClient } = require("redis");

const redisClient = createClient({
  url: process.env.REDIS_URL || "redis://127.0.0.1:6379"
});

redisClient.on("error", (error) => {
  console.error("Redis Error:", error);
});

const connectRedis = async () => {
  try {
    if (!redisClient.isOpen) {
      await redisClient.connect();
    }

    console.log("Redis connected successfully");
  } catch (error) {
    console.error("Redis connection failed:", error.message);
  }
};

module.exports = {
  redisClient,
  connectRedis
};
Update .env

Add:

REDIS_URL=redis://127.0.0.1:6379

If you are using Redis Cloud later, replace this with your Redis Cloud connection URL.

Update server.js

Add:

const { connectRedis } = require("./config/redis");

Then:

connectRedis();

Your server will now use:

MongoDB
   +
Redis
   +
Express
   +
Socket.io
DAY 25 — Redis Caching

Caching helps reduce repeated MongoDB queries.

For example:

User requests boards
        ↓
Check Redis
   ↓          ↓
Found       Not Found
 ↓              ↓
Return       MongoDB
                ↓
             Redis
                ↓
             Return
Create Cache Service

Create:

server/services/cacheService.js
const { redisClient } = require("../config/redis");

const getCache = async (key) => {
  try {
    const data = await redisClient.get(key);

    if (!data) {
      return null;
    }

    return JSON.parse(data);
  } catch (error) {
    console.error("Redis get error:", error.message);
    return null;
  }
};

const setCache = async (key, data, expiration = 300) => {
  try {
    await redisClient.set(
      key,
      JSON.stringify(data),
      {
        EX: expiration
      }
    );
  } catch (error) {
    console.error("Redis set error:", error.message);
  }
};

const deleteCache = async (key) => {
  try {
    await redisClient.del(key);
  } catch (error) {
    console.error("Redis delete error:", error.message);
  }
};

module.exports = {
  getCache,
  setCache,
  deleteCache
};
Cache Workspaces

Open:

server/controllers/workspaceController.js

Add:

const {
  getCache,
  setCache,
  deleteCache
} = require("../services/cacheService");

Modify getMyWorkspaces:

const getMyWorkspaces = async (req, res) => {
  try {
    const cacheKey = `workspaces:${req.user._id}`;

    const cached = await getCache(cacheKey);

    if (cached) {
      return res.json(cached);
    }

    const workspaces = await Workspace.find({
      members: req.user._id
    }).populate(
      "owner",
      "name email"
    );

    await setCache(
      cacheKey,
      workspaces,
      300
    );

    res.json(workspaces);
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

After creating a workspace:

await deleteCache(
  `workspaces:${req.user._id}`
);

This ensures old cached data is removed.

DAY 26 — Security

This is one of the most important days.

Install:

cd server
npm install helmet express-rate-limit express-validator
1. Helmet

In server.js:

const helmet = require("helmet");

Then:

app.use(helmet());

This adds useful HTTP security headers.

2. Rate Limiting

Add:

const rateLimit = require("express-rate-limit");

Create:

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "Too many requests. Please try again later."
});

Then:

app.use("/api/", limiter);
3. Better CORS

Instead of:

app.use(cors());

use:

app.use(
  cors({
    origin: process.env.CLIENT_URL,
    credentials: true
  })
);
4. Socket.io JWT Authentication

Your project specification requires secure WebSocket handshakes.

Update:

server/socket/socketHandler.js

Use:

const jwt = require("jsonwebtoken");

const setupSocket = (io) => {
  io.use((socket, next) => {
    try {
      const token = socket.handshake.auth.token;

      if (!token) {
        return next(
          new Error("Authentication required")
        );
      }

      const decoded = jwt.verify(
        token,
        process.env.JWT_SECRET
      );

      socket.userId = decoded.userId;

      next();
    } catch (error) {
      next(
        new Error("Invalid socket authentication")
      );
    }
  });

  io.on("connection", (socket) => {
    console.log(
      `Authenticated user connected: ${socket.userId}`
    );

    socket.join(`user:${socket.userId}`);

    socket.on("join-board", (boardId) => {
      socket.join(`board:${boardId}`);
    });

    socket.on("leave-board", (boardId) => {
      socket.leave(`board:${boardId}`);
    });

    socket.on("typing", ({ boardId, userName }) => {
      socket
        .to(`board:${boardId}`)
        .emit("user-typing", {
          userName
        });
    });

    socket.on(
      "stop-typing",
      ({ boardId, userName }) => {
        socket
          .to(`board:${boardId}`)
          .emit(
            "user-stopped-typing",
            { userName }
          );
      }
    );

    socket.on("disconnect", () => {
      console.log(
        `User disconnected: ${socket.userId}`
      );
    });
  });
};

module.exports = setupSocket;
5. Update Frontend Socket

Open:

client/src/services/socket.js

Change it to:

import { io } from "socket.io-client";

const socket = io("http://localhost:5000", {
  autoConnect: false,
  auth: {
    token: localStorage.getItem("token")
  }
});

export default socket;

Now the Socket.io connection sends the JWT.

DAY 27 — Performance & UI Polish

Now improve the application's user experience.

1. Add Loading State

In Board.jsx:

const [loading, setLoading] = useState(true);

Update loading function:

const loadBoard = async () => {
  try {
    setLoading(true);

    const listsResponse = await api.get(
      `/lists/board/${boardId}`
    );

    setLists(listsResponse.data);

    const cardData = {};

    for (const list of listsResponse.data) {
      const response = await api.get(
        `/cards/list/${list._id}`
      );

      cardData[list._id] = response.data;
    }

    setCards(cardData);
  } catch (error) {
    console.error(
      "Failed to load board:",
      error
    );
  } finally {
    setLoading(false);
  }
};

Before rendering:

if (loading) {
  return (
    <div className="p-6">
      Loading board...
    </div>
  );
}
2. Error Message

Add:

const [error, setError] = useState("");

In catch:

setError("Unable to load board");

Display:

{error && (
  <div className="bg-red-100 text-red-700 p-3">
    {error}
  </div>
)}
3. MongoDB Indexes

For frequently searched fields, add indexes.

In Card.js:

cardSchema.index({
  title: "text",
  description: "text"
});

In Notification.js:

notificationSchema.index({
  user: 1,
  createdAt: -1
});

In List.js:

listSchema.index({
  board: 1,
  position: 1
});

These improve database query performance.

4. Debounced Search

Instead of sending an API request for every keystroke, use a small delay.

Install:

cd client
npm install lodash

Then:

import { debounce } from "lodash";

Example:

const searchCards = debounce(async (value) => {
  if (!value.trim()) {
    setResults([]);
    return;
  }

  try {
    const response = await api.get(
      `/cards/search/${boardId}?q=${encodeURIComponent(value)}`
    );

    setResults(response.data);
  } catch (error) {
    console.error(error);
  }
}, 400);

This reduces unnecessary API requests.

5. Suggested Final UI

Your application can now have:

--------------------------------------------------
| Collaborative Workspace       🔍 Search   🔔   |
--------------------------------------------------
| Sidebar       |             Board               |
|               |                                 |
| Workspace     |  TODO       | IN PROGRESS      |
|               |             |                  |
| Board 1       |  Task 1     | Task 3           |
| Board 2       |  Task 2     | Task 4           |
|               |             |                  |
| Members       |  DONE                          |
| Settings      |  Task 5                         |
--------------------------------------------------
DAY 28 — Deployment & Final Testing

Your project is now ready for deployment.

Production Architecture
             React Frontend
                  |
                  ↓
            Vercel / Netlify
                  |
                  ↓
        Node.js + Express Server
                  |
          ┌───────┼────────┐
          ↓       ↓        ↓
      MongoDB   Redis   Socket.io
       Atlas    Cloud     Real-time
1. Production Environment Variables

Backend .env:

PORT=5000

MONGO_URI=your_mongodb_atlas_connection

JWT_SECRET=your_strong_secret

CLIENT_URL=https://your-frontend-url.com

REDIS_URL=your_redis_connection_url

Do not upload .env to GitHub.

2. .gitignore

Create:

server/.gitignore
node_modules/
.env

Frontend:

client/.gitignore
node_modules/
dist/
.env
3. Frontend Production API

Instead of hardcoding:

baseURL: "http://localhost:5000/api"

use:

const api = axios.create({
  baseURL:
    import.meta.env.VITE_API_URL ||
    "http://localhost:5000/api"
});

Create:

client/.env
VITE_API_URL=https://your-backend-url.com/api
4. Production Socket URL

Update:

client/src/services/socket.js
import { io } from "socket.io-client";

const socket = io(
  import.meta.env.VITE_SOCKET_URL ||
    "http://localhost:5000",
  {
    autoConnect: false,
    auth: {
      token: localStorage.getItem("token")
    }
  }
);

export default socket;

Frontend .env:

VITE_SOCKET_URL=https://your-backend-url.com
5. Final Testing Checklist

Before submitting the project, test everything.

Authentication
[ ] Register
[ ] Login
[ ] JWT generated
[ ] Protected routes
[ ] Invalid token rejected
Workspace
[ ] Create workspace
[ ] View workspace
[ ] Workspace members
Boards
[ ] Create board
[ ] View board
[ ] Update board
[ ] Delete board
Lists
[ ] Create list
[ ] Update list
[ ] Delete list
Cards
[ ] Create card
[ ] Update card
[ ] Delete card
[ ] Move card
[ ] Assign card
Real-Time
[ ] Board rooms
[ ] Card creation
[ ] Card movement
[ ] Comments
[ ] Typing indicator
Week 4
[ ] Search
[ ] Notifications
[ ] Redis
[ ] Redis caching
[ ] Security
[ ] Rate limiting
[ ] Socket authentication
[ ] Loading states
[ ] Error handling
[ ] Responsive UI
Final Project Structure

By the end of Week 4, your project should look approximately like this:

collaborative-workspace/
│
├── client/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Card.jsx
│   │   │   ├── List.jsx
│   │   │   ├── Comments.jsx
│   │   │   ├── SearchBar.jsx
│   │   │   ├── Notifications.jsx
│   │   │   └── TypingIndicator.jsx
│   │   │
│   │   ├── pages/
│   │   ├── context/
│   │   ├── services/
│   │   │   ├── api.js
│   │   │   └── socket.js
│   │   │
│   │   ├── hooks/
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   └── index.css
│   │
│   ├── .env
│   └── package.json
│
├── server/
│   ├── config/
│   │   ├── db.js
│   │   └── redis.js
│   │
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── workspaceController.js
│   │   ├── boardController.js
│   │   ├── listController.js
│   │   ├── cardController.js
│   │   └── notificationController.js
│   │
│   ├── middleware/
│   │   └── authMiddleware.js
│   │
│   ├── models/
│   │   ├── User.js
│   │   ├── Workspace.js
│   │   ├── Board.js
│   │   ├── List.js
│   │   ├── Card.js
│   │   ├── Comment.js
│   │   └── Notification.js
│   │
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── workspaceRoutes.js
│   │   ├── boardRoutes.js
│   │   ├── listRoutes.js
│   │   ├── cardRoutes.js
│   │   ├── commentRoutes.js
│   │   └── notificationRoutes.js
│   │
│   ├── services/
│   │   └── cacheService.js
│   │
│   ├── socket/
│   │   ├── socketHandler.js
│   │   └── io.js
│   │
│   ├── .env
│   ├── .gitignore
│   ├── server.js
│   └── package.json
│
└── README.md
🎯 Final Outcome

After completing all 4 weeks, your project will demonstrate:

MERN Stack
→ MongoDB + Express + React + Node.js

Authentication
→ JWT + protected routes

Agile Management
→ Workspaces + Boards + Lists + Cards

Kanban
→ Drag-and-drop task management

Real-Time Collaboration
→ Socket.io + board rooms + comments + typing indicators

Search
→ Task/card search

Notifications
→ Real-time user notifications

Caching
→ Redis

Security
→ JWT + Helmet + rate limiting + protected WebSockets

Performance
→ MongoDB indexes + caching + debounced search

Deployment
→ Production frontend + backend + MongoDB + Redis

# web-project-2
Real-Time Collaborative Workspace (Agile Management Tool)
1. Create the project folder

Open VS Code → Terminal and run:

mkdir collaborative-workspace
cd collaborative-workspace

Create frontend:

npm create vite@latest client -- --template react

Then:

cd client
npm install
cd ..
2. Create the backend
mkdir server
cd server
npm init -y

Install backend packages:

npm install express mongoose dotenv cors bcryptjs jsonwebtoken socket.io redis

Install development tool:

npm install --save-dev nodemon
3. Backend folder structure

Inside server, create:

server/
│
├── config/
├── controllers/
├── middleware/
├── models/
├── routes/
├── socket/
├── services/
├── .env
├── server.js
└── package.json
4. Create server.js
const express = require("express");
const cors = require("cors");
const dotenv = require("dotenv");

dotenv.config();

const app = express();

app.use(cors());
app.use(express.json());

app.get("/", (req, res) => {
  res.json({
    message: "Collaborative Workspace API is running"
  });
});

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
5. Create .env

Inside the server folder:

PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/collaborative_workspace
JWT_SECRET=your_super_secret_key

Later, you can replace the MongoDB URL with a MongoDB Atlas connection string.

6. Update package.json

Inside server/package.json, change the scripts to:

"scripts": {
  "start": "node server.js",
  "dev": "nodemon server.js"
}
7. Start the backend

From the server folder:

npm run dev

You should see:

Server running on port 5000

Open your browser and visit:

http://localhost:5000

You should get:

{
  "message": "Collaborative Workspace API is running"
}




Day 2 — MongoDB Connection + User & Workspace Models

Today we'll connect MongoDB and create the first two database models.

1. Install MongoDB package

You already installed mongoose on Day 1, so no additional package is needed.

Inside server, create:

server/
├── config/
│   └── db.js
├── models/
│   ├── User.js
│   └── Workspace.js
├── server.js
└── .env
2. Create config/db.js
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log("MongoDB connected successfully");
  } catch (error) {
    console.error("MongoDB connection failed:", error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
3. Update server.js

Replace your current server.js with:

const express = require("express");
const cors = require("cors");
const dotenv = require("dotenv");
const connectDB = require("./config/db");

dotenv.config();

connectDB();

const app = express();

app.use(cors());
app.use(express.json());

app.get("/", (req, res) => {
  res.json({
    message: "Collaborative Workspace API is running"
  });
});

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
4. Create the User model

Create:

server/models/User.js

Add:

const mongoose = require("mongoose");

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true
    },

    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true
    },

    password: {
      type: String,
      required: true
    },

    avatar: {
      type: String,
      default: ""
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("User", userSchema);

This stores:

User
 ├── name
 ├── email
 ├── password
 ├── avatar
 ├── createdAt
 └── updatedAt
5. Create the Workspace model

Create:

server/models/Workspace.js

Add:

const mongoose = require("mongoose");

const workspaceSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true
    },

    owner: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    },

    members: [
      {
        type: mongoose.Schema.Types.ObjectId,
        ref: "User"
      }
    ]
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("Workspace", workspaceSchema);

The relationship will look like:

User
  │
  └── owns ──> Workspace
                   │
                   └── members ──> Users
6. Start the server

Make sure MongoDB is running, then:

cd server
npm run dev

You should see:

MongoDB connected successfully
Server running on port 5000

If you are using MongoDB Atlas instead of local MongoDB, put your Atlas connection string in .env:

MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/collaborative_workspace

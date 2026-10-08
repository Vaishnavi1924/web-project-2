
We already installed:

npm install socket.io

Now we'll create a proper Socket.io server.

1. Update server.js

Replace your current server.js with:

const express = require("express");
const cors = require("cors");
const dotenv = require("dotenv");
const http = require("http");

const { Server } = require("socket.io");

const connectDB = require("./config/db");

const authRoutes = require("./routes/authRoutes");
const workspaceRoutes = require("./routes/workspaceRoutes");
const boardRoutes = require("./routes/boardRoutes");
const listRoutes = require("./routes/listRoutes");
const cardRoutes = require("./routes/cardRoutes");

dotenv.config();

connectDB();

const app = express();

const server = http.createServer(app);

const io = new Server(server, {
  cors: {
    origin: process.env.CLIENT_URL || "http://localhost:5173",
    methods: ["GET", "POST", "PUT", "DELETE"]
  }
});

app.use(cors());

app.use(express.json());

app.use("/api/auth", authRoutes);

app.use("/api/workspaces", workspaceRoutes);

app.use("/api/boards", boardRoutes);

app.use("/api/lists", listRoutes);

app.use("/api/cards", cardRoutes);

app.get("/", (req, res) => {
  res.json({
    message: "Collaborative Workspace API is running"
  });
});

io.on("connection", (socket) => {

  console.log(
    `Socket connected: ${socket.id}`
  );

  socket.on("disconnect", () => {

    console.log(
      `Socket disconnected: ${socket.id}`
    );

  });

});

const PORT = process.env.PORT || 5000;

server.listen(PORT, () => {

  console.log(
    `Server running on port ${PORT}`
  );

});
Important

Previously we used:

app.listen(...)

Now we use:

server.listen(...)

because Socket.io needs the HTTP server.

DAY 16 — Board Rooms

We don't want every user receiving updates from every board.

For example:

Board A
 ├── Vaishnavi
 └── User B

Board B
 ├── User C
 └── User D

Users in Board A should only receive Board A updates.

Socket.io rooms solve this.

1. Create socket folder

Create:

server/socket/socketHandler.js
const setupSocket = (io) => {

  io.on("connection", (socket) => {

    console.log(
      `User connected: ${socket.id}`
    );

    // Join board
    socket.on("join-board", (boardId) => {

      socket.join(`board:${boardId}`);

      console.log(
        `${socket.id} joined board:${boardId}`
      );

    });


    // Leave board
    socket.on("leave-board", (boardId) => {

      socket.leave(`board:${boardId}`);

      console.log(
        `${socket.id} left board:${boardId}`
      );

    });


    socket.on("disconnect", () => {

      console.log(
        `User disconnected: ${socket.id}`
      );

    });

  });

};

module.exports = setupSocket;
2. Update server.js

Remove this:

io.on("connection", (socket) => {

  console.log(
    `Socket connected: ${socket.id}`
  );

  socket.on("disconnect", () => {

    console.log(
      `Socket disconnected: ${socket.id}`
    );

  });

});

Add:

const setupSocket = require("./socket/socketHandler");

setupSocket(io);

So the relevant part becomes:

const setupSocket = require("./socket/socketHandler");

setupSocket(io);
3. Frontend Socket.io client

You already installed:

npm install socket.io-client

Create:

client/src/services/socket.js
import { io } from "socket.io-client";

const socket = io("http://localhost:5000", {
  autoConnect: false
});

export default socket;
4. Join board from React

Update:

client/src/pages/Board.jsx

Add:

import { useEffect } from "react";
import socket from "../services/socket";

Inside the component:

useEffect(() => {

  socket.connect();

  socket.emit(
    "join-board",
    boardId
  );

  return () => {

    socket.emit(
      "leave-board",
      boardId
    );

    socket.disconnect();

  };

}, [boardId]);

Now when a user opens a board:

React
 ↓
Socket.io
 ↓
join-board
 ↓
board:12345
DAY 17 — Real-Time Card Creation & Updates

Now we will broadcast card changes.

1. Card creation event

In:

server/controllers/cardController.js

We need access to Socket.io.

The cleanest approach is to create:

server/socket/io.js
let io;

const setIO = (socketIO) => {
  io = socketIO;
};

const getIO = () => {
  if (!io) {
    throw new Error("Socket.io not initialized");
  }

  return io;
};

module.exports = {
  setIO,
  getIO
};
2. Update server.js

Add:

const { setIO } = require("./socket/io");

After creating io:

const io = new Server(server, {
  cors: {
    origin: process.env.CLIENT_URL || "http://localhost:5173",
    methods: ["GET", "POST", "PUT", "DELETE"]
  }
});

setIO(io);
3. Broadcast card creation

At the top of:

cardController.js

add:

const { getIO } = require("../socket/io");

Inside createCard, after:

const card = await Card.create({
  title,
  description,
  list: listId,
  position
});

get the board:

const board = access.board;

Then:

getIO()
  .to(`board:${board._id}`)
  .emit("card-created", {
    card
  });

So the final section becomes:

const card = await Card.create({
  title,
  description,
  list: listId,
  position
});

getIO()
  .to(`board:${access.board._id}`)
  .emit("card-created", {
    card
  });

res.status(201).json({
  message: "Card created",
  card
});
4. Receive card-created event

In Board.jsx:

useEffect(() => {

  socket.connect();

  socket.emit(
    "join-board",
    boardId
  );

  const handleCardCreated = ({ card }) => {

    setCards((previous) => {

      const updated = {
        ...previous
      };

      if (!updated[card.list]) {
        updated[card.list] = [];
      }

      updated[card.list] = [
        ...updated[card.list],
        card
      ];

      return updated;

    });

  };

  socket.on(
    "card-created",
    handleCardCreated
  );

  return () => {

    socket.off(
      "card-created",
      handleCardCreated
    );

    socket.emit(
      "leave-board",
      boardId
    );

    socket.disconnect();

  };

}, [boardId]);

Now:

User A creates card
       ↓
MongoDB
       ↓
Socket.io
       ↓
board room
       ↓
User B
       ↓
Card appears automatically
DAY 18 — Real-Time Drag & Drop

This is the most important Socket.io feature.

Currently Week 2 does:

Drag
 ↓
Update local UI
 ↓
PUT API
 ↓
MongoDB

Now we'll add:

Drag
 ↓
Update local UI
 ↓
MongoDB
 ↓
Socket.io
 ↓
Other users
1. Update moveCard

In cardController.js, find:

await card.save();

Immediately after it add:

getIO()
  .to(`board:${newAccess.board._id}`)
  .emit("card-moved", {
    cardId: card._id,
    listId: card.list,
    position: card.position
  });

Your final section:

card.list = listId;
card.position = position;

await card.save();

getIO()
  .to(`board:${newAccess.board._id}`)
  .emit("card-moved", {
    cardId: card._id,
    listId: card.list,
    position: card.position
  });

res.json({
  message: "Card moved",
  card
});
2. Listen for card movement

In Board.jsx:

useEffect(() => {

  const handleCardMoved = ({
    cardId,
    listId,
    position
  }) => {

    setCards((previous) => {

      const updated = {};

      Object.keys(previous).forEach(
        (key) => {

          updated[key] = [
            ...previous[key]
          ];

        }
      );

      let movedCard = null;

      Object.keys(updated).forEach(
        (key) => {

          const index = updated[key].findIndex(
            (card) => card._id === cardId
          );

          if (index !== -1) {

            movedCard =
              updated[key][index];

            updated[key].splice(
              index,
              1
            );

          }

        }
      );

      if (!movedCard) {
        return previous;
      }

      movedCard.list = listId;

      updated[listId].splice(
        position,
        0,
        movedCard
      );

      return updated;

    });

  };

  socket.on(
    "card-moved",
    handleCardMoved
  );

  return () => {

    socket.off(
      "card-moved",
      handleCardMoved
    );

  };

}, []);

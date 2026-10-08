DAY 19 — Card Comments

Now we'll add comments.

The structure becomes:

Card
 ├── title
 ├── description
 └── comments
      ├── User A: "Working on this"
      └── User B: "Looks good"
1. Comment model

Create:

server/models/Comment.js
const mongoose = require("mongoose");

const commentSchema = new mongoose.Schema(
  {
    text: {
      type: String,
      required: true,
      trim: true
    },

    card: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Card",
      required: true
    },

    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    }
  },
  {
    timestamps: true
  }
);

module.exports =
  mongoose.model("Comment", commentSchema);
2. Comment controller

Create:

server/controllers/commentController.js
const Comment = require("../models/Comment");
const Card = require("../models/Card");
const List = require("../models/List");
const Board = require("../models/Board");
const Workspace = require("../models/Workspace");

const { getIO } = require("../socket/io");


const createComment = async (req, res) => {

  try {

    const { text } = req.body;

    const card = await Card.findById(
      req.params.cardId
    );

    if (!card) {
      return res.status(404).json({
        message: "Card not found"
      });
    }

    const list = await List.findById(
      card.list
    );

    const board = await Board.findById(
      list.board
    );

    const workspace =
      await Workspace.findOne({
        _id: board.workspace,
        members: req.user._id
      });

    if (!workspace) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const comment = await Comment.create({
      text,
      card: card._id,
      user: req.user._id
    });

    await comment.populate(
      "user",
      "name email"
    );

    getIO()
      .to(`board:${board._id}`)
      .emit(
        "comment-created",
        comment
      );

    res.status(201).json(comment);

  } catch (error) {

    res.status(500).json({
      message: "Server error",
      error: error.message
    });

  }

};


const getComments = async (req, res) => {

  try {

    const comments =
      await Comment.find({
        card: req.params.cardId
      })
        .populate(
          "user",
          "name email"
        )
        .sort({
          createdAt: 1
        });

    res.json(comments);

  } catch (error) {

    res.status(500).json({
      message: "Server error",
      error: error.message
    });

  }

};


module.exports = {
  createComment,
  getComments
};
3. Comment routes

Create:

server/routes/commentRoutes.js
const express = require("express");

const protect =
  require("../middleware/authMiddleware");

const {
  createComment,
  getComments
} = require("../controllers/commentController");

const router = express.Router();

router.post(
  "/card/:cardId",
  protect,
  createComment
);

router.get(
  "/card/:cardId",
  protect,
  getComments
);

module.exports = router;

Add to server.js:

const commentRoutes =
  require("./routes/commentRoutes");

app.use(
  "/api/comments",
  commentRoutes
);
4. Comment frontend

Create:

client/src/components/Comments.jsx
import { useEffect, useState } from "react";

import api from "../services/api";
import socket from "../services/socket";

function Comments({ cardId }) {

  const [comments, setComments] =
    useState([]);

  const [text, setText] =
    useState("");

  useEffect(() => {

    loadComments();

    const handleComment = (comment) => {

      if (
        comment.card === cardId ||
        comment.card?._id === cardId
      ) {

        setComments(
          (previous) => [
            ...previous,
            comment
          ]
        );

      }

    };

    socket.on(
      "comment-created",
      handleComment
    );

    return () => {

      socket.off(
        "comment-created",
        handleComment
      );

    };

  }, [cardId]);


  const loadComments = async () => {

    const response =
      await api.get(
        `/comments/card/${cardId}`
      );

    setComments(response.data);

  };


  const addComment = async (e) => {

    e.preventDefault();

    if (!text.trim()) {
      return;
    }

    await api.post(
      `/comments/card/${cardId}`,
      {
        text
      }
    );

    setText("");

  };


  return (
    <div>

      <h3>Comments</h3>

      <div>

        {comments.map((comment) => (

          <div
            key={comment._id}
            style={{
              padding: "10px",
              borderBottom:
                "1px solid #ddd"
            }}
          >

            <strong>
              {comment.user?.name}
            </strong>

            <p>
              {comment.text}
            </p>

          </div>

        ))}

      </div>


      <form onSubmit={addComment}>

        <input
          value={text}
          onChange={(e) =>
            setText(e.target.value)
          }
          placeholder="Write a comment..."
        />

        <button type="submit">
          Send
        </button>

      </form>

    </div>
  );
}

export default Comments;
DAY 20 — Typing Indicators

Now we'll implement:

Vaishnavi is typing...

This doesn't need MongoDB because typing status is temporary.

1. Server typing events

Update:

server/socket/socketHandler.js

Use:

const setupSocket = (io) => {

  io.on("connection", (socket) => {

    console.log(
      `User connected: ${socket.id}`
    );


    socket.on(
      "join-board",
      (boardId) => {

        socket.join(
          `board:${boardId}`
        );

      }
    );


    socket.on(
      "leave-board",
      (boardId) => {

        socket.leave(
          `board:${boardId}`
        );

      }
    );


    socket.on(
      "typing",
      ({
        boardId,
        userName
      }) => {

        socket
          .to(`board:${boardId}`)
          .emit(
            "user-typing",
            {
              userName
            }
          );

      }
    );


    socket.on(
      "stop-typing",
      ({
        boardId,
        userName
      }) => {

        socket
          .to(`board:${boardId}`)
          .emit(
            "user-stopped-typing",
            {
              userName
            }
          );

      }
    );


    socket.on(
      "disconnect",
      () => {

        console.log(
          `User disconnected: ${socket.id}`
        );

      }
    );

  });

};

module.exports = setupSocket;
2. Typing component

Create:

client/src/components/TypingIndicator.jsx
import { useEffect, useState } from "react";

import socket from "../services/socket";

function TypingIndicator() {

  const [typingUser, setTypingUser] =
    useState("");

  useEffect(() => {

    const handleTyping =
      ({ userName }) => {

        setTypingUser(
          `${userName} is typing...`
        );

      };


    const handleStoppedTyping =
      () => {

        setTypingUser("");

      };


    socket.on(
      "user-typing",
      handleTyping
    );

    socket.on(
      "user-stopped-typing",
      handleStoppedTyping
    );


    return () => {

      socket.off(
        "user-typing",
        handleTyping
      );

      socket.off(
        "user-stopped-typing",
        handleStoppedTyping
      );

    };

  }, []);


  return (
    <div
      style={{
        height: "25px",
        fontSize: "14px",
        color: "#666"
      }}
    >
      {typingUser}
    </div>
  );

}

export default TypingIndicator;
3. Using typing events

For example, in a comment input:

const handleTyping = () => {

  socket.emit(
    "typing",
    {
      boardId,
      userName: user.name
    }
  );

};

When input stops:

const handleStopTyping = () => {

  socket.emit(
    "stop-typing",
    {
      boardId,
      userName: user.name
    }
  );

};

You can connect these to:

<input
  onChange={handleTyping}
  onBlur={handleStopTyping}
/>
DAY 21 — Complete Real-Time Testing

Now test with two browser windows.

For example:

Chrome Window 1
     ↓
User A

Chrome Window 2
     ↓
User B

Both users should open the same board.

Test 1 — Board room

Open:

Board A

in both browsers.

Terminal should show users joining:

User connected
User joined board:xxxx
Test 2 — Card creation

User A:

Create card

User B should immediately see:

New card

without refreshing.

Test 3 — Card movement

User A:

To Do
 ↓
In Progress

User B should immediately see:

To Do
 ↓
In Progress
Test 4 — Comments

User A:

"Backend API is ready."

User B should immediately see the comment.

Test 5 — Typing indicator

User A starts typing:

Vaishnavi is typing...

User B should see it.

When User A stops:

Vaishnavi is typing...

should disappear.

Week 3 Final Architecture

After Week 3, your backend should look like:

server/
│
├── config/
│   └── db.js
│
├── controllers/
│   ├── authController.js
│   ├── workspaceController.js
│   ├── boardController.js
│   ├── listController.js
│   ├── cardController.js
│   └── commentController.js
│
├── middleware/
│   └── authMiddleware.js
│
├── models/
│   ├── User.js
│   ├── Workspace.js
│   ├── Board.js
│   ├── List.js
│   ├── Card.js
│   └── Comment.js
│
├── routes/
│   ├── authRoutes.js
│   ├── workspaceRoutes.js
│   ├── boardRoutes.js
│   ├── listRoutes.js
│   ├── cardRoutes.js
│   └── commentRoutes.js
│
├── socket/
│   ├── io.js
│   └── socketHandler.js
│
├── server.js
└── .env

Frontend:

client/src/
│
├── components/
│   ├── Card.jsx
│   ├── List.jsx
│   ├── Comments.jsx
│   └── TypingIndicator.jsx
│
├── pages/
│   └── Board.jsx
│
├── services/
│   ├── api.js
│   └── socket.js
│
├── App.jsx
└── index.css

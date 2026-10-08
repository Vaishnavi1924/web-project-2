DAY 10 — Cards CRUD

Now we'll create the most important object:

Card

Example:

To Do
 ├── Create Login Page
 ├── Create Dashboard
 └── Connect MongoDB
1. Card model

Create:

server/models/Card.js
const mongoose = require("mongoose");

const cardSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: true,
      trim: true
    },

    description: {
      type: String,
      default: ""
    },

    list: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "List",
      required: true
    },

    position: {
      type: Number,
      default: 0
    },

    assignee: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      default: null
    },

    labels: [
      {
        type: String
      }
    ]
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("Card", cardSchema);
2. Card controller

Create:

server/controllers/cardController.js
const Card = require("../models/Card");
const List = require("../models/List");
const Board = require("../models/Board");
const Workspace = require("../models/Workspace");

const checkListAccess = async (listId, userId) => {
  const list = await List.findById(listId);

  if (!list) {
    return null;
  }

  const board = await Board.findById(list.board);

  if (!board) {
    return null;
  }

  const workspace = await Workspace.findOne({
    _id: board.workspace,
    members: userId
  });

  if (!workspace) {
    return null;
  }

  return {
    list,
    board
  };
};


// Create Card
const createCard = async (req, res) => {
  try {
    const {
      title,
      description,
      listId
    } = req.body;

    if (!title || !listId) {
      return res.status(400).json({
        message: "Title and list are required"
      });
    }

    const access = await checkListAccess(
      listId,
      req.user._id
    );

    if (!access) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const lastCard = await Card.findOne({
      list: listId
    }).sort({ position: -1 });

    const position = lastCard
      ? lastCard.position + 1
      : 0;

    const card = await Card.create({
      title,
      description,
      list: listId,
      position
    });

    res.status(201).json({
      message: "Card created",
      card
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Get Cards
const getCards = async (req, res) => {
  try {
    const { listId } = req.params;

    const access = await checkListAccess(
      listId,
      req.user._id
    );

    if (!access) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const cards = await Card.find({
      list: listId
    })
      .populate("assignee", "name email")
      .sort({ position: 1 });

    res.json(cards);

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Update Card
const updateCard = async (req, res) => {
  try {
    const card = await Card.findById(req.params.id);

    if (!card) {
      return res.status(404).json({
        message: "Card not found"
      });
    }

    const access = await checkListAccess(
      card.list,
      req.user._id
    );

    if (!access) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const {
      title,
      description,
      assignee,
      labels
    } = req.body;

    if (title !== undefined) {
      card.title = title;
    }

    if (description !== undefined) {
      card.description = description;
    }

    if (assignee !== undefined) {
      card.assignee = assignee;
    }

    if (labels !== undefined) {
      card.labels = labels;
    }

    await card.save();

    res.json({
      message: "Card updated",
      card
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Delete Card
const deleteCard = async (req, res) => {
  try {
    const card = await Card.findById(req.params.id);

    if (!card) {
      return res.status(404).json({
        message: "Card not found"
      });
    }

    const access = await checkListAccess(
      card.list,
      req.user._id
    );

    if (!access) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    await card.deleteOne();

    res.json({
      message: "Card deleted"
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Move Card
const moveCard = async (req, res) => {
  try {
    const {
      listId,
      position
    } = req.body;

    const card = await Card.findById(req.params.id);

    if (!card) {
      return res.status(404).json({
        message: "Card not found"
      });
    }

    const oldAccess = await checkListAccess(
      card.list,
      req.user._id
    );

    if (!oldAccess) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const newAccess = await checkListAccess(
      listId,
      req.user._id
    );

    if (!newAccess) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    card.list = listId;
    card.position = position;

    await card.save();

    res.json({
      message: "Card moved",
      card
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


module.exports = {
  createCard,
  getCards,
  updateCard,
  deleteCard,
  moveCard
};
3. Card routes

Create:

server/routes/cardRoutes.js
const express = require("express");

const protect = require("../middleware/authMiddleware");

const {
  createCard,
  getCards,
  updateCard,
  deleteCard,
  moveCard
} = require("../controllers/cardController");

const router = express.Router();

router.post("/", protect, createCard);

router.get(
  "/list/:listId",
  protect,
  getCards
);

router.put("/:id", protect, updateCard);

router.delete("/:id", protect, deleteCard);

router.put(
  "/:id/move",
  protect,
  moveCard
);

module.exports = router;

Add to server.js:

const cardRoutes = require("./routes/cardRoutes");

app.use("/api/cards", cardRoutes);
DAY 11 — Kanban Board Frontend

Now we'll create the actual Kanban UI.

Install the drag-and-drop library:

cd client
npm install @hello-pangea/dnd

@hello-pangea/dnd is a maintained alternative to the older react-beautiful-dnd package.

1. API service

Create:

client/src/services/api.js
import axios from "axios";

const api = axios.create({
  baseURL: "http://localhost:5000/api"
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");

  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});

export default api;
2. Card component

Create:

client/src/components/Card.jsx
import { Draggable } from "@hello-pangea/dnd";

function Card({ card, index }) {
  return (
    <Draggable
      draggableId={card._id}
      index={index}
    >
      {(provided) => (
        <div
          ref={provided.innerRef}
          {...provided.draggableProps}
          {...provided.dragHandleProps}
          style={{
            background: "white",
            padding: "15px",
            marginBottom: "10px",
            borderRadius: "8px",
            boxShadow: "0 2px 5px rgba(0,0,0,0.1)",
            ...provided.draggableProps.style
          }}
        >
          <h4>{card.title}</h4>

          {card.description && (
            <p>{card.description}</p>
          )}
        </div>
      )}
    </Draggable>
  );
}

export default Card;
3. List component

Create:

client/src/components/List.jsx
import { Droppable } from "@hello-pangea/dnd";
import Card from "./Card";

function List({ list, cards }) {
  return (
    <div
      style={{
        width: "280px",
        background: "#e8ebf0",
        padding: "15px",
        borderRadius: "10px"
      }}
    >
      <h3>{list.title}</h3>

      <Droppable droppableId={list._id}>
        {(provided) => (
          <div
            ref={provided.innerRef}
            {...provided.droppableProps}
            style={{
              minHeight: "100px",
              marginTop: "15px"
            }}
          >
            {cards.map((card, index) => (
              <Card
                key={card._id}
                card={card}
                index={index}
              />
            ))}

            {provided.placeholder}
          </div>
        )}
      </Droppable>
    </div>
  );
}

export default List;
DAY 12 — Drag & Drop

Create:

client/src/pages/Board.jsx
import { useEffect, useState } from "react";
import {
  DragDropContext
} from "@hello-pangea/dnd";

import api from "../services/api";
import List from "../components/List";

function Board({ boardId }) {
  const [lists, setLists] = useState([]);
  const [cards, setCards] = useState({});

  useEffect(() => {
    loadBoard();
  }, [boardId]);

  const loadBoard = async () => {
    try {
      const listResponse = await api.get(
        `/lists/board/${boardId}`
      );

      const boardLists = listResponse.data;

      setLists(boardLists);

      const cardData = {};

      for (const list of boardLists) {
        const response = await api.get(
          `/cards/list/${list._id}`
        );

        cardData[list._id] = response.data;
      }

      setCards(cardData);

    } catch (error) {
      console.error(error);
    }
  };

  const handleDragEnd = async (result) => {
    const {
      source,
      destination,
      draggableId
    } = result;

    if (!destination) {
      return;
    }

    if (
      source.droppableId === destination.droppableId &&
      source.index === destination.index
    ) {
      return;
    }

    const sourceList = [
      ...(cards[source.droppableId] || [])
    ];

    const destinationList =
      source.droppableId === destination.droppableId
        ? sourceList
        : [
            ...(cards[destination.droppableId] || [])
          ];

    const [movedCard] = sourceList.splice(
      source.index,
      1
    );

    destinationList.splice(
      destination.index,
      0,
      movedCard
    );

    const updatedCards = {
      ...cards
    };

    updatedCards[source.droppableId] = sourceList;

    updatedCards[destination.droppableId] =
      destinationList;

    setCards(updatedCards);

    try {
      await api.put(
        `/cards/${draggableId}/move`,
        {
          listId: destination.droppableId,
          position: destination.index
        }
      );

    } catch (error) {
      console.error("Move failed:", error);

      loadBoard();
    }
  };

  return (
    <div
      style={{
        padding: "30px"
      }}
    >
      <h1>Kanban Board</h1>

      <DragDropContext
        onDragEnd={handleDragEnd}
      >
        <div
          style={{
            display: "flex",
            gap: "20px",
            overflowX: "auto",
            marginTop: "30px"
          }}
        >
          {lists.map((list) => (
            <List
              key={list._id}
              list={list}
              cards={cards[list._id] || []}
            />
          ))}
        </div>
      </DragDropContext>
    </div>
  );
}

export default Board;
DAY 13 — Optimistic UI

This is an important part of your project.

Without optimistic updates:
Drag card
    ↓
Wait for server
    ↓
Server response
    ↓
Update screen

The user experiences delay.

With optimistic updates:
Drag card
    ↓
Update UI immediately
    ↓
Send request to server
    ↓
Server confirms

Our handleDragEnd() already does this:

setCards(updatedCards);

await api.put(
  `/cards/${draggableId}/move`,
  {
    listId: destination.droppableId,
    position: destination.index
  }
);

If the server fails:

catch (error) {
  loadBoard();
}

So the application automatically restores the correct server state.

DAY 14 — Week 2 Testing

Now test the complete Kanban engine.

Test 1 — Create Board
POST /api/boards

Example:

{
  "name": "MERN Project",
  "workspaceId": "..."
}
Test 2 — Create Lists
POST /api/lists
To Do
{
  "title": "To Do",
  "boardId": "..."
}
In Progress
{
  "title": "In Progress",
  "boardId": "..."
}
Done
{
  "title": "Done",
  "boardId": "..."
}
Test 3 — Create Cards
POST /api/cards

Example:

{
  "title": "Create Login Page",
  "description": "Build React login page",
  "listId": "..."
}

Create several cards.

Test 4 — Drag cards

Your UI should allow:

┌─────────────┐
│   TO DO     │
├─────────────┤
│ Login Page  │
│ Dashboard   │
└─────────────┘

        ↓ Drag

┌─────────────┐
│ IN PROGRESS │
├─────────────┤
│ Login Page  │
└─────────────┘

The card should move immediately, and MongoDB should then be updated.

Week 2 Final Architecture

At the end of Week 2, your backend becomes:

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
│   └── cardController.js
│
├── middleware/
│   └── authMiddleware.js
│
├── models/
│   ├── User.js
│   ├── Workspace.js
│   ├── Board.js
│   ├── List.js
│   └── Card.js
│
├── routes/
│   ├── authRoutes.js
│   ├── workspaceRoutes.js
│   ├── boardRoutes.js
│   ├── listRoutes.js
│   └── cardRoutes.js
│
├── server.js
└── .env

Frontend:

client/src/
│
├── components/
│   ├── Card.jsx
│   └── List.jsx
│
├── pages/
│   └── Board.jsx
│
├── services/
│   └── api.js
│
├── App.jsx
└── index.css

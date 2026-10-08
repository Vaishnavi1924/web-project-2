Workspace
   │
   └── Board
        │
        ├── To Do
        │    ├── Task 1
        │    └── Task 2
        │
        ├── In Progress
        │    └── Task 3
        │
        └── Done
             └── Task 4

Users will be able to:

Create boards
View boards
Create lists
Rename/delete lists
Create cards
Edit cards
Delete cards
Move cards
Drag cards between lists
Reorder cards
Update the UI immediately before the server responds
DAY 8 — Board Model + Board APIs
1. Create Board model

Create:

server/models/Board.js
const mongoose = require("mongoose");

const boardSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true
    },

    workspace: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Workspace",
      required: true
    },

    createdBy: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("Board", boardSchema);
2. Board controller

Create:

server/controllers/boardController.js
const Board = require("../models/Board");
const Workspace = require("../models/Workspace");

// Create Board
const createBoard = async (req, res) => {
  try {
    const { name, workspaceId } = req.body;

    if (!name || !workspaceId) {
      return res.status(400).json({
        message: "Board name and workspace are required"
      });
    }

    const workspace = await Workspace.findOne({
      _id: workspaceId,
      members: req.user._id
    });

    if (!workspace) {
      return res.status(403).json({
        message: "You are not a member of this workspace"
      });
    }

    const board = await Board.create({
      name,
      workspace: workspaceId,
      createdBy: req.user._id
    });

    res.status(201).json({
      message: "Board created successfully",
      board
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Get Boards
const getBoards = async (req, res) => {
  try {
    const { workspaceId } = req.params;

    const workspace = await Workspace.findOne({
      _id: workspaceId,
      members: req.user._id
    });

    if (!workspace) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const boards = await Board.find({
      workspace: workspaceId
    }).sort({ createdAt: -1 });

    res.json(boards);

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Get Single Board
const getBoard = async (req, res) => {
  try {
    const board = await Board.findById(req.params.id);

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

    res.json(board);

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Update Board
const updateBoard = async (req, res) => {
  try {
    const { name } = req.body;

    const board = await Board.findById(req.params.id);

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

    board.name = name || board.name;

    await board.save();

    res.json({
      message: "Board updated",
      board
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Delete Board
const deleteBoard = async (req, res) => {
  try {
    const board = await Board.findById(req.params.id);

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

    await board.deleteOne();

    res.json({
      message: "Board deleted successfully"
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


module.exports = {
  createBoard,
  getBoards,
  getBoard,
  updateBoard,
  deleteBoard
};
3. Board routes

Create:

server/routes/boardRoutes.js
const express = require("express");

const protect = require("../middleware/authMiddleware");

const {
  createBoard,
  getBoards,
  getBoard,
  updateBoard,
  deleteBoard
} = require("../controllers/boardController");

const router = express.Router();

router.post("/", protect, createBoard);

router.get(
  "/workspace/:workspaceId",
  protect,
  getBoards
);

router.get("/:id", protect, getBoard);

router.put("/:id", protect, updateBoard);

router.delete("/:id", protect, deleteBoard);

module.exports = router;
4. Add route to server.js

Add:

const boardRoutes = require("./routes/boardRoutes");

Then:

app.use("/api/boards", boardRoutes);

Your routes are now:

POST   /api/boards
GET    /api/boards/workspace/:workspaceId
GET    /api/boards/:id
PUT    /api/boards/:id
DELETE /api/boards/:id
5. Test Board creation

Using Postman:

POST
http://localhost:5000/api/boards

Headers:

Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

Body:

{
  "name": "Development Board",
  "workspaceId": "YOUR_WORKSPACE_ID"
}

Expected:

{
  "message": "Board created successfully",
  "board": {
    "_id": "...",
    "name": "Development Board"
  }
}
DAY 9 — Lists CRUD

A board contains multiple lists.

Example:

Board
│
├── To Do
├── In Progress
└── Done
1. Create List model

Create:

server/models/List.js
const mongoose = require("mongoose");

const listSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: true,
      trim: true
    },

    board: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Board",
      required: true
    },

    position: {
      type: Number,
      default: 0
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("List", listSchema);
2. List controller

Create:

server/controllers/listController.js
const List = require("../models/List");
const Board = require("../models/Board");
const Workspace = require("../models/Workspace");

const checkBoardAccess = async (boardId, userId) => {
  const board = await Board.findById(boardId);

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

  return board;
};


// Create List
const createList = async (req, res) => {
  try {
    const { title, boardId } = req.body;

    if (!title || !boardId) {
      return res.status(400).json({
        message: "Title and board are required"
      });
    }

    const board = await checkBoardAccess(
      boardId,
      req.user._id
    );

    if (!board) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const lastList = await List.findOne({
      board: boardId
    }).sort({ position: -1 });

    const position = lastList
      ? lastList.position + 1
      : 0;

    const list = await List.create({
      title,
      board: boardId,
      position
    });

    res.status(201).json({
      message: "List created",
      list
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Get Lists
const getLists = async (req, res) => {
  try {
    const { boardId } = req.params;

    const board = await checkBoardAccess(
      boardId,
      req.user._id
    );

    if (!board) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    const lists = await List.find({
      board: boardId
    }).sort({ position: 1 });

    res.json(lists);

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Update List
const updateList = async (req, res) => {
  try {
    const { title } = req.body;

    const list = await List.findById(req.params.id);

    if (!list) {
      return res.status(404).json({
        message: "List not found"
      });
    }

    const board = await checkBoardAccess(
      list.board,
      req.user._id
    );

    if (!board) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    list.title = title || list.title;

    await list.save();

    res.json({
      message: "List updated",
      list
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


// Delete List
const deleteList = async (req, res) => {
  try {
    const list = await List.findById(req.params.id);

    if (!list) {
      return res.status(404).json({
        message: "List not found"
      });
    }

    const board = await checkBoardAccess(
      list.board,
      req.user._id
    );

    if (!board) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    await list.deleteOne();

    res.json({
      message: "List deleted"
    });

  } catch (error) {
    res.status(500).json({
      message: "Server error",
      error: error.message
    });
  }
};


module.exports = {
  createList,
  getLists,
  updateList,
  deleteList
};
3. List routes

Create:

server/routes/listRoutes.js
const express = require("express");

const protect = require("../middleware/authMiddleware");

const {
  createList,
  getLists,
  updateList,
  deleteList
} = require("../controllers/listController");

const router = express.Router();

router.post("/", protect, createList);

router.get(
  "/board/:boardId",
  protect,
  getLists
);

router.put("/:id", protect, updateList);

router.delete("/:id", protect, deleteList);

module.exports = router;

Add to server.js:

const listRoutes = require("./routes/listRoutes");

app.use("/api/lists", listRoutes);

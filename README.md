const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const cors = require("cors");
const dotenv = require("dotenv");

dotenv.config();
const app = express();
app.use(express.json());
app.use(cors());

// MongoDB Connection
mongoose.connect(process.env.MONGO_URI)
    .then(() => console.log("✅ MongoDB Connected"))
    .catch(err => console.error("MongoDB Connection Error:", err));

// Models
const userSchema = new mongoose.Schema({
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
});

const incomeSchema = new mongoose.Schema({
    user_id: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    source: { type: String, required: true },
    amount: { type: Number, required: true },
    date: { type: Date, default: Date.now },
});

const expenseSchema = new mongoose.Schema({
    user_id: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    category: { type: String, required: true },
    amount: { type: Number, required: true },
    date: { type: Date, default: Date.now },
});

const savingsSchema = new mongoose.Schema({
    user_id: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    goal: { type: String, required: true },
    targetAmount: { type: Number, required: true },
    currentAmount: { type: Number, default: 0 },
    deadline: { type: Date, required: true },
});

const investmentSchema = new mongoose.Schema({
    user_id: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    type: { type: String, required: true },
    initialAmount: { type: Number, required: true },
    currentValue: { type: Number, required: true },
});

const goalSchema = new mongoose.Schema({
    user_id: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    goal: { type: String, required: true },
    targetAmount: { type: Number, required: true },
    currentAmount: { type: Number, default: 0 },
    deadline: { type: Date, required: true },
});

const User = mongoose.model("User", userSchema);
const Income = mongoose.model("Income", incomeSchema);
const Expense = mongoose.model("Expense", expenseSchema);
const Savings = mongoose.model("Savings", savingsSchema);
const Investment = mongoose.model("Investment", investmentSchema);
const Goal = mongoose.model("Goal", goalSchema);

// Middleware to verify JWT
const authMiddleware = (req, res, next) => {
    const token = req.header("Authorization")?.replace("Bearer ", "");
    if (!token) return res.status(401).json({ error: "Access denied. No token provided." });

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user_id = decoded.user_id;
        next();
    } catch (error) {
        res.status(400).json({ error: "Invalid token." });
    }
};

// Routes
// Register User
app.post("/api/register", async (req, res) => {
    const { name, email, password } = req.body;

    // Validate input
    if (!name || !email || !password) {
        return res.status(400).json({ error: "Please provide name, email, and password." });
    }

    try {
        // Check if user already exists
        const existingUser = await User.findOne({ email });
        if (existingUser) {
            return res.status(400).json({ error: "User already exists." });
        }

        // Hash password
        const hashedPassword = await bcrypt.hash(password, 10);

        // Create new user
        const user = new User({ name, email, password: hashedPassword });
        await user.save();

        res.status(201).json({ message: "User registered successfully" });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Login User
app.post("/api/login", async (req, res) => {
    const { email, password } = req.body;

    // Validate input
    if (!email || !password) {
        return res.status(400).json({ error: "Please provide email and password." });
    }

    try {
        // Find user by email
        const user = await User.findOne({ email });
        if (!user) {
            return res.status(400).json({ error: "Invalid email or password." });
        }

        // Check password
        const isPasswordValid = await bcrypt.compare(password, user.password);
        if (!isPasswordValid) {
            return res.status(400).json({ error: "Invalid email or password." });
        }

        // Generate JWT token
        const token = jwt.sign({ user_id: user._id }, process.env.JWT_SECRET);
        res.json({ token });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Add Income
app.post("/api/income/add", authMiddleware, async (req, res) => {
    const { source, amount } = req.body;

    // Validate input
    if (!source || !amount) {
        return res.status(400).json({ error: "Please provide source and amount." });
    }

    try {
        const income = new Income({ user_id: req.user_id, source, amount });
        await income.save();
        res.status(201).json({ message: "Income added successfully", income });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Get Income History
app.get("/api/income/history", authMiddleware, async (req, res) => {
    try {
        const incomeHistory = await Income.find({ user_id: req.user_id });
        res.status(200).json(incomeHistory);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Add Expense
app.post("/api/expenses/add", authMiddleware, async (req, res) => {
    const { category, amount } = req.body;

    // Validate input
    if (!category || !amount) {
        return res.status(400).json({ error: "Please provide category and amount." });
    }

    try {
        const expense = new Expense({ user_id: req.user_id, category, amount });
        await expense.save();
        res.status(201).json({ message: "Expense added successfully", expense });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Get Expense Reports
app.get("/api/expenses/reports", authMiddleware, async (req, res) => {
    try {
        const expenses = await Expense.find({ user_id: req.user_id });
        res.status(200).json(expenses);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Add Savings Goal
app.post("/api/savings/add", authMiddleware, async (req, res) => {
    const { goal, targetAmount, deadline } = req.body;

    // Validate input
    if (!goal || !targetAmount || !deadline) {
        return res.status(400).json({ error: "Please provide goal, target amount, and deadline." });
    }

    try {
        const savings = new Savings({ user_id: req.user_id, goal, targetAmount, deadline });
        await savings.save();
        res.status(201).json({ message: "Savings goal added successfully", savings });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Get Savings Progress
app.get("/api/savings/progress", authMiddleware, async (req, res) => {
    try {
        const savings = await Savings.find({ user_id: req.user_id });
        res.status(200).json(savings);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Add Investment
app.post("/api/investments/add", authMiddleware, async (req, res) => {
    const { type, initialAmount, currentValue } = req.body;

    // Validate input
    if (!type || !initialAmount || !currentValue) {
        return res.status(400).json({ error: "Please provide type, initial amount, and current value." });
    }

    try {
        const investment = new Investment({ user_id: req.user_id, type, initialAmount, currentValue });
        await investment.save();
        res.status(201).json({ message: "Investment added successfully", investment });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Get Investment Returns
app.get("/api/investments/returns", authMiddleware, async (req, res) => {
    try {
        const investments = await Investment.find({ user_id: req.user_id });
        const returns = investments.map((investment) => ({
            type: investment.type,
            roi: ((investment.currentValue - investment.initialAmount) / investment.initialAmount) * 100,
        }));
        res.status(200).json(returns);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Add Financial Goal
app.post("/api/goals/add", authMiddleware, async (req, res) => {
    const { goal, targetAmount, deadline } = req.body;

    // Validate input
    if (!goal || !targetAmount || !deadline) {
        return res.status(400).json({ error: "Please provide goal, target amount, and deadline." });
    }

    try {
        const goalData = new Goal({ user_id: req.user_id, goal, targetAmount, deadline });
        await goalData.save();
        res.status(201).json({ message: "Financial goal added successfully", goalData });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Get Goal Progress
app.get("/api/goals/progress", authMiddleware, async (req, res) => {
    try {
        const goals = await Goal.find({ user_id: req.user_id });
        res.status(200).json(goals);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// Start Server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`🚀 Server running on port ${PORT}`));

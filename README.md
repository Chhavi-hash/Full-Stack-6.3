const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');

const app = express();
app.use(bodyParser.json());

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/bankdb', {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('✅ Connected to MongoDB'))
.catch(err => console.error('❌ MongoDB connection error:', err));

// Define Account Schema
const accountSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  balance: { type: Number, required: true, min: 0 }
});

const Account = mongoose.model('Account', accountSchema);

// Create sample accounts (run only once if needed)
app.post('/create', async (req, res) => {
  try {
    const { username, balance } = req.body;
    const account = new Account({ username, balance });
    await account.save();
    res.json({ message: 'Account created successfully', account });
  } catch (error) {
    res.status(400).json({ message: 'Error creating account', error: error.message });
  }
});

// Transfer money endpoint
app.post('/transfer', async (req, res) => {
  const { fromUser, toUser, amount } = req.body;

  if (!fromUser || !toUser || !amount || amount <= 0) {
    return res.status(400).json({ message: 'Invalid input data' });
  }

  try {
    // Fetch both accounts
    const sender = await Account.findOne({ username: fromUser });
    const receiver = await Account.findOne({ username: toUser });

    // Validate existence
    if (!sender) {
      return res.status(404).json({ message: `Sender account (${fromUser}) not found` });
    }
    if (!receiver) {
      return res.status(404).json({ message: `Receiver account (${toUser}) not found` });
    }

    // Check sufficient balance
    if (sender.balance < amount) {
      return res.status(400).json({ message: 'Insufficient balance for transfer' });
    }

    // Update balances logically
    sender.balance -= amount;
    receiver.balance += amount;

    // Save updates sequentially
    await sender.save();
    await receiver.save();

    res.json({
      message: `₹${amount} transferred successfully from ${fromUser} to ${toUser}`,
      fromUser: { username: sender.username, newBalance: sender.balance },
      toUser: { username: receiver.username, newBalance: receiver.balance }
    });
  } catch (error) {
    res.status(500).json({ message: 'Error during transfer', error: error.message });
  }
});

// Get account details (for testing)
app.get('/account/:username', async (req, res) => {
  try {
    const account = await Account.findOne({ username: req.params.username });
    if (!account) return res.status(404).json({ message: 'Account not found' });
    res.json(account);
  } catch (error) {
    res.status(500).json({ message: 'Error retrieving account', error: error.message });
  }
});

// Start server
app.listen(3000, () => console.log('🚀 Server running on http://localhost:3000'));

const express = require('express');
const session = require('express-session');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const dotenv = require('dotenv');

dotenv.config();
const app = express();
const PORT = 3000;

// MongoDB
mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true });
const User = mongoose.model('User', new mongoose.Schema({
  email: String,
  password: String
}));

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));
app.use(session({
  secret: 'supersecretkey',
  resave: false,
  saveUninitialized: true
}));

// Views
const htmlPage = (body) => `
<!DOCTYPE html>
<html>
<head>
  <title>TV Streaming</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: sans-serif; background: #f7f7f7; padding: 20px; }
    form { background: #fff; padding: 20px; max-width: 300px; margin: auto; border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
    input { width: 100%; padding: 10px; margin: 8px 0; }
    button { width: 100%; padding: 10px; background: #007bff; color: white; border: none; border-radius: 5px; }
    iframe { width: 100%; height: 600px; border: none; margin-top: 20px; }
    .link { text-align: center; margin-top: 10px; }
    a { color: #007bff; text-decoration: none; }
  </style>
</head>
<body>
  ${body}
</body>
</html>
`;

// Routes
app.get('/', (req, res) => res.redirect('/login'));

app.get('/signup', (req, res) => {
  res.send(htmlPage(`
    <h2>Sign Up</h2>
    <form method="post" action="/signup">
      <input type="email" name="email" placeholder="Email" required />
      <input type="password" name="password" placeholder="Password" required />
      <button type="submit">Register</button>
      <div class="link"><a href="/login">Already have an account? Login</a></div>
    </form>
  `));
});

app.post('/signup', async (req, res) => {
  const { email, password } = req.body;
  const hashed = await bcrypt.hash(password, 10);
  await User.create({ email, password: hashed });
  res.redirect('/login');
});

app.get('/login', (req, res) => {
  res.send(htmlPage(`
    <h2>Login</h2>
    <form method="post" action="/login">
      <input type="email" name="email" placeholder="Email" required />
      <input type="password" name="password" placeholder="Password" required />
      <button type="submit">Login</button>
      <div class="link"><a href="/signup">Don't have an account? Sign up</a></div>
    </form>
  `));
});

app.post('/login', async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  if (user && await bcrypt.compare(req.body.password, user.password)) {
    req.session.userId = user._id;
    res.redirect('/dashboard');
  } else {
    res.send(htmlPage('<h2>Login failed</h2><p>Invalid credentials. <a href="/login">Try again</a></p>'));
  }
});

app.get('/dashboard', (req, res) => {
  if (!req.session.userId) return res.redirect('/login');

  res.send(htmlPage(`
    <h2>Welcome to Mobile TV</h2>
    <p>Your IPTV Playlist:</p>
    <iframe src="${process.env.M3U_URL}" allowfullscreen></iframe>
    <div class="link"><a href="/logout">Logout</a></div>
  `));
});

app.get('/logout', (req, res) => {
  req.session.destroy(() => res.redirect('/login'));
});

// Start
app.listen(PORT, () => console.log(`Server running: http://localhost:${PORT}`));

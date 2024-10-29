// index.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

app.use(express.static('public'));

let drawingData = [];
let users = {};

io.on('connection', (socket) => {
    console.log('A user connected: ' + socket.id);

    // Send existing drawing data to new user
    socket.emit('load drawing', drawingData);

    // Broadcast drawing data to all clients
    socket.on('drawing', (data) => {
        drawingData.push(data);
        socket.broadcast.emit('drawing', data);
    });

    // Handle user joining
    socket.on('join', (username) => {
        users[socket.id] = username;
        console.log(username + ' joined.');
        io.emit('update users', Object.values(users));
    });

    // Handle user disconnecting
    socket.on('disconnect', () => {
        delete users[socket.id];
        console.log('User disconnected: ' + socket.id);
        io.emit('update users', Object.values(users));
    });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});

<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Multiplayer Drawing Game</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>Multiplayer Drawing Game</h1>
        <div id="canvas-container">
            <canvas id="drawing-canvas"></canvas>
        </div>
        <div>
            <input type="text" id="username" placeholder="Enter your name">
            <button id="join-button">Join Game</button>
        </div>
        <div id="user-list"></div>
    </div>
    <script src="/socket.io/socket.io.js"></script>
    <script src="app.js"></script>
</body>
</html>

// public/app.js
const canvas = document.getElementById('drawing-canvas');
const ctx = canvas.getContext('2d');
const socket = io();
const usernameInput = document.getElementById('username');
const joinButton = document.getElementById('join-button');
const userList = document.getElementById('user-list');
let drawing = false;

canvas.width = 800;  // Set canvas width
canvas.height = 400; // Set canvas height

// Handle joining the game
joinButton.addEventListener('click', () => {
    const username = usernameInput.value;
    if (username) {
        socket.emit('join', username);
        usernameInput.value = '';
    }
});

// Drawing on canvas
canvas.addEventListener('mousedown', () => {
    drawing = true;
});

canvas.addEventListener('mouseup', () => {
    drawing = false;
    ctx.beginPath(); // Reset for new line
});

canvas.addEventListener('mousemove', (event) => {
    if (!drawing) return;

    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;

    draw(x, y);
    socket.emit('drawing', { x, y });
});

// Draw on canvas
function draw(x, y) {
    ctx.lineWidth = 2;
    ctx.lineCap = 'round';
    ctx.strokeStyle = 'black';

    ctx.lineTo(x, y);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(x, y);
}

// Load existing drawing
socket.on('load drawing', (data) => {
    data.forEach(({ x, y }) => {
        draw(x, y);
    });
});

// Update drawing in real-time
socket.on('drawing', (data) => {
    draw(data.x, data.y);
});

// Update user list
socket.on('update users', (users) => {
    userList.innerHTML = 'Users: ' + users.join(', ');
});

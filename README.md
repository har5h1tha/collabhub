# CollabHub

CollabHub is a real-time messaging platform with private and group chats.

## Features

### Authentication

- Register with a username, email, and password.
- Log in with email and password.
- Receive a JWT used by protected REST routes and Socket.IO connections.
- View the current user's profile.

### Private Chat

- List users and conversations.
- Send private messages.
- Load private message history with pagination.
- Show sent, delivered, and read message status.

### Group Chat

- Create groups.
- Add and remove members.
- Assign the group creator as an administrator.
- Leave a group when the current user is not an administrator.
- Send and receive group messages through Socket.IO rooms.
- Load group message history through a paginated REST endpoint.

### Real-time

- Typing and stop-typing indicators for private chats.
- Typing and stop-typing indicators for groups.
- Online and offline presence.
- Delivery and read status updates.
- Multiple Socket.IO connections for one user.

## Tech Stack

### Backend

- Node.js
- Express
- MongoDB
- Mongoose
- Socket.IO
- JSON Web Tokens
- bcrypt

### Frontend

- React
- Vite
- Socket.IO Client

### Tools

- Nodemon
- Docker
- Docker Compose

## Architecture

```text
                    CollabHub
                       │
              ┌────────┴────────┐
              │                 │
          REST API           Socket.IO
              │                 │
          Express          Real-time events
              │                 │
              └────────┬────────┘
                       │
                    MongoDB
```

REST handles authentication, user and conversation queries, group management, message history, and database writes exposed by the routes. Socket.IO handles live private messages, group messages, typing indicators, presence, delivery updates, read updates, and group room membership. Both REST and Socket.IO verify the user's JWT before protected work.

## API Endpoints

Protected endpoints require:

```http
Authorization: Bearer YOUR_JWT_TOKEN
```

### Authentication

| Method | Endpoint | Description | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/auth/register` | Register a user | No |
| `POST` | `/api/auth/login` | Log in and receive a JWT | No |
| `GET` | `/api/auth/profile` | Get the current user's ID and username | Yes |

### Users

| Method | Endpoint | Description | Auth |
| --- | --- | --- | --- |
| `GET` | `/api/users` | List users | Yes |
| `GET` | `/api/users/me` | Get the current user | Yes |

### Private Messages

| Method | Endpoint | Description | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/messages/send` | Create and send a private message | Yes |
| `GET` | `/api/messages/conversations` | Get conversations and unread counts | Yes |
| `GET` | `/api/messages/:userId` | Get private message history; supports `page` and `limit` | Yes |

### Groups

| Method | Endpoint | Description | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/groups` | Create a group | Yes |
| `GET` | `/api/groups` | List groups for the current user | Yes |
| `GET` | `/api/groups/:groupId` | Get group details | Yes |
| `POST` | `/api/groups/:groupId/members` | Add a member; administrator only | Yes |
| `DELETE` | `/api/groups/:groupId/members/:userId` | Remove a member; administrator only | Yes |
| `DELETE` | `/api/groups/:groupId/leave` | Leave a group; administrators cannot currently leave | Yes |
| `POST` | `/api/groups/:groupId/messages` | Create a group message through REST | Yes |
| `GET` | `/api/groups/:groupId/messages` | Get paginated group message history | Yes |

## Socket.IO Events

The client sends the JWT in `socket.handshake.auth.token`.

### Client to Server

| Event | Payload | Purpose |
| --- | --- | --- |
| `typing` | `{ receiverId }` | Tell a private-chat recipient that the user is typing |
| `stop-typing` | `{ receiverId }` | Stop a private-chat typing indicator |
| `message-delivered` | `{ messageId }` | Mark a private message as delivered |
| `message-read` | `{ messageId }` | Mark a private message as read |
| `joinGroup` | `groupId` | Join a group room after membership is checked |
| `send-group-message` | `{ groupId, content }` | Save and broadcast a group message |
| `group-typing` | `{ groupId }` | Send a group typing indicator |
| `group-stop-typing` | `{ groupId }` | Stop a group typing indicator |
| `leaveGroup` | `{ groupId }` | Leave a group room |

### Server to Client

| Event | Payload | Purpose |
| --- | --- | --- |
| `online-users` | User ID array | Send the users currently tracked as online |
| `user-online` | `{ userId }` | Notify that a user connected |
| `user-offline` | `{ userId }` | Notify that a user has no remaining connections |
| `user-typing` | `{ userId }` | Notify about private-chat typing |
| `user-stop-typing` | `{ userId }` | Stop a private-chat typing indicator |
| `receive-message` | Message object | Deliver a private message |
| `message-delivered` | `{ messageId, status }` | Confirm delivery status |
| `message-read` | `{ messageId, status }` | Confirm read status |
| `messages-read` | `{ messageIds }` | Mark several messages read after history is opened |
| `message-error` | `{ message }` | Report a private-message status error |
| `group-joined` | `{ groupId }` | Confirm that a socket joined a group room |
| `group-left` | `{ groupId }` | Confirm that a socket left a group room |
| `new-group-message` | `{ groupId, message }` | Broadcast a new group message to the room |
| `group-user-typing` | `{ userId, username }` | Notify room members about group typing |
| `group-user-stop-typing` | `{ userId }` | Stop a group typing indicator |
| `group-error` | `{ message }` | Report a group join error |
| `group-message-error` | `{ message }` | Report a group message or typing error |

## Design Decisions

### Multi-device connections

The server maps each user ID to a `Set` of Socket.IO IDs. A `user-offline` event is emitted only after the disconnected socket is removed and that set becomes empty. Events such as messages and private typing indicators are sent to every socket in the user's set.

### Message status

Private messages use the statuses `sent`, `delivered`, and `read`. Conditional database updates only move `sent` to `delivered` and `sent` or `delivered` to `read`, so a later event cannot move a message backward.

### JWT verification

The REST middleware reads a `Bearer` token from the `Authorization` header and verifies it with `JWT_SECRET`. Socket.IO middleware reads the token from `socket.handshake.auth.token` and verifies it with the same secret.

### Group administrator permissions

The server stores administrators separately from members. Add and remove operations require the requester to be in the group's `admins` list, and the server also checks membership before group details, history, room joins, and group messages are allowed.

## Group Chat

Groups are created with `POST /api/groups`. The creator is added as the first member and administrator. Administrators can add members and remove other members through REST routes. An administrator cannot currently remove themselves or leave the group.

The frontend joins a group with `joinGroup`, sends messages with `send-group-message`, and receives them through `new-group-message`. Group typing uses `group-typing` and `group-stop-typing`. The frontend loads group history from `GET /api/groups/:groupId/messages?page=1&limit=20`, so it currently loads only the first page.

## Project Structure

```text
collabhub/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── chat/
│   │   │   ├── AddGroupMember.jsx
│   │   │   ├── ChatWindow.jsx
│   │   │   ├── CreateGroup.jsx
│   │   │   ├── GroupChat.jsx
│   │   │   ├── GroupDetails.jsx
│   │   │   └── GroupList.jsx
│   │   ├── pages/
│   │   │   ├── Chat.jsx
│   │   │   ├── Login.jsx
│   │   │   └── Register.jsx
│   │   ├── App.jsx
│   │   ├── index.css
│   │   └── main.jsx
│   ├── Dockerfile
│   ├── package.json
│   └── vite.config.js
├── src/
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── groupController.js
│   │   ├── message.controller.js
│   │   └── user.controller.js
│   ├── database/
│   │   └── db.js
│   ├── middleware/
│   │   ├── auth.middleware.js
│   │   ├── error.middleware.js
│   │   └── socketAuth.middleware.js
│   ├── models/
│   │   ├── group.model.js
│   │   ├── groupMessage.model.js
│   │   ├── message.model.js
│   │   └── user.model.js
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── group.routes.js
│   │   ├── message.routes.js
│   │   └── user.routes.js
│   ├── socket/
│   │   └── socket.js
│   └── utils/
│       ├── ApiError.js
│       └── asyncHandler.js
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── package.json
└── server.js
```

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/har5h1tha/collabhub.git
cd collabhub
```

### 2. Install backend dependencies

```bash
npm install
```

### 3. Configure the backend

Create `.env` in the project root:

```env
PORT=3000
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
```

### 4. Start the backend

```bash
npm run dev
```

The backend listens on port `3000` by default.

### 5. Configure and start the frontend

Create `frontend/.env`:

```env
VITE_API_URL=http://localhost:3000
```

Then install dependencies and start Vite:

```bash
cd frontend
npm install
npm run dev
```

## Current Status

### Completed

- [x] User registration and login
- [x] JWT authentication for REST and Socket.IO
- [x] Protected REST routes
- [x] User profile and user listing
- [x] Private messaging
- [x] Private message history pagination
- [x] Sent, delivered, and read status
- [x] Typing indicators
- [x] Online/offline presence
- [x] Multi-device socket handling
- [x] Group chats
- [x] Socket.IO rooms
- [x] Group administrator roles
- [x] Group member management
- [x] Group history pagination
- [x] Docker and Docker Compose configuration

### Planned

- [ ] Redis
- [ ] Rate limiting
- [ ] Structured logging
- [ ] Refresh tokens
- [ ] Automated tests
- [ ] Hosted deployment

## Known Limitations

- Group history pagination exists in the API, but the frontend currently loads only page 1 with a limit of 20.
- Administrators cannot currently leave a group.
- An administrator cannot remove themselves from a group.
- There is no Redis integration.
- There are no automated tests.
- There is no refresh-token flow.
- There is no hosted deployment or live demo.

## Screenshots

<!-- ADD SCREENSHOT -->

## Live Demo

<!-- ADD LIVE LINK -->

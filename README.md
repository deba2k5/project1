# Nexus - Privacy-First Collaborative Workspace

> **Zero-Cloud File Sharing • Real-Time P2P Collaboration • AI-Assisted Task Management**

## 🎯 The Problem

Modern collaboration tools force teams into an impossible choice:
- **Upload sensitive files** to centralized cloud storage (AWS, Google Drive, Dropbox)
- **Fragment work** across Jira, Slack, Drive, and Calendar
- **Manually track** progress, summaries, and action items
- **Trust** third-party servers with client data

**For agencies, consultancies, and client-facing teams—this is a dealbreaker.**

## 💡 Our Solution

**Nexus** reimagines collaboration with three core principles:

### 1. **Zero-Cloud File Storage**
Files transfer **peer-to-peer via WebRTC**. Your server never sees file contents—only metadata.

### 2. **Room-Based Privacy**
Workspace → Client Rooms → Sub-Rooms. Every client is isolated. No data leaks.

### 3. **AI That Reduces Noise**
AI generates summaries, extracts tasks from chat, and tracks progress—**without adding busywork.**

---

## ✨ Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| 🔐 **OAuth Authentication** | Google OAuth + Firebase Auth | ✅ Required |
| 🏢 **Hierarchical Workspaces** | Workspace → Rooms → Sub-Rooms | ✅ Required |
| 📋 **Agile Task Boards** | Jira-like kanban with drag-and-drop | ✅ Required |
| ⚡ **Real-Time Sync** | WebSocket-powered task/chat updates | ✅ Required |
| 💬 **Private Chat** | Sub-room scoped, session-logged | ✅ Required |
| 📁 **P2P File Sharing** | Zero cloud storage, WebRTC-based | ✅ **Core Differentiator** |
| 🔍 **Session Privacy Logs** | Audit who accessed what, when | ✅ Required |
| 🤖 **AI Summaries** | Workspace/task summaries via LLM | ✅ Required |
| ✍️ **AI Task Generation** | Extract tasks from chat/notes | ✅ Required |
| 📅 **Calendar Integration** | Google Calendar sync | ✅ Required |


---

## 🛠️ Tech Stack

### **Frontend**
- **Core**: React 18 + TypeScript + Vite
- **UI**: Tailwind CSS + ShadCN UI + Material UI (minimal)
- **State**: React Query (server state) + Zustand (client state)
- **Real-time**: Socket.io-client
- **P2P**: WebRTC (RTCDataChannel)
- **3D**: Three.js (landing page only)

### **Backend**
- **Runtime**: Node.js + Express.js (TypeScript)
- **Real-time**: Socket.io
- **Auth**: Firebase Authentication (Google OAuth)
- **Database**: Firebase Firestore (metadata only)
- **AI**: Groq + LangChain + LangGraph

### **Infrastructure**
- **Frontend Hosting**: Vercel / Firebase Hosting
- **Backend Hosting**: Railway / Render / Firebase Functions
- **P2P**: STUN servers (no TURN needed for demo)

---

## 📊 Data Models

### **Core Collections**

#### `users`
```typescript
{
  uid: string;              // Firebase Auth UID
  name: string;
  email: string;
  photoURL: string;
  workspaceId: string;
  createdAt: Timestamp;
  lastLoginAt: Timestamp;
}
```

#### `workspaces`
```typescript
{
  id: string;
  name: string;
  createdBy: string;        // uid
  members: {
    [uid: string]: 'admin' | 'member'
  };
  inviteCode: string;       // UUID
  createdAt: Timestamp;
}
```

#### `rooms`
```typescript
{
  id: string;
  workspaceId: string;
  name: string;             // e.g., "Client - Startup X"
  createdBy: string;
  members: {
    [uid: string]: 'admin' | 'member'
  };
  createdAt: Timestamp;
}
```

#### `subrooms`
```typescript
{
  id: string;
  workspaceId: string;
  roomId: string;
  name: string;             // e.g., "Sprint 1"
  createdBy: string;
  members: {
    [uid: string]: 'admin' | 'member'
  };
  createdAt: Timestamp;
}
```

#### `projects`
```typescript
{
  id: string;
  workspaceId: string;
  roomId: string;
  subroomId: string;
  name: string;
  description: string;
  createdBy: string;
  members: {
    [uid: string]: 'admin' | 'member'
  };
  createdAt: Timestamp;
}
```

#### `boards`
```typescript
{
  id: string;
  projectId: string;
  columns: ['Todo', 'In Progress', 'Done'];
  createdAt: Timestamp;
}
```

#### `tasks`
```typescript
{
  id: string;
  workspaceId: string;
  roomId: string;
  subroomId: string;
  projectId: string;
  boardId: string;
  title: string;
  description: string;
  status: 'Todo' | 'In Progress' | 'Done';
  assignees: string[];      // [uid]
  createdBy: string;
  createdAt: Timestamp;
  updatedAt: Timestamp;
  dueDate?: Timestamp;
  fileIds: string[];
}
```

#### `files` (Metadata Only - No File Content!)
```typescript
{
  id: string;
  taskId: string;
  subroomId: string;
  projectId: string;
  fileName: string;
  fileSize: number;
  fileType: string;
  sharedBy: string;         // uid
  allowedReceivers: string[]; // [uid]
  createdAt: Timestamp;
}
```

#### `sessions` (Privacy Audit Log)
```typescript
{
  sessionId: string;        // UUID
  uid: string;
  workspaceId: string;
  roomId: string;
  subroomId: string;
  entryTime: Timestamp;
  exitTime?: Timestamp;
  actionsCount: number;
  createdAt: Timestamp;
  expiresAt: Timestamp;     // +1 year
}
```

#### `taskActivityLogs`
```typescript
{
  id: string;
  taskId: string;
  action: 'CREATED' | 'STATUS_CHANGED' | 'ASSIGNED' | 'COMMENTED';
  previousValue?: string;
  newValue: string;
  performedBy: string;      // uid
  sessionId: string;
  timestamp: Timestamp;
}
```

### **Firestore Indexes (Critical for Demo Performance)**
```javascript
// Composite indexes required:
tasks: [subroomId, status]
projects: [subroomId]
files: [taskId]
sessions: [uid, createdAt]
taskActivityLogs: [taskId, timestamp]
```

---

## 🔌 Backend API Routes

### **Authentication**
```
POST   /api/auth/login              # Google OAuth login
POST   /api/auth/logout             # End session
GET    /api/auth/me                 # Get current user
```

### **Workspaces**
```
POST   /api/workspaces              # Create workspace
GET    /api/workspaces/:id          # Get workspace details
POST   /api/workspaces/:id/invite   # Generate invite code
POST   /api/workspaces/join         # Join via invite code
GET    /api/workspaces/:id/members  # List members
```

### **Rooms**
```
POST   /api/workspaces/:wid/rooms           # Create room
GET    /api/workspaces/:wid/rooms           # List rooms
GET    /api/rooms/:id                       # Get room details
PUT    /api/rooms/:id                       # Update room
DELETE /api/rooms/:id                       # Delete room
POST   /api/rooms/:id/members               # Add member
```

### **Sub-Rooms**
```
POST   /api/rooms/:rid/subrooms             # Create sub-room
GET    /api/rooms/:rid/subrooms             # List sub-rooms
GET    /api/subrooms/:id                    # Get sub-room details
PUT    /api/subrooms/:id                    # Update sub-room
DELETE /api/subrooms/:id                    # Delete sub-room
```

### **Projects & Boards**
```
POST   /api/subrooms/:sid/projects          # Create project
GET    /api/subrooms/:sid/projects          # List projects
GET    /api/projects/:id                    # Get project details
POST   /api/projects/:pid/boards            # Create board (auto-created)
GET    /api/boards/:id                      # Get board with tasks
```

### **Tasks**
```
POST   /api/boards/:bid/tasks               # Create task
GET    /api/boards/:bid/tasks               # List tasks by board
GET    /api/tasks/:id                       # Get task details
PUT    /api/tasks/:id                       # Update task
DELETE /api/tasks/:id                       # Delete task
PATCH  /api/tasks/:id/status                # Update task status
POST   /api/tasks/:id/assign                # Assign user
GET    /api/tasks/:id/activity              # Get activity logs
```

### **File Sharing (Metadata Only)**
```
POST   /api/tasks/:tid/files/metadata       # Store file metadata
GET    /api/tasks/:tid/files                # List file metadata
DELETE /api/files/:id                       # Delete file metadata
GET    /api/files/:id/receivers             # Get allowed receivers
```

### **Chat**
```
POST   /api/subrooms/:sid/messages          # Send message (also via WS)
GET    /api/subrooms/:sid/messages          # Get message history
```

### **AI Services**
```
POST   /api/ai/summarize/workspace/:wid     # Generate workspace summary
POST   /api/ai/summarize/subroom/:sid       # Generate sub-room summary
POST   /api/ai/summarize/task/:tid          # Generate task summary
POST   /api/ai/extract-tasks                # Extract tasks from text
```

### **Calendar**
```
POST   /api/subrooms/:sid/meetings          # Create meeting + sync to Google
GET    /api/subrooms/:sid/meetings          # List meetings
PUT    /api/meetings/:id                    # Update meeting
DELETE /api/meetings/:id                    # Delete meeting
```

### **Sessions**
```
POST   /api/sessions/start                  # Start session (auto on login)
POST   /api/sessions/:id/end                # End session
GET    /api/sessions/user/:uid              # Get user sessions
GET    /api/sessions/subroom/:sid           # Get sub-room sessions
```

### **WebSocket Events**
```javascript
// Client → Server
'join:subroom'              // Join sub-room for real-time updates
'task:create'               // Create task
'task:update'               // Update task
'task:move'                 // Drag-and-drop status change
'chat:message'              // Send chat message
'webrtc:signal'             // WebRTC signaling for P2P

// Server → Client
'task:created'              // Task created by another user
'task:updated'              // Task updated
'task:moved'                // Task status changed
'chat:message'              // New chat message
'webrtc:signal'             // WebRTC signaling response
'user:joined'               // User joined sub-room
'user:left'                 // User left sub-room
```

---

## 📱 Frontend Pages/Routes

### **Public Routes**
```
/                           # Landing page (Three.js hero)
/login                      # Google OAuth login
```

### **Authenticated Routes**
```
/dashboard                  # Workspace selector
/workspace/:wid             # Workspace overview + rooms list
/workspace/:wid/invite      # Join workspace via invite code

/room/:rid                  # Room overview + sub-rooms list
/subroom/:sid               # Sub-room dashboard (chat + projects)

/project/:pid               # Project overview
/board/:bid                 # Kanban board (main view)

/task/:tid                  # Task detail modal/page

/settings                   # User settings
/workspace/:wid/settings    # Workspace settings (admin only)
```

### **Component Structure**
```
src/
├── pages/
│   ├── LandingPage.tsx
│   ├── Login.tsx
│   ├── Dashboard.tsx
│   ├── WorkspaceView.tsx
│   ├── RoomView.tsx
│   ├── SubRoomView.tsx
│   ├── BoardView.tsx
│   └── TaskDetailModal.tsx
├── components/
│   ├── auth/
│   │   └── GoogleLoginButton.tsx
│   ├── workspace/
│   │   ├── WorkspaceCard.tsx
│   │   └── InviteCodeInput.tsx
│   ├── room/
│   │   └── RoomCard.tsx
│   ├── subroom/
│   │   ├── ChatPanel.tsx
│   │   └── ProjectList.tsx
│   ├── board/
│   │   ├── KanbanBoard.tsx
│   │   ├── TaskCard.tsx
│   │   └── ColumnDrop.tsx
│   ├── task/
│   │   ├── TaskForm.tsx
│   │   ├── TaskDetail.tsx
│   │   └── FileAttachment.tsx
│   ├── file/
│   │   ├── P2PFileUploader.tsx
│   │   └── FileReceiver.tsx
│   ├── ai/
│   │   ├── SummaryButton.tsx
│   │   └── TaskExtractor.tsx
│   └── calendar/
│       └── MeetingScheduler.tsx
├── hooks/
│   ├── useWebRTC.ts
│   ├── useSocket.ts
│   ├── useAuth.ts
│   └── useFileTransfer.ts
└── lib/
    ├── firebase.ts
    ├── socket.ts
    └── webrtc.ts
```

---

## 🚀 Getting Started

### **Prerequisites**
- Node.js 18+
- npm/yarn/pnpm
- Firebase project
- Google OAuth credentials
- Groq API key

### **1. Clone & Install**
```bash
# Clone repository
git clone https://github.com/your-team/nexus.git
cd nexus

# Install frontend dependencies
cd frontend
npm install

# Install backend dependencies
cd ../backend
npm install
```

### **2. Environment Setup**

**Frontend** (`.env`)
```env
VITE_FIREBASE_API_KEY=your_firebase_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_FIREBASE_APP_ID=your_app_id
VITE_BACKEND_URL=http://localhost:5000
VITE_SOCKET_URL=http://localhost:5000
```

**Backend** (`.env`)
```env
PORT=5000
NODE_ENV=development

# Firebase Admin SDK
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_PRIVATE_KEY="your_private_key"
FIREBASE_CLIENT_EMAIL=your_service_account_email

# Groq AI
GROQ_API_KEY=your_groq_api_key

# Google Calendar (OAuth)
GOOGLE_CLIENT_ID=your_client_id
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_REDIRECT_URI=http://localhost:5000/api/auth/google/callback

# CORS
CORS_ORIGIN=http://localhost:5173

# STUN Server
STUN_SERVER=stun:stun.l.google.com:19302
```

### **3. Firebase Setup**
```bash
# Install Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# Initialize project
firebase init firestore
firebase init functions
firebase init hosting
```

**Firestore Rules** (`firestore.rules`)
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can read/write their own data
    match /users/{userId} {
      allow read, write: if request.auth.uid == userId;
    }
    
    // Workspace members can access workspace data
    match /workspaces/{workspaceId} {
      allow read: if request.auth.uid in resource.data.members;
      allow write: if request.auth.uid == resource.data.createdBy;
    }
    
    // Similar rules for rooms, subrooms, tasks, etc.
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

### **4. Run Development Servers**

**Terminal 1 - Backend**
```bash
cd backend
npm run dev
# Runs on http://localhost:5000
```

**Terminal 2 - Frontend**
```bash
cd frontend
npm run dev
# Runs on http://localhost:5173
```

### **5. Build for Production**
```bash
# Frontend
cd frontend
npm run build
firebase deploy --only hosting

# Backend
cd backend
npm run build
# Deploy to Railway/Render/Firebase Functions
```

---

## 🎬 User Flows

### **Flow 1: Workspace Creation**
1. User signs in via Google OAuth
2. Creates workspace "Acme Corp"
3. Workspace invite code generated (UUID)
4. Internal members join using code

### **Flow 2: Client Room Setup**
1. Admin creates room: "Client - Startup X"
2. Invites team members
3. Room is isolated from other clients

### **Flow 3: Sub-Room Creation**
1. Inside "Client - Startup X"
2. Create sub-rooms: "Sprint 1", "Frontend Team"
3. Each sub-room has: chat, tasks, files, meetings

### **Flow 4: Task Management**
1. Create project "MVP Delivery" in sub-room
2. Project auto-creates board (Todo/In Progress/Done)
3. Add task "Implement Auth"
4. Drag task from Todo → In Progress
5. Real-time sync via WebSocket

### **Flow 5: P2P File Sharing**
1. Open task "Implement Auth"
2. Click "Share File"
3. Select file (e.g., design.pdf)
4. Backend stores metadata only
5. WebRTC establishes P2P connection
6. File transfers directly to allowed receivers
7. File never touches server

### **Flow 6: AI Summary**
1. Click "Generate Summary" in sub-room
2. AI reads: tasks, status changes, comments
3. Outputs: "Sprint progress summary"
4. Shows blocked tasks and key updates

### **Flow 7: Calendar Sync**
1. Schedule meeting in sub-room
2. Sync to Google Calendar
3. Metadata stored (title, time, participants)

### **Flow 8: Session Privacy**
1. User enters workspace → session starts
2. Logged: session ID, user, workspace, room, time
3. User exits → session ends
4. Privacy audit trail created

---

## 🎯 What Makes This Hackathon-Winning

### **1. Clear Problem-Solution Fit**
- **Problem**: Cloud storage privacy concerns + tool fragmentation
- **Solution**: P2P file sharing + unified workspace

### **2. Technical Innovation**
- WebRTC for zero-cloud file transfer
- Real-time collaboration via WebSocket
- Privacy-first session logging

### **3. AI That Actually Helps**
- Summaries reduce meeting overhead
- Task extraction from chat saves time
- No AI noise, just utility

### **4. Production-Ready Architecture**
- Scalable backend (Firebase + Express)
- Modern frontend (React + TypeScript)
- Real deployment strategy

### **5. Client-Facing Use Case**
- Agencies, consultancies, freelancers
- Immediate market need
- Clear monetization path

---

## 🔮 Future Roadmap

### **Phase 1: Post-Hackathon MVP** (Week 1-2)
- [ ] End-to-end encryption for P2P transfers
- [ ] Mobile app (React Native)
- [ ] Offline-first with IndexedDB
- [ ] TURN server for NAT traversal

### **Phase 2: Enterprise Features** (Month 1-3)
- [ ] SSO (Okta, Azure AD)
- [ ] Advanced analytics dashboard
- [ ] Compliance exports (GDPR, HIPAA)
- [ ] Slack/Discord integrations

### **Phase 3: Scale** (Month 3-6)
- [ ] Self-hosted option
- [ ] API for third-party integrations
- [ ] Advanced AI agents (auto-prioritization)
- [ ] Video/audio P2P calls

---

## 👥 Team Structure (3-4 Developers)

### **Developer 1: Frontend Lead**
- React components + UI/UX
- WebRTC client implementation
- Real-time state management

### **Developer 2: Backend Lead**
- Express API + Socket.io
- Firebase integration
- WebRTC signaling server

### **Developer 3: Full-Stack**
- Task management flows
- Chat implementation
- Calendar integration

### **Developer 4 (Optional): AI/DevOps**
- Groq AI integration
- LangChain workflows
- Deployment + CI/CD

---

## 📄 License

MIT License - feel free to use this for your hackathon!

---

## 🙏 Acknowledgments

- Firebase for auth & database
- Groq for AI inference
- WebRTC for P2P magic
- The open-source community

---

**Built with ❤️ for privacy-conscious teams**

*Questions? Open an issue or reach out to the team!*

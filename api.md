# Frontend Pages & Backend Routes - Detailed Specification

## 📱 FRONTEND PAGES

### 1. **Landing Page** `/`
**Purpose**: Marketing page with hero section

**Components Required**:
- Three.js animated background (abstract shapes, particles)
- Hero section with headline and CTA
- Feature showcase (3 columns: P2P, Privacy, AI)
- "Get Started" button → redirects to `/login`

**State Management**:
- No auth required
- No data fetching

**Libraries**:
- Three.js for 3D animation
- Framer Motion for scroll animations

---

### 2. **Login Page** `/login`
**Purpose**: Google OAuth authentication

**Components Required**:
- Google login button (Firebase Auth)
- Logo and tagline
- "Privacy First" messaging

**State Management**:
- `useAuth` hook
- Listen for auth state changes
- Redirect to `/dashboard` on success

**API Calls**:
- Firebase Auth: `signInWithPopup(auth, googleProvider)`
- Backend: `POST /api/auth/login` (create user session)

**Form Fields**: None (OAuth only)

---

### 3. **Dashboard** `/dashboard`
**Purpose**: View all workspaces user belongs to

**Components Required**:
- Workspace list (cards)
- "Create Workspace" button
- "Join Workspace" button (via invite code)
- User profile dropdown (logout, settings)

**Data Fetching**:
```typescript
// GET /api/workspaces (returns user's workspaces)
{
  workspaces: [
    {
      id: string,
      name: string,
      role: 'admin' | 'member',
      memberCount: number,
      createdAt: Timestamp
    }
  ]
}
```

**Actions**:
- Click workspace → Navigate to `/workspace/:wid`
- Create workspace → Open modal
- Join workspace → Input invite code

**State Management**:
- React Query: `useQuery(['workspaces'])`
- Zustand: UI state (modals)

---

### 4. **Create Workspace Modal**
**Purpose**: Create new workspace

**Form Fields**:
- Workspace name (text input, required)

**API Call**:
```typescript
POST /api/workspaces
Input: { name: string }
Output: { 
  workspace: {
    id: string,
    name: string,
    inviteCode: string,
    createdBy: string,
    members: { [uid]: 'admin' }
  }
}
```

**On Success**:
- Close modal
- Refetch workspaces
- Navigate to `/workspace/:wid`

---

### 5. **Workspace View** `/workspace/:wid`
**Purpose**: View all client rooms in workspace

**Components Required**:
- Workspace header (name, invite code button)
- Room list (cards with member count)
- "Create Room" button (admin only)
- Sidebar: workspace members list

**Data Fetching**:
```typescript
// GET /api/workspaces/:wid
{
  workspace: {
    id: string,
    name: string,
    inviteCode: string,
    members: { [uid]: 'admin' | 'member' }
  }
}

// GET /api/workspaces/:wid/rooms
{
  rooms: [
    {
      id: string,
      name: string,
      subroomCount: number,
      memberCount: number,
      createdAt: Timestamp
    }
  ]
}
```

**Actions**:
- Click room → Navigate to `/room/:rid`
- Create room → Open modal
- Copy invite code → Show toast notification

**Real-time**:
- Socket: Listen for `room:created`, `room:updated`

---

### 6. **Create Room Modal**
**Purpose**: Create client room

**Form Fields**:
- Room name (text input, e.g., "Client - Startup X")
- Description (textarea, optional)

**API Call**:
```typescript
POST /api/workspaces/:wid/rooms
Input: { 
  name: string,
  description?: string 
}
Output: { 
  room: {
    id: string,
    name: string,
    workspaceId: string,
    createdBy: string,
    members: { [uid]: 'admin' }
  }
}
```

---

### 7. **Room View** `/room/:rid`
**Purpose**: View all sub-rooms within client room

**Components Required**:
- Room header (name, breadcrumb: Workspace > Room)
- Sub-room list (cards)
- "Create Sub-Room" button
- Room settings (admin only)

**Data Fetching**:
```typescript
// GET /api/rooms/:rid
{
  room: {
    id: string,
    name: string,
    workspaceId: string,
    members: { [uid]: 'admin' | 'member' }
  }
}

// GET /api/rooms/:rid/subrooms
{
  subrooms: [
    {
      id: string,
      name: string,
      projectCount: number,
      activeTaskCount: number,
      lastActivity: Timestamp
    }
  ]
}
```

**Actions**:
- Click sub-room → Navigate to `/subroom/:sid`
- Create sub-room → Open modal

---

### 8. **Sub-Room View** `/subroom/:sid`
**Purpose**: Main collaboration space (chat + projects + files)

**Layout**: Split screen
- Left: Chat panel (30%)
- Right: Projects list + AI summary button (70%)

**Components Required**:

#### **Left Panel: Chat**
- Message list (scrollable)
- Message input box
- File upload button (stores metadata, triggers P2P)
- User avatars

#### **Right Panel: Projects**
- Project cards (click to view board)
- "Create Project" button
- "Generate AI Summary" button (workspace/sub-room level)
- "Schedule Meeting" button

**Data Fetching**:
```typescript
// GET /api/subrooms/:sid
{
  subroom: {
    id: string,
    name: string,
    roomId: string,
    workspaceId: string
  }
}

// GET /api/subrooms/:sid/messages
{
  messages: [
    {
      id: string,
      text: string,
      senderId: string,
      senderName: string,
      senderPhoto: string,
      timestamp: Timestamp,
      sessionId: string
    }
  ]
}

// GET /api/subrooms/:sid/projects
{
  projects: [
    {
      id: string,
      name: string,
      description: string,
      taskCount: number,
      completedTasks: number
    }
  ]
}
```

**Real-time (Socket.io)**:
```typescript
// Connect to sub-room
socket.emit('join:subroom', { subroomId: sid });

// Listen for new messages
socket.on('chat:message', (message) => {
  // Add to message list
});

// Send message
socket.emit('chat:message', {
  subroomId: sid,
  text: string,
  sessionId: string
});
```

**Actions**:
- Send message → Socket emit
- Create project → Open modal
- Generate AI summary → Show loading, then modal with summary
- Schedule meeting → Open calendar modal

---

### 9. **Create Project Modal**
**Purpose**: Create project within sub-room

**Form Fields**:
- Project name (text input)
- Description (textarea)

**API Call**:
```typescript
POST /api/subrooms/:sid/projects
Input: { 
  name: string,
  description: string 
}
Output: { 
  project: {
    id: string,
    name: string,
    subroomId: string,
    boardId: string  // Auto-created
  }
}
```

**On Success**:
- Refetch projects
- Navigate to `/board/:boardId`

---

### 10. **Board View (Kanban)** `/board/:bid`
**Purpose**: Jira-like task board

**Layout**: 3 columns (Todo, In Progress, Done)

**Components Required**:
- Kanban board (drag-and-drop)
- Task cards (title, assignee avatar, file count)
- "Create Task" button
- Column headers with task count
- Filter/search bar

**Data Fetching**:
```typescript
// GET /api/boards/:bid
{
  board: {
    id: string,
    projectId: string,
    columns: ['Todo', 'In Progress', 'Done']
  },
  tasks: [
    {
      id: string,
      title: string,
      description: string,
      status: 'Todo' | 'In Progress' | 'Done',
      assignees: [
        { uid: string, name: string, photoURL: string }
      ],
      fileCount: number,
      createdBy: string,
      createdAt: Timestamp,
      dueDate?: Timestamp
    }
  ]
}
```

**Real-time (Socket.io)**:
```typescript
// Join board room
socket.emit('join:board', { boardId: bid });

// Listen for task updates
socket.on('task:created', (task) => { /* add to board */ });
socket.on('task:updated', (task) => { /* update task */ });
socket.on('task:moved', (task) => { /* move to new column */ });

// Drag and drop task
const handleDragEnd = (taskId, newStatus) => {
  socket.emit('task:move', {
    taskId,
    oldStatus: task.status,
    newStatus
  });
};
```

**Actions**:
- Create task → Open modal/sidebar
- Click task → Open task detail modal
- Drag task → Update status via socket
- Assign user → Dropdown menu

**Drag-and-Drop**:
- Library: `@dnd-kit/core` or `react-beautiful-dnd`
- On drop → Optimistic update + socket emit

---

### 11. **Create Task Modal**
**Purpose**: Add new task to board

**Form Fields**:
- Title (text input, required)
- Description (textarea, rich text)
- Assignees (multi-select dropdown)
- Due date (date picker, optional)
- Status (select: Todo/In Progress/Done)

**API Call**:
```typescript
POST /api/boards/:bid/tasks
Input: {
  title: string,
  description: string,
  assignees: string[],  // [uid]
  status: 'Todo' | 'In Progress' | 'Done',
  dueDate?: Timestamp
}
Output: {
  task: {
    id: string,
    title: string,
    description: string,
    status: string,
    assignees: string[],
    createdBy: string,
    createdAt: Timestamp
  }
}
```

**On Success**:
- Close modal
- Socket broadcasts `task:created` to all board viewers

---

### 12. **Task Detail Modal** `/task/:tid` (or modal)
**Purpose**: View/edit task, attach files, view activity

**Tabs**:
1. **Details** - Edit task info
2. **Files** - P2P file sharing
3. **Activity** - Timeline of changes
4. **AI Summary** - Generate task summary

**Components Required**:

#### **Details Tab**
- Editable title (click to edit)
- Editable description (markdown editor)
- Assignee chips (add/remove)
- Status dropdown
- Due date picker
- "Delete Task" button (creator/admin only)

**Data Fetching**:
```typescript
// GET /api/tasks/:tid
{
  task: {
    id: string,
    title: string,
    description: string,
    status: string,
    assignees: [
      { uid: string, name: string, photoURL: string }
    ],
    createdBy: string,
    createdAt: Timestamp,
    updatedAt: Timestamp,
    dueDate?: Timestamp,
    fileIds: string[]
  }
}
```

#### **Files Tab**
- File list (metadata from server)
- "Share File" button (opens P2P uploader)
- File receivers list (who can download)

**Data Fetching**:
```typescript
// GET /api/tasks/:tid/files
{
  files: [
    {
      id: string,
      fileName: string,
      fileSize: number,
      fileType: string,
      sharedBy: string,
      allowedReceivers: string[],
      createdAt: Timestamp
    }
  ]
}
```

**P2P File Upload Flow**:
1. Click "Share File"
2. Select file from device
3. Select allowed receivers (multi-select)
4. Click "Share"
5. **Frontend**:
   ```typescript
   // Store metadata first
   POST /api/tasks/:tid/files/metadata
   Input: {
     fileName: string,
     fileSize: number,
     fileType: string,
     allowedReceivers: string[]
   }
   Output: {
     fileId: string,
     metadata: { /* file metadata */ }
   }
   
   // Then establish WebRTC connection
   const peerConnection = new RTCPeerConnection({
     iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
   });
   
   // Create data channel
   const dataChannel = peerConnection.createDataChannel('file-transfer');
   
   // Signal via Socket.io
   socket.emit('webrtc:signal', {
     type: 'offer',
     fileId: fileId,
     targetUsers: allowedReceivers,
     sdp: offer
   });
   
   // On answer received, send file chunks
   dataChannel.send(fileChunk);
   ```

**P2P File Download Flow**:
1. Receiver sees file in list
2. Click "Download"
3. **Frontend**:
   ```typescript
   // Accept WebRTC connection
   socket.on('webrtc:signal', async ({ type, fileId, sdp }) => {
     if (type === 'offer') {
       // Create answer
       await peerConnection.setRemoteDescription(sdp);
       const answer = await peerConnection.createAnswer();
       
       // Send answer back
       socket.emit('webrtc:signal', {
         type: 'answer',
         fileId: fileId,
         sdp: answer
       });
     }
   });
   
   // Receive file chunks
   dataChannel.onmessage = (event) => {
     // Reconstruct file from chunks
     fileChunks.push(event.data);
   };
   
   // Download complete
   const blob = new Blob(fileChunks);
   saveAs(blob, fileName);
   ```

#### **Activity Tab**
**Data Fetching**:
```typescript
// GET /api/tasks/:tid/activity
{
  activities: [
    {
      id: string,
      action: 'CREATED' | 'STATUS_CHANGED' | 'ASSIGNED' | 'COMMENTED',
      previousValue?: string,
      newValue: string,
      performedBy: string,
      performedByName: string,
      timestamp: Timestamp
    }
  ]
}
```

**Display**: Timeline showing who did what when

#### **AI Summary Tab**
- "Generate Summary" button
- Loading state
- Display summary in markdown

**API Call**:
```typescript
POST /api/ai/summarize/task/:tid
Input: (none, reads task data server-side)
Output: {
  summary: string,  // Markdown formatted
  generatedAt: Timestamp
}
```

---

### 13. **AI Summary Modal**
**Purpose**: Display workspace/sub-room summary

**Trigger**: Click "Generate AI Summary" button

**API Call**:
```typescript
POST /api/ai/summarize/workspace/:wid
// OR
POST /api/ai/summarize/subroom/:sid

Output: {
  summary: {
    title: string,
    overallProgress: string,
    keyUpdates: string[],
    blockedTasks: string[],
    topContributors: [
      { name: string, actionsCount: number }
    ],
    nextSteps: string[]
  }
}
```

**Display**:
- Markdown renderer
- Copy to clipboard button
- Export as PDF button

---

### 14. **Task Extractor Modal**
**Purpose**: Extract tasks from chat/meeting notes

**Form Fields**:
- Text input (large textarea)
- "Extract Tasks" button

**API Call**:
```typescript
POST /api/ai/extract-tasks
Input: {
  text: string,
  subroomId: string
}
Output: {
  suggestedTasks: [
    {
      title: string,
      description: string,
      suggestedAssignee?: string
    }
  ]
}
```

**Display**:
- List of suggested tasks (editable)
- Checkboxes to select which to create
- "Create Selected Tasks" button

**On Confirm**:
```typescript
// Batch create tasks
POST /api/boards/:bid/tasks/batch
Input: {
  tasks: [
    { title: string, description: string, assignees: string[] }
  ]
}
```

---

### 15. **Calendar/Meeting Scheduler Modal**
**Purpose**: Schedule meeting and sync to Google Calendar

**Form Fields**:
- Meeting title (text input)
- Description (textarea)
- Date & time (datetime picker)
- Duration (select: 15m, 30m, 1h, 2h)
- Participants (multi-select from sub-room members)

**API Call**:
```typescript
POST /api/subrooms/:sid/meetings
Input: {
  title: string,
  description: string,
  startTime: Timestamp,
  duration: number,  // minutes
  participants: string[]  // [uid]
}
Output: {
  meeting: {
    id: string,
    title: string,
    startTime: Timestamp,
    endTime: Timestamp,
    googleCalendarEventId: string,
    googleCalendarLink: string
  }
}
```

**Display**:
- Success toast with Google Calendar link
- "Open in Google Calendar" button

---

### 16. **Settings Page** `/settings`
**Purpose**: User preferences

**Sections**:
- Profile (name, email, photo - read-only from Google)
- Notifications (toggle email/push notifications)
- Privacy (view session logs)
- Danger zone (delete account)

**API Calls**:
```typescript
// Get user sessions
GET /api/sessions/user/:uid
Output: {
  sessions: [
    {
      sessionId: string,
      workspaceName: string,
      roomName: string,
      subroomName: string,
      entryTime: Timestamp,
      exitTime: Timestamp,
      actionsCount: number
    }
  ]
}
```

---

### 17. **Workspace Settings** `/workspace/:wid/settings` (Admin Only)
**Purpose**: Manage workspace

**Sections**:
- General (workspace name, description)
- Members (list, add, remove, change roles)
- Invite Code (regenerate, view)
- Danger Zone (delete workspace)

**API Calls**:
```typescript
// Get members
GET /api/workspaces/:wid/members
Output: {
  members: [
    {
      uid: string,
      name: string,
      email: string,
      photoURL: string,
      role: 'admin' | 'member',
      joinedAt: Timestamp
    }
  ]
}

// Add member
POST /api/workspaces/:wid/members
Input: { uid: string, role: 'admin' | 'member' }

// Remove member
DELETE /api/workspaces/:wid/members/:uid

// Regenerate invite code
POST /api/workspaces/:wid/invite
Output: { inviteCode: string }
```

---

## 🔌 BACKEND API ROUTES (Detailed)

### **Authentication Routes**

#### `POST /api/auth/login`
**Purpose**: Create user session after OAuth

**Input**:
```typescript
{
  firebaseToken: string  // From Firebase Auth
}
```

**Process**:
1. Verify Firebase token
2. Get/create user in Firestore
3. Generate session UUID
4. Store session metadata

**Output**:
```typescript
{
  user: {
    uid: string,
    name: string,
    email: string,
    photoURL: string,
    workspaceId?: string
  },
  sessionId: string
}
```

**Status Codes**:
- 200: Success
- 401: Invalid token
- 500: Server error

---

#### `POST /api/auth/logout`
**Purpose**: End user session

**Input**:
```typescript
{
  sessionId: string
}
```

**Process**:
1. Update session exitTime
2. Clear auth tokens

**Output**:
```typescript
{
  message: 'Logged out successfully'
}
```

---

#### `GET /api/auth/me`
**Purpose**: Get current user info

**Headers**:
```
Authorization: Bearer <firebaseToken>
```

**Output**:
```typescript
{
  user: {
    uid: string,
    name: string,
    email: string,
    photoURL: string,
    workspaceId?: string
  }
}
```

---

### **Workspace Routes**

#### `POST /api/workspaces`
**Purpose**: Create new workspace

**Input**:
```typescript
{
  name: string
}
```

**Process**:
1. Generate invite code (UUID)
2. Create workspace in Firestore
3. Add creator as admin

**Output**:
```typescript
{
  workspace: {
    id: string,
    name: string,
    inviteCode: string,
    createdBy: string,
    members: {
      [uid]: 'admin'
    },
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/workspaces`
**Purpose**: Get user's workspaces

**Headers**:
```
Authorization: Bearer <firebaseToken>
```

**Output**:
```typescript
{
  workspaces: [
    {
      id: string,
      name: string,
      role: 'admin' | 'member',
      memberCount: number,
      roomCount: number,
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/workspaces/:wid`
**Purpose**: Get workspace details

**Output**:
```typescript
{
  workspace: {
    id: string,
    name: string,
    inviteCode: string,
    createdBy: string,
    members: {
      [uid]: 'admin' | 'member'
    },
    createdAt: Timestamp
  }
}
```

---

#### `POST /api/workspaces/join`
**Purpose**: Join workspace via invite code

**Input**:
```typescript
{
  inviteCode: string
}
```

**Process**:
1. Find workspace by invite code
2. Add user to members

**Output**:
```typescript
{
  workspace: {
    id: string,
    name: string
  }
}
```

**Status Codes**:
- 200: Success
- 404: Invalid invite code
- 409: Already a member

---

#### `POST /api/workspaces/:wid/invite`
**Purpose**: Regenerate invite code (admin only)

**Process**:
1. Generate new UUID
2. Update workspace

**Output**:
```typescript
{
  inviteCode: string
}
```

---

#### `GET /api/workspaces/:wid/members`
**Purpose**: List workspace members

**Output**:
```typescript
{
  members: [
    {
      uid: string,
      name: string,
      email: string,
      photoURL: string,
      role: 'admin' | 'member',
      joinedAt: Timestamp,
      lastActive: Timestamp
    }
  ]
}
```

---

### **Room Routes**

#### `POST /api/workspaces/:wid/rooms`
**Purpose**: Create client room

**Input**:
```typescript
{
  name: string,
  description?: string
}
```

**Output**:
```typescript
{
  room: {
    id: string,
    workspaceId: string,
    name: string,
    description: string,
    createdBy: string,
    members: {
      [uid]: 'admin'
    },
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/workspaces/:wid/rooms`
**Purpose**: List all rooms in workspace

**Output**:
```typescript
{
  rooms: [
    {
      id: string,
      name: string,
      description: string,
      subroomCount: number,
      memberCount: number,
      lastActivity: Timestamp,
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/rooms/:rid`
**Purpose**: Get room details

**Output**:
```typescript
{
  room: {
    id: string,
    workspaceId: string,
    name: string,
    description: string,
    members: {
      [uid]: 'admin' | 'member'
    },
    createdAt: Timestamp
  }
}
```

---

#### `PUT /api/rooms/:rid`
**Purpose**: Update room

**Input**:
```typescript
{
  name?: string,
  description?: string
}
```

**Output**:
```typescript
{
  room: {
    id: string,
    name: string,
    description: string,
    updatedAt: Timestamp
  }
}
```

---

#### `DELETE /api/rooms/:rid`
**Purpose**: Delete room (admin only)

**Process**:
1. Delete all sub-rooms
2. Delete room

**Output**:
```typescript
{
  message: 'Room deleted successfully'
}
```

---

### **Sub-Room Routes**

#### `POST /api/rooms/:rid/subrooms`
**Purpose**: Create sub-room

**Input**:
```typescript
{
  name: string
}
```

**Output**:
```typescript
{
  subroom: {
    id: string,
    roomId: string,
    workspaceId: string,
    name: string,
    createdBy: string,
    members: {
      [uid]: 'admin'
    },
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/rooms/:rid/subrooms`
**Purpose**: List sub-rooms in room

**Output**:
```typescript
{
  subrooms: [
    {
      id: string,
      name: string,
      projectCount: number,
      activeTaskCount: number,
      lastActivity: Timestamp,
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/subrooms/:sid`
**Purpose**: Get sub-room details

**Output**:
```typescript
{
  subroom: {
    id: string,
    roomId: string,
    workspaceId: string,
    name: string,
    members: {
      [uid]: 'admin' | 'member'
    },
    createdAt: Timestamp
  }
}
```

---

### **Project & Board Routes**

#### `POST /api/subrooms/:sid/projects`
**Purpose**: Create project (auto-creates board)

**Input**:
```typescript
{
  name: string,
  description: string
}
```

**Process**:
1. Create project
2. Auto-create board with 3 columns

**Output**:
```typescript
{
  project: {
    id: string,
    subroomId: string,
    name: string,
    description: string,
    boardId: string,
    createdBy: string,
    createdAt: Timestamp
  },
  board: {
    id: string,
    projectId: string,
    columns: ['Todo', 'In Progress', 'Done']
  }
}
```

---

#### `GET /api/subrooms/:sid/projects`
**Purpose**: List projects in sub-room

**Output**:
```typescript
{
  projects: [
    {
      id: string,
      name: string,
      description: string,
      boardId: string,
      taskCount: number,
      completedTasks: number,
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/boards/:bid`
**Purpose**: Get board with all tasks

**Output**:
```typescript
{
  board: {
    id: string,
    projectId: string,
    columns: ['Todo', 'In Progress', 'Done']
  },
  tasks: [
    {
      id: string,
      title: string,
      description: string,
      status: 'Todo' | 'In Progress' | 'Done',
      assignees: [
        {
          uid: string,
          name: string,
          photoURL: string
        }
      ],
      fileCount: number,
      createdBy: string,
      createdAt: Timestamp,
      updatedAt: Timestamp,
      dueDate?: Timestamp
    }
  ]
}
```

---

### **Task Routes**

#### `POST /api/boards/:bid/tasks`
**Purpose**: Create task

**Input**:
```typescript
{
  title: string,
  description: string,
  status: 'Todo' | 'In Progress' | 'Done',
  assignees: string[],  // [uid]
  dueDate?: Timestamp
}
```

**Process**:
1. Create task in Firestore
2. Create activity log (CREATED)
3. Emit socket event `task:created`

**Output**:
```typescript
{
  task: {
    id: string,
    boardId: string,
    title: string,
    description: string,
    status: string,
    assignees: string[],
    fileIds: [],
    createdBy: string,
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/boards/:bid/tasks`
**Purpose**: Get all tasks in board

**Query Params**:
- `status` (optional): Filter by status

**Output**:
```typescript
{
  tasks: [
    {
      id: string,
      title: string,
      status: string,
      assignees: [...],
      fileCount: number,
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/tasks/:tid`
**Purpose**: Get task details

**Output**:
```typescript
{
  task: {
    id: string,
    boardId: string,
    projectId: string,
    subroomId: string,
    title: string,
    description: string,
    status: string,
    assignees: [
      {
        uid: string,
        name: string,
        photoURL: string
      }
    ],
    fileIds: string[],
    createdBy: string,
    createdAt: Timestamp,
    updatedAt: Timestamp,
    dueDate?: Timestamp
  }
}
```

---

#### `PUT /api/tasks/:tid`
**Purpose**: Update task

**Input**:
```typescript
{
  title?: string,
  description?: string,
  assignees?: string[],
  dueDate?: Timestamp
}
```

**Process**:
1. Update task
2. Create activity log
3. Emit socket event `task:updated`

**Output**:
```typescript
{
  task: {
    id: string,
    ...updatedFields,
    updatedAt: Timestamp
  }
}
```

---

#### `PATCH /api/tasks/:tid/status`
**Purpose**: Update task status (drag-and-drop)

**Input**:
```typescript
{
  status: 'Todo' | 'In Progress' | 'Done',
  previousStatus: string
}
```

**Process**:
1. Update task status
2. Create activity log (STATUS_CHANGED)
3. Emit socket event `task:moved`

**Output**:

```typescript
{
  task: {
    id: string,
    status: string,
    updatedAt: Timestamp
  }
}
```

---

#### `POST /api/tasks/:tid/assign`
**Purpose**: Assign user to task

**Input**:
```typescript
{
  userId: string
}
```

**Process**:
1. Add user to assignees
2. Create activity log (ASSIGNED)
3. Emit socket event

**Output**:
```typescript
{
  task: {
    id: string,
    assignees: string[]
  }
}
```

---

#### `DELETE /api/tasks/:tid`
**Purpose**: Delete task

**Process**:
1. Delete task
2. Delete associated files metadata
3. Emit socket event

**Output**:
```typescript
{
  message: 'Task deleted successfully'
}
```

---

#### `GET /api/tasks/:tid/activity`
**Purpose**: Get task activity log

**Output**:
```typescript
{
  activities: [
    {
      id: string,
      taskId: string,
      action: 'CREATED' | 'STATUS_CHANGED' | 'ASSIGNED' | 'COMMENTED',
      previousValue?: string,
      newValue: string,
      performedBy: string,
      performedByName: string,
      performedByPhoto: string,
      sessionId: string,
      timestamp: Timestamp
    }
  ]
}
```

---

#### `POST /api/boards/:bid/tasks/batch`
**Purpose**: Create multiple tasks (from AI extraction)

**Input**:
```typescript
{
  tasks: [
    {
      title: string,
      description: string,
      assignees: string[]
    }
  ]
}
```

**Output**:
```typescript
{
  tasks: [
    {
      id: string,
      title: string,
      status: 'Todo',
      createdAt: Timestamp
    }
  ]
}
```

---

### **File Routes (Metadata Only)**

#### `POST /api/tasks/:tid/files/metadata`
**Purpose**: Store file metadata before P2P transfer

**Input**:
```typescript
{
  fileName: string,
  fileSize: number,
  fileType: string,
  allowedReceivers: string[]  // [uid]
}
```

**Process**:
1. Store metadata in Firestore
2. Return file ID for WebRTC signaling

**Output**:
```typescript
{
  file: {
    id: string,
    taskId: string,
    fileName: string,
    fileSize: number,
    fileType: string,
    sharedBy: string,
    allowedReceivers: string[],
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/tasks/:tid/files`
**Purpose**: Get file metadata for task

**Output**:
```typescript
{
  files: [
    {
      id: string,
      fileName: string,
      fileSize: number,
      fileType: string,
      sharedBy: string,
      sharedByName: string,
      allowedReceivers: string[],
      createdAt: Timestamp
    }
  ]
}
```

---

#### `GET /api/files/:fid/receivers`
**Purpose**: Check if user can download file

**Output**:
```typescript
{
  canDownload: boolean,
  allowedReceivers: [
    {
      uid: string,
      name: string,
      online: boolean
    }
  ]
}
```

---

#### `DELETE /api/files/:fid`
**Purpose**: Delete file metadata

**Process**:
1. Delete from Firestore
2. Notify peers via socket (stop transfer if ongoing)

**Output**:
```typescript
{
  message: 'File metadata deleted'
}
```

---

### **Chat Routes**

#### `POST /api/subrooms/:sid/messages`
**Purpose**: Send message (backup to socket)

**Input**:
```typescript
{
  text: string,
  sessionId: string
}
```

**Process**:
1. Store message in Firestore
2. Emit socket event `chat:message`

**Output**:
```typescript
{
  message: {
    id: string,
    subroomId: string,
    text: string,
    senderId: string,
    senderName: string,
    senderPhoto: string,
    sessionId: string,
    timestamp: Timestamp
  }
}
```

---

#### `GET /api/subrooms/:sid/messages`
**Purpose**: Get chat history

**Query Params**:
- `limit` (default: 50)
- `before` (Timestamp for pagination)

**Output**:
```typescript
{
  messages: [
    {
      id: string,
      text: string,
      senderId: string,
      senderName: string,
      senderPhoto: string,
      timestamp: Timestamp
    }
  ],
  hasMore: boolean
}
```

---

### **AI Routes**

#### `POST /api/ai/summarize/workspace/:wid`
**Purpose**: Generate workspace summary

**Process**:
1. Fetch all tasks, activity logs, sessions
2. Send to Groq LLM via LangChain
3. Generate structured summary

**Output**:
```typescript
{
  summary: {
    title: string,
    period: string,  // e.g., "Last 7 days"
    overallProgress: string,
    keyUpdates: string[],
    blockedTasks: [
      {
        taskId: string,
        title: string,
        reason: string
      }
    ],
    topContributors: [
      {
        name: string,
        actionsCount: number,
        contribution: string
      }
    ],
    nextSteps: string[],
    generatedAt: Timestamp
  }
}
```

**Status Codes**:
- 200: Success
- 429: Rate limit exceeded
- 500: AI service error

---

#### `POST /api/ai/summarize/subroom/:sid`
**Purpose**: Generate sub-room summary

**Output**: Same structure as workspace summary

---

#### `POST /api/ai/summarize/task/:tid`
**Purpose**: Generate task summary

**Output**:
```typescript
{
  summary: {
    title: string,
    description: string,
    progressSummary: string,
    recentActivities: string[],
    blockers: string[],
    suggestions: string[],
    generatedAt: Timestamp
  }
}
```

---

#### `POST /api/ai/extract-tasks`
**Purpose**: Extract tasks from text

**Input**:
```typescript
{
  text: string,  // Chat messages or meeting notes
  subroomId: string
}
```

**Process**:
1. Send text to Groq LLM
2. Extract action items
3. Return structured tasks

**Output**:
```typescript
{
  suggestedTasks: [
    {
      title: string,
      description: string,
      priority: 'high' | 'medium' | 'low',
      suggestedAssignee?: string,
      confidence: number  // 0-1
    }
  ]
}
```

---

### **Calendar Routes**

#### `POST /api/subrooms/:sid/meetings`
**Purpose**: Create meeting and sync to Google Calendar

**Input**:
```typescript
{
  title: string,
  description: string,
  startTime: Timestamp,
  duration: number,  // minutes
  participants: string[]  // [uid]
}
```

**Process**:
1. Create meeting in Firestore
2. Create event in Google Calendar via API
3. Return meeting + calendar link

**Output**:
```typescript
{
  meeting: {
    id: string,
    subroomId: string,
    title: string,
    description: string,
    startTime: Timestamp,
    endTime: Timestamp,
    participants: string[],
    googleCalendarEventId: string,
    googleCalendarLink: string,
    createdAt: Timestamp
  }
}
```

---

#### `GET /api/subrooms/:sid/meetings`
**Purpose**: List upcoming meetings

**Query Params**:
- `upcoming` (boolean): Only future meetings

**Output**:
```typescript
{
  meetings: [
    {
      id: string,
      title: string,
      startTime: Timestamp,
      endTime: Timestamp,
      participants: [
        {
          uid: string,
          name: string,
          photoURL: string
        }
      ],
      googleCalendarLink: string
    }
  ]
}
```

---

#### `PUT /api/meetings/:mid`
**Purpose**: Update meeting

**Input**:
```typescript
{
  title?: string,
  startTime?: Timestamp,
  duration?: number
}
```

**Process**:
1. Update in Firestore
2. Update in Google Calendar

**Output**:
```typescript
{
  meeting: {
    id: string,
    ...updatedFields,
    updatedAt: Timestamp
  }
}
```

---

#### `DELETE /api/meetings/:mid`
**Purpose**: Delete meeting

**Process**:
1. Delete from Firestore
2. Delete from Google Calendar

**Output**:
```typescript
{
  message: 'Meeting deleted successfully'
}
```

---

### **Session Routes**

#### `POST /api/sessions/start`
**Purpose**: Start user session (auto on login)

**Input**:
```typescript
{
  workspaceId: string,
  roomId?: string,
  subroomId?: string
}
```

**Output**:
```typescript
{
  session: {
    sessionId: string,  // UUID
    uid: string,
    workspaceId: string,
    roomId?: string,
    subroomId?: string,
    entryTime: Timestamp,
    createdAt: Timestamp
  }
}
```

---

#### `POST /api/sessions/:sid/end`
**Purpose**: End session

**Input**:
```typescript
{
  actionsCount: number  // Total actions in session
}
```

**Output**:
```typescript
{
  session: {
    sessionId: string,
    exitTime: Timestamp,
    duration: number,  // seconds
    actionsCount: number
  }
}
```

---

#### `GET /api/sessions/user/:uid`
**Purpose**: Get user's session history

**Query Params**:
- `limit` (default: 20)
- `workspaceId` (optional filter)

**Output**:
```typescript
{
  sessions: [
    {
      sessionId: string,
      workspaceName: string,
      roomName?: string,
      subroomName?: string,
      entryTime: Timestamp,
      exitTime?: Timestamp,
      duration: number,
      actionsCount: number
    }
  ]
}
```

---

#### `GET /api/sessions/subroom/:sid`
**Purpose**: Get sub-room session activity

**Output**:
```typescript
{
  sessions: [
    {
      sessionId: string,
      userName: string,
      userPhoto: string,
      entryTime: Timestamp,
      exitTime?: Timestamp,
      actionsCount: number
    }
  ],
  totalSessions: number,
  uniqueUsers: number
}
```

---

## 🔌 WebSocket Events (Socket.io)

### **Connection Events**

#### `join:workspace`
**Client → Server**
```typescript
{
  workspaceId: string
}
```

#### `join:room`
**Client → Server**
```typescript
{
  roomId: string
}
```

#### `join:subroom`
**Client → Server**
```typescript
{
  subroomId: string
}
```

#### `join:board`
**Client → Server**
```typescript
{
  boardId: string
}
```

---

### **Task Events**

#### `task:create`
**Client → Server**
```typescript
{
  boardId: string,
  title: string,
  description: string,
  status: string,
  assignees: string[]
}
```

#### `task:created`
**Server → All Clients in Board**
```typescript
{
  task: {
    id: string,
    title: string,
    status: string,
    createdBy: string,
    createdAt: Timestamp
  }
}
```

---

#### `task:update`
**Client → Server**
```typescript
{
  taskId: string,
  updates: {
    title?: string,
    description?: string,
    assignees?: string[]
  }
}
```

#### `task:updated`
**Server → All Clients**
```typescript
{
  task: {
    id: string,
    ...updatedFields,
    updatedBy: string,
    updatedAt: Timestamp
  }
}
```

---

#### `task:move`
**Client → Server** (drag-and-drop)
```typescript
{
  taskId: string,
  oldStatus: string,
  newStatus: string
}
```

#### `task:moved`
**Server → All Clients**
```typescript
{
  task: {
    id: string,
    status: string,
    movedBy: string,
    timestamp: Timestamp
  }
}
```

---

### **Chat Events**

#### `chat:message`
**Client → Server**
```typescript
{
  subroomId: string,
  text: string,
  sessionId: string
}
```

#### `chat:message`
**Server → All Clients in Sub-room**
```typescript
{
  message: {
    id: string,
    text: string,
    senderId: string,
    senderName: string,
    senderPhoto: string,
    timestamp: Timestamp
  }
}
```

---

### **WebRTC Signaling Events**

#### `webrtc:signal`
**Client → Server** (offer/answer/ice-candidate)
```typescript
{
  type: 'offer' | 'answer' | 'ice-candidate',
  fileId: string,
  targetUserId: string,
  sdp?: RTCSessionDescriptionInit,
  candidate?: RTCIceCandidateInit
}
```

#### `webrtc:signal`
**Server → Target Client**
```typescript
{
  type: 'offer' | 'answer' | 'ice-candidate',
  fileId: string,
  fromUserId: string,
  sdp?: RTCSessionDescriptionInit,
  candidate?: RTCIceCandidateInit
}
```

---

### **Presence Events**

#### `user:joined`
**Server → All Clients**
```typescript
{
  user: {
    uid: string,
    name: string,
    photoURL: string
  },
  location: 'workspace' | 'room' | 'subroom' | 'board',
  locationId: string
}
```

#### `user:left`
**Server → All Clients**
```typescript
{
  userId: string,
  location: string,
  locationId: string
}
```

---

## 📦 Key Frontend Hooks

### `useAuth.ts`
```typescript
export const useAuth = () => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (firebaseUser) => {
      if (firebaseUser) {
        // Fetch user from backend
        fetchUser(firebaseUser.uid).then(setUser);
      } else {
        setUser(null);
      }
      setLoading(false);
    });
    
    return unsubscribe;
  }, []);
  
  const login = async () => {
    const result = await signInWithPopup(auth, googleProvider);
    const token = await result.user.getIdToken();
    await axios.post('/api/auth/login', { firebaseToken: token });
  };
  
  const logout = async () => {
    await axios.post('/api/auth/logout');
    await signOut(auth);
  };
  
  return { user, loading, login, logout };
};
```

---

### `useSocket.ts`
```typescript
export const useSocket = (subroomId?: string) => {
  const [socket, setSocket] = useState<Socket | null>(null);
  
  useEffect(() => {
    const newSocket = io(process.env.VITE_SOCKET_URL);
    setSocket(newSocket);
    
    if (subroomId) {
      newSocket.emit('join:subroom', { subroomId });
    }
    
    return () => {
      newSocket.disconnect();
    };
  }, [subroomId]);
  
  return socket;
};
```

---

### `useWebRTC.ts`
```typescript
export const useWebRTC = (fileId: string, isReceiver: boolean) => {
  const [peerConnection, setPeerConnection] = useState<RTCPeerConnection | null>(null);
  const [dataChannel, setDataChannel] = useState<RTCDataChannel | null>(null);
  const socket = useSocket();
  
  useEffect(() => {
    const pc = new RTCPeerConnection({
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    });
    
    setPeerConnection(pc);
    
    if (!isReceiver) {
      const channel = pc.createDataChannel('file-transfer');
      setDataChannel(channel);
    } else {
      pc.ondatachannel = (event) => {
        setDataChannel(event.channel);
      };
    }
    
    // Handle signaling
    pc.onicecandidate = (event) => {
      if (event.candidate) {
        socket?.emit('webrtc:signal', {
          type: 'ice-candidate',
          fileId,
          candidate: event.candidate
        });
      }
    };
    
    return () => {
      pc.close();
    };
  }, [fileId, isReceiver]);
  
  const sendFile = async (file: File) => {
    if (!dataChannel) return;
    
    const chunkSize = 16384;
    const fileReader = new FileReader();
    let offset = 0;
    
    fileReader.onload = (e) => {
      if (e.target?.result) {
        dataChannel.send(e.target.result as ArrayBuffer);
        offset += chunkSize;
        
        if (offset < file.size) {
          readSlice(offset);
        }
      }
    };
    
    const readSlice = (o: number) => {
      const slice = file.slice(o, o + chunkSize);
      fileReader.readAsArrayBuffer(slice);
    };
    
    readSlice(0);
  };
  
  return { peerConnection, dataChannel, sendFile };
};
```

---

### `useFileTransfer.ts`
```typescript
export const useFileTransfer = (taskId: string) => {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  
  const uploadFile = async (file: File, allowedReceivers: string[]) => {
    setUploading(true);
    
    // Step 1: Store metadata
    const { data } = await axios.post(`/api/tasks/${taskId}/files/metadata`, {
      fileName: file.name,
      fileSize: file.size,
      fileType: file.type,
      allowedReceivers
    });
    
    // Step 2: Establish WebRTC connection
    const { sendFile } = useWebRTC(data.file.id, false);
    
    // Step 3: Send file P2P
    await sendFile(file);
    
    setUploading(false);
    setProgress(100);
  };
  
  return { uploadFile, uploading, progress };
};
```

---

## 🎨 Key UI Components

### Task Card Component
```typescript
interface TaskCardProps {
  task: Task;
  onDragStart: () => void;
  onDragEnd: () => void;
}

const TaskCard: React.FC<TaskCardProps> = ({ task, onDragStart, onDragEnd }) => {
  return (
    <div
      draggable
      onDragStart={onDragStart}
      onDragEnd={onDragEnd}
      className="bg-white p-4 rounded-lg shadow hover:shadow-md cursor-move"
    >
      <h3 className="font-semibold">{task.title}</h3>
      <p className="text-sm text-gray-600 mt-2">{task.description}</p>
      
      <div className="flex items-center justify-between mt-4">
        <div className="flex -space-x-2">
          {task.assignees.map((user) => (
            <img
              key={user.uid}
              src={user.photoURL}
              alt={user.name}
              className="w-6 h-6 rounded-full border-2 border-white"
            />
          ))}
        </div>
        
        {task.fileCount > 0 && (
          <span className="text-xs text-gray-500 flex items-center">
            <PaperclipIcon className="w-4 h-4 mr-1" />
            {task.fileCount}
          </span>
        )}
      </div>
    </div>
  );
};
```


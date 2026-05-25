# Capstone Project Management System
## Database Schema Documentation

**Version:** 1.0  
**Author:** Varun Chimata  
**Date:** 22 May 2026  

---

## 1. Overview

This document describes the MongoDB database structure used in the Capstone Project Management System. The system supports:

- User authentication and role-based access
- Guide-student group management
- Join request workflow
- Task posting and file submissions
- Announcements

| Property | Value |
|---|---|
| Database Type | MongoDB |
| ODM | Mongoose |
| Architecture | MVC |

---

## 2. Collections Overview

| Collection | Purpose |
|---|---|
| users | Stores all user accounts — students, guides, and HOD |
| groups | Stores capstone groups created by guides |
| joinrequests | Stores student requests to join a group |
| tasks | Stores tasks posted by guides to their groups |
| submissions | Stores student file uploads against tasks |
| announcements | Stores guide announcements to their group |

---

## 3. Users Collection

**Collection Name:** users

**Purpose:**  
Stores authentication details, profile information, and role data for all users in the system — students, guides, and HOD.

**Schema Structure:**
```js
const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 8,
    select: false
  },
  role: {
    type: String,
    enum: ['student', 'guide', 'hod'],
    required: true
  },
  avatar: {
    type: String,
    default: null
  },
  department: {
    type: String,
    trim: true,
    default: null
  },
  groupId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Group',
    default: null
  },
  isActive: {
    type: Boolean,
    default: true
  },
  refreshToken: {
    type: String,
    select: false,
    default: null
  },
  passwordResetToken: {
    type: String,
    select: false
  },
  passwordResetExpires: {
    type: Date,
    select: false
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| name | String | Yes | — | Full name |
| email | String | Yes | — | Login email, unique |
| password | String | Yes | — | Bcrypt hashed, never returned |
| role | Enum | Yes | — | student / guide / hod |
| avatar | String | No | null | Profile picture URL |
| department | String | No | null | e.g. Computer Science |
| groupId | ObjectId | No | null | Ref to Group — students only |
| isActive | Boolean | No | true | Soft disable account |
| refreshToken | String | No | null | JWT refresh token, never returned |
| passwordResetToken | String | No | — | Reset flow token, never returned |
| passwordResetExpires | Date | No | — | Reset token expiry |
| createdAt | Date | auto | — | Account creation time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `email` — unique
- `role` — for role-based queries
- `groupId` — sparse, for student group lookup

**Security Rules:**
- Password stored using bcrypt hashing (12 rounds)
- Password excluded from all API responses via `select: false`
- refreshToken excluded from all API responses via `select: false`
- One student can only belong to one group at a time
- Role modification restricted to admin endpoints

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e123456",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "student",
  "avatar": null,
  "department": "Computer Science",
  "groupId": "6651e7b2d2a12d4f5e654321",
  "isActive": true,
  "createdAt": "2026-05-25T10:00:00.000Z",
  "updatedAt": "2026-05-25T11:00:00.000Z"
}
```

---

## 4. Groups Collection

**Collection Name:** groups

**Purpose:**  
Stores capstone groups created by guides. Each group can have a maximum of 4 students.

**Schema Structure:**
```js
const groupSchema = new mongoose.Schema({
  groupName: {
    type: String,
    required: true,
    trim: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  members: {
    type: [{
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    }],
    validate: {
      validator: function(members) {
        return members.length <= this.maxMembers
      },
      message: 'Group has reached maximum member limit'
    }
  },
  maxMembers: {
    type: Number,
    default: 4
  },
  projectTitle: {
    type: String,
    trim: true,
    default: ''
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| groupName | String | Yes | — | Display name of the group |
| createdBy | ObjectId | Yes | — | Ref to User (guide) |
| members | [ObjectId] | No | [] | Ref to User — max 4 students |
| maxMembers | Number | No | 4 | Maximum student capacity |
| projectTitle | String | No | '' | What the group is building |
| isActive | Boolean | No | true | Soft disable group |
| createdAt | Date | auto | — | Group creation time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `createdBy` — for fetching guide's own groups

**Business Rules:**
- Guide is stored in `createdBy` — NOT in members array
- `members.length` enforced at schema level AND service layer
- A guide can own multiple groups
- A student can only be in one group at a time

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e654321",
  "groupName": "Team Alpha",
  "createdBy": "6651e7b2d2a12d4f5e111111",
  "members": [
    "6651e7b2d2a12d4f5e123456",
    "6651e7b2d2a12d4f5e789012"
  ],
  "maxMembers": 4,
  "projectTitle": "AI Attendance System",
  "isActive": true,
  "createdAt": "2026-05-25T10:00:00.000Z",
  "updatedAt": "2026-05-25T11:00:00.000Z"
}
```

---

## 5. JoinRequests Collection

**Collection Name:** joinrequests

**Purpose:**  
Tracks student requests to join a group. Every request requires guide approval before the student is added to the group.

**Schema Structure:**
```js
const joinRequestSchema = new mongoose.Schema({
  requestedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  groupId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Group',
    required: true
  },
  status: {
    type: String,
    enum: ['pending', 'approved', 'rejected'],
    default: 'pending'
  },
  message: {
    type: String,
    trim: true,
    default: ''
  },
  decidedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    default: null
  },
  decidedAt: {
    type: Date,
    default: null
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| requestedBy | ObjectId | Yes | — | Ref to User (student) |
| groupId | ObjectId | Yes | — | Ref to Group |
| status | Enum | No | pending | pending / approved / rejected |
| message | String | No | '' | Optional note from student |
| decidedBy | ObjectId | No | null | Ref to User (guide who decided) |
| decidedAt | Date | No | null | When decision was made |
| createdAt | Date | auto | — | Request creation time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `(requestedBy, groupId)` — compound unique — one request per student per group

**Business Rules:**
- Student cannot send duplicate requests to the same group
- On approve: student `groupId` updated + pushed to `group.members`
- On reject: student `groupId` stays null
- Only the guide who owns the group can approve or reject

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e999888",
  "requestedBy": "6651e7b2d2a12d4f5e123456",
  "groupId": "6651e7b2d2a12d4f5e654321",
  "status": "approved",
  "message": "I would like to join your group",
  "decidedBy": "6651e7b2d2a12d4f5e111111",
  "decidedAt": "2026-05-25T12:00:00.000Z",
  "createdAt": "2026-05-25T10:00:00.000Z",
  "updatedAt": "2026-05-25T12:00:00.000Z"
}
```

---

## 6. Tasks Collection

**Collection Name:** tasks

**Purpose:**  
Stores tasks posted by guides to their groups. Students submit files against open tasks.

**Schema Structure:**
```js
const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    trim: true,
    default: ''
  },
  groupId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Group',
    required: true
  },
  postedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  attachmentUrl: {
    type: String,
    default: null
  },
  dueDate: {
    type: Date,
    default: null
  },
  status: {
    type: String,
    enum: ['open', 'closed'],
    default: 'open'
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| title | String | Yes | — | Task heading |
| description | String | No | '' | Full task details |
| groupId | ObjectId | Yes | — | Ref to Group |
| postedBy | ObjectId | Yes | — | Ref to User (guide) |
| attachmentUrl | String | No | null | Optional reference file URL |
| dueDate | Date | No | null | Submission deadline |
| status | Enum | No | open | open / closed |
| createdAt | Date | auto | — | Task creation time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `groupId` — for fetching all tasks in a group
- `postedBy` — for fetching all tasks by a guide

**Business Rules:**
- Students cannot submit to a closed task
- Only the guide who created the task can update or delete it
- Guide can attach a reference file via `attachmentUrl`

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e777666",
  "title": "Submit Literature Review",
  "description": "Upload a 10 page literature review in PDF format",
  "groupId": "6651e7b2d2a12d4f5e654321",
  "postedBy": "6651e7b2d2a12d4f5e111111",
  "attachmentUrl": null,
  "dueDate": "2026-06-01T00:00:00.000Z",
  "status": "open",
  "createdAt": "2026-05-25T10:00:00.000Z",
  "updatedAt": "2026-05-25T10:00:00.000Z"
}
```

---

## 7. Submissions Collection

**Collection Name:** submissions

**Purpose:**  
Stores student file uploads against specific tasks. One submission per student per task. Guide reviews and gives feedback directly on the submission document.

**Schema Structure:**
```js
const submissionSchema = new mongoose.Schema({
  taskId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Task',
    required: true
  },
  studentId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  groupId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Group',
    required: true
  },
  fileUrl: {
    type: String,
    required: true
  },
  fileName: {
    type: String,
    required: true
  },
  fileSize: {
    type: Number,
    required: true
  },
  publicId: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['submitted', 'reviewed'],
    default: 'submitted'
  },
  feedback: {
    type: String,
    default: null
  },
  feedbackAt: {
    type: Date,
    default: null
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| taskId | ObjectId | Yes | — | Ref to Task |
| studentId | ObjectId | Yes | — | Ref to User (student) |
| groupId | ObjectId | Yes | — | Ref to Group |
| fileUrl | String | Yes | — | Cloudinary file URL |
| fileName | String | Yes | — | Original file name |
| fileSize | Number | Yes | — | File size in bytes |
| publicId | String | Yes | — | Cloudinary ID for deletion |
| status | Enum | No | submitted | submitted / reviewed |
| feedback | String | No | null | Guide's written feedback |
| feedbackAt | Date | No | null | When feedback was given |
| createdAt | Date | auto | — | Submission time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `(taskId, studentId)` — compound unique — one submission per student per task
- `groupId` — for fetching all submissions in a group

**Business Rules:**
- One submission per student per task — resubmitting updates existing document
- Student cannot delete their submission
- Only the guide of that group can give feedback
- `publicId` required for Cloudinary file deletion on resubmit

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e555444",
  "taskId": "6651e7b2d2a12d4f5e777666",
  "studentId": "6651e7b2d2a12d4f5e123456",
  "groupId": "6651e7b2d2a12d4f5e654321",
  "fileUrl": "https://res.cloudinary.com/demo/raw/upload/v1/capstone/submissions/report.pdf",
  "fileName": "literature_review.pdf",
  "fileSize": 204800,
  "publicId": "capstone/submissions/literature_review",
  "status": "reviewed",
  "feedback": "Good work. Improve the introduction section.",
  "feedbackAt": "2026-05-25T14:00:00.000Z",
  "createdAt": "2026-05-25T10:00:00.000Z",
  "updatedAt": "2026-05-25T14:00:00.000Z"
}
```

---

## 8. Announcements Collection

**Collection Name:** announcements

**Purpose:**  
Stores announcements posted by guides to their groups. Pinned announcements appear at the top.

**Schema Structure:**
```js
const announcementSchema = new mongoose.Schema({
  groupId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Group',
    required: true
  },
  postedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  title: {
    type: String,
    required: true,
    trim: true
  },
  body: {
    type: String,
    required: true,
    trim: true
  },
  pinned: {
    type: Boolean,
    default: false
  }
}, { timestamps: true })
```

**Field Documentation:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| _id | ObjectId | auto | — | Unique MongoDB ID |
| groupId | ObjectId | Yes | — | Ref to Group |
| postedBy | ObjectId | Yes | — | Ref to User (guide) |
| title | String | Yes | — | Announcement headline |
| body | String | Yes | — | Full announcement content |
| pinned | Boolean | No | false | Pinned announcements show first |
| createdAt | Date | auto | — | Post time |
| updatedAt | Date | auto | — | Last update time |

**Indexes:**
- `groupId` — for fetching all announcements in a group

**Business Rules:**
- Only the guide who created the announcement can update or delete it
- Pinned announcements sorted to top on every fetch
- Both title and body are required — empty announcements not allowed

**Example Document:**
```json
{
  "_id": "6651e7b2d2a12d4f5e333222",
  "groupId": "6651e7b2d2a12d4f5e654321",
  "postedBy": "6651e7b2d2a12d4f5e111111",
  "title": "Progress Review Meeting",
  "body": "We have a progress review meeting tomorrow at 10am in Room 301. Please bring your updated reports.",
  "pinned": true,
  "createdAt": "2026-05-25T08:00:00.000Z",
  "updatedAt": "2026-05-25T08:00:00.000Z"
}
```

---

## 9. Relationships Map

```
User (guide)   ──── createdBy ────►  Group
User (student) ──── groupId ────────► Group
User (student) ──── requestedBy ───► JoinRequest ──── groupId ────► Group
User (guide)   ──── decidedBy ────►  JoinRequest
Group          ──── groupId ────────► Task
User (guide)   ──── postedBy ──────► Task
Task           ──── taskId ─────────► Submission
User (student) ──── studentId ─────► Submission
Group          ──── groupId ────────► Announcement
User (guide)   ──── postedBy ──────► Announcement
```

---

## 10. Index Summary

| Collection | Index | Type | Purpose |
|---|---|---|---|
| users | email | Unique | Fast login lookup |
| users | role | Regular | Role-based queries |
| users | groupId | Sparse | Student group lookup |
| groups | createdBy | Regular | Guide's own groups |
| joinrequests | (requestedBy, groupId) | Compound Unique | Prevent duplicate requests |
| tasks | groupId | Regular | Tasks per group |
| tasks | postedBy | Regular | Tasks by guide |
| submissions | (taskId, studentId) | Compound Unique | One submission per student per task |
| submissions | groupId | Regular | Submissions per group |
| announcements | groupId | Regular | Announcements per group |

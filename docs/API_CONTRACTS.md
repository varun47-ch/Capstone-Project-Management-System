# Capstone Project Management — API Contracts
**Version:** 1.0  
**Author:** Varun Chimata  
**Date:** 22 May 2026  
## Objects

### User Object
```
{
  _id: ObjectId
  name: string
  email: string
  role: string (student / guide / hod)
  department: string
  groupId: ObjectId
  avatar: string
  isActive: boolean
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

### Group Object
```
{
  _id: ObjectId
  groupName: string
  createdBy: <user_object>
  members: [<user_object>]
  maxMembers: integer
  projectTitle: string
  isActive: boolean
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

### JoinRequest Object
```
{
  _id: ObjectId
  requestedBy: <user_object>
  groupId: ObjectId
  status: string (pending / approved / rejected)
  message: string
  decidedBy: <user_object>
  decidedAt: datetime (iso 8601)
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

### Task Object
```
{
  _id: ObjectId
  title: string
  description: string
  groupId: ObjectId
  postedBy: <user_object>
  attachmentUrl: string
  dueDate: datetime (iso 8601)
  status: string (open / closed)
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

### Submission Object
```
{
  _id: ObjectId
  taskId: ObjectId
  studentId: <user_object>
  groupId: ObjectId
  fileUrl: string
  fileName: string
  fileSize: integer
  publicId: string
  status: string (submitted / reviewed)
  feedback: string
  feedbackAt: datetime (iso 8601)
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

### Announcement Object
```
{
  _id: ObjectId
  groupId: ObjectId
  postedBy: <user_object>
  title: string
  body: string
  pinned: boolean
  createdAt: datetime (iso 8601)
  updatedAt: datetime (iso 8601)
}
```

---

## Auth Routes

---

### POST /api/auth/register
Creates a new user account and returns the new object.

* URL Params: None
* Headers: Content-Type: application/json
* Data Params:
```
{
  name: string
  email: string
  password: string (min 8 chars)
  role: string (student / guide / hod)
}
```
* Success Response:
  * Code: 201
  * Content:
```
{
  success: true
  accessToken: string
  user: <user_object>
}
```
* Error Response:
  * Code: 400 Content: `{ success: false, message: "Missing or invalid fields" }`
  * Code: 409 Content: `{ success: false, message: "email already exists" }`

---

### POST /api/auth/login
Authenticates user and returns access token.

* URL Params: None
* Headers: Content-Type: application/json
* Data Params:
```
{
  email: string
  password: string
}
```
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  accessToken: string
  user: <user_object>
}
```
> Note: refreshToken is set automatically as httpOnly cookie

* Error Response:
  * Code: 400 Content: `{ success: false, message: "Missing fields" }`
  * Code: 401 Content: `{ success: false, message: "Invalid credentials" }`

---

### POST /api/auth/logout
Clears refresh token cookie and invalidates session.

* URL Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: None
* Success Response:
  * Code: 200 Content: `{ success: true, message: "Logged out" }`
* Error Response:
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

### POST /api/auth/refresh
Issues a new access token using the refresh token cookie.

* URL Params: None
* Headers: Content-Type: application/json
* Data Params: None
* Success Response:
  * Code: 200 Content: `{ success: true, accessToken: string }`
* Error Response:
  * Code: 401 Content: `{ success: false, message: "No refresh token" }` OR
  * Code: 401 Content: `{ success: false, message: "Token expired" }`

---

### GET /api/auth/me
Returns the currently logged in user's profile.

* URL Params: None
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200 Content: `{ success: true, user: <user_object> }`
* Error Response:
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

### PATCH /api/auth/me
Updates the currently logged in user's profile.

* URL Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: (all optional)
```
{
  name: string
  avatar: string
  department: string
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, user: <user_object> }`
* Error Response:
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

## Group Routes

---

### GET /api/groups
Returns groups based on role — students see all active groups, guides see own groups, HOD sees all.

* URL Params: None
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  groups: [
    {<group_object>},
    {<group_object>}
  ]
}
```
* Error Response:
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

### POST /api/groups
Creates a new group. Guide only.

* URL Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  groupName: string
  projectTitle: string (optional)
}
```
* Success Response:
  * Code: 201 Content: `{ success: true, group: <group_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "groupName is required" }`
  * Code: 403 Content: `{ success: false, message: "Access denied" }`

---

### GET /api/groups/:groupId
Returns the specified group with members and guide info.

* URL Params: Required `groupId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200 Content: `{ success: true, group: <group_object> }`
* Error Response:
  * Code: 404 Content: `{ success: false, message: "Group not found" }`
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

### PATCH /api/groups/:groupId
Updates the specified group. Guide only — must be group owner.

* URL Params: Required `groupId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: (all optional)
```
{
  groupName: string
  projectTitle: string
  isActive: boolean
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, group: <group_object> }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your group" }`
  * Code: 404 Content: `{ success: false, message: "Group not found" }`

---

### DELETE /api/groups/:groupId
Deletes the specified group. Guide only — must be group owner.

* URL Params: Required `groupId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: None
* Success Response:
  * Code: 200 Content: `{ success: true, message: "Group deleted" }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your group" }`
  * Code: 404 Content: `{ success: false, message: "Group not found" }`

---

## Join Request Routes

---

### POST /api/groups/:groupId/join-request
Student sends a request to join the specified group.

* URL Params: Required `groupId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: (optional)
```
{
  message: string
}
```
* Success Response:
  * Code: 201 Content: `{ success: true, request: <joinrequest_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "You are already in a group" }` OR
  * Code: 400 Content: `{ success: false, message: "Group is full" }` OR
  * Code: 404 Content: `{ success: false, message: "Group not found" }` OR
  * Code: 409 Content: `{ success: false, message: "Request already sent" }`

---

### GET /api/groups/:groupId/join-requests
Returns all pending join requests for a group. Guide only.

* URL Params: Required `groupId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  requests: [
    {<joinrequest_object>},
    {<joinrequest_object>}
  ]
}
```
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your group" }`

---

### PATCH /api/groups/:groupId/join-requests/:requestId/decide
Guide approves or rejects a join request.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `requestId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  decision: string (approved / rejected)
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, request: <joinrequest_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "Request already decided" }` OR
  * Code: 400 Content: `{ success: false, message: "Group is full" }` OR
  * Code: 403 Content: `{ success: false, message: "Not your group" }` OR
  * Code: 404 Content: `{ success: false, message: "Request not found" }`

---

## Task Routes

---

### POST /api/groups/:groupId/tasks
Guide posts a new task to the group.

* URL Params: Required `groupId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  title: string
  description: string (optional)
  dueDate: datetime (optional)
}
```
* Success Response:
  * Code: 201 Content: `{ success: true, task: <task_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "title is required" }`
  * Code: 403 Content: `{ success: false, message: "Not your group" }`

---

### GET /api/groups/:groupId/tasks
Returns all tasks for the specified group.

* URL Params: Required `groupId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  tasks: [
    {<task_object>},
    {<task_object>}
  ]
}
```
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Access denied" }`
  * Code: 404 Content: `{ success: false, message: "Group not found" }`

---

### GET /api/groups/:groupId/tasks/:taskId
Returns the specified task.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200 Content: `{ success: true, task: <task_object> }`
* Error Response:
  * Code: 404 Content: `{ success: false, message: "Task not found" }`
  * Code: 401 Content: `{ success: false, message: "Unauthorized" }`

---

### PATCH /api/groups/:groupId/tasks/:taskId
Updates the specified task. Guide only — must be task owner.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: (all optional)
```
{
  title: string
  description: string
  dueDate: datetime
  status: string (open / closed)
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, task: <task_object> }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your task" }`
  * Code: 404 Content: `{ success: false, message: "Task not found" }`

---

### DELETE /api/groups/:groupId/tasks/:taskId
Deletes the specified task. Guide only — must be task owner.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: None
* Success Response:
  * Code: 200 Content: `{ success: true, message: "Task deleted" }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your task" }`
  * Code: 404 Content: `{ success: false, message: "Task not found" }`

---

## Submission Routes

---

### POST /api/groups/:groupId/tasks/:taskId/submission
Student uploads a file for the specified task. Resubmitting replaces existing.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Headers: Content-Type: multipart/form-data Authorization: Bearer `<token>`
* Data Params: `file=[file] (pdf / doc / docx / zip — max 20MB)`
* Success Response:
  * Code: 201 Content: `{ success: true, submission: <submission_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "No file uploaded" }` OR
  * Code: 400 Content: `{ success: false, message: "Task is closed" }` OR
  * Code: 400 Content: `{ success: false, message: "You are not in this group" }`

---

### GET /api/groups/:groupId/tasks/:taskId/submission
Returns the logged in student's submission for the specified task.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200 Content: `{ success: true, submission: <submission_object> }`
* Error Response:
  * Code: 404 Content: `{ success: false, message: "No submission found" }`

---

### PATCH /api/groups/:groupId/tasks/:taskId/submission
Student resubmits file — replaces existing submission.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Headers: Content-Type: multipart/form-data Authorization: Bearer `<token>`
* Data Params: `file=[file] (pdf / doc / docx / zip — max 20MB)`
* Success Response:
  * Code: 200 Content: `{ success: true, submission: <submission_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "Task is closed" }`
  * Code: 404 Content: `{ success: false, message: "No submission found" }`

---

### GET /api/groups/:groupId/tasks/:taskId/submissions
Returns all student submissions for a task. Guide only.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  submissions: [
    {<submission_object>},
    {<submission_object>}
  ]
}
```
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your group" }`

---

### PATCH /api/groups/:groupId/tasks/:taskId/submissions/:subId
Guide gives feedback on a student submission.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `taskId=[ObjectId]`
  * Required `subId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  feedback: string
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, submission: <submission_object> }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your group" }`
  * Code: 404 Content: `{ success: false, message: "Submission not found" }`

---

## Announcement Routes

---

### POST /api/groups/:groupId/announcements
Guide posts an announcement to the group.

* URL Params: Required `groupId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  title: string
  body: string
  pinned: boolean (optional, default: false)
}
```
* Success Response:
  * Code: 201 Content: `{ success: true, announcement: <announcement_object> }`
* Error Response:
  * Code: 400 Content: `{ success: false, message: "title and body are required" }`
  * Code: 403 Content: `{ success: false, message: "Not your group" }`

---

### GET /api/groups/:groupId/announcements
Returns all announcements for a group. Pinned announcements returned first.

* URL Params: Required `groupId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  announcements: [
    {<announcement_object>},
    {<announcement_object>}
  ]
}
```
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Access denied" }`

---

### PATCH /api/groups/:groupId/announcements/:announcementId
Updates the specified announcement. Guide only — must be announcement owner.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `announcementId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: (all optional)
```
{
  title: string
  body: string
  pinned: boolean
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, announcement: <announcement_object> }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your announcement" }`
  * Code: 404 Content: `{ success: false, message: "Announcement not found" }`

---

### DELETE /api/groups/:groupId/announcements/:announcementId
Deletes the specified announcement. Guide only — must be announcement owner.

* URL Params:
  * Required `groupId=[ObjectId]`
  * Required `announcementId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params: None
* Success Response:
  * Code: 200 Content: `{ success: true, message: "Announcement deleted" }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Not your announcement" }`
  * Code: 404 Content: `{ success: false, message: "Announcement not found" }`

---

## HOD Routes

---

### GET /api/users
Returns all users in the system. HOD only.

* URL Params: None
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200
  * Content:
```
{
  success: true
  users: [
    {<user_object>},
    {<user_object>}
  ]
}
```
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Access denied" }`

---

### GET /api/users/:userId
Returns the specified user. HOD only.

* URL Params: Required `userId=[ObjectId]`
* Data Params: None
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Success Response:
  * Code: 200 Content: `{ success: true, user: <user_object> }`
* Error Response:
  * Code: 404 Content: `{ success: false, message: "User not found" }` OR
  * Code: 403 Content: `{ success: false, message: "Access denied" }`

---

### PATCH /api/users/:userId/status
Activates or deactivates a user account. HOD only.

* URL Params: Required `userId=[ObjectId]`
* Headers: Content-Type: application/json Authorization: Bearer `<token>`
* Data Params:
```
{
  isActive: boolean
}
```
* Success Response:
  * Code: 200 Content: `{ success: true, message: "User deactivated" }`
* Error Response:
  * Code: 403 Content: `{ success: false, message: "Access denied" }`
  * Code: 404 Content: `{ success: false, message: "User not found" }`

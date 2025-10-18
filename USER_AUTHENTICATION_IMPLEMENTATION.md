# User Authentication & Authorization Implementation

## Overview
This document explains how user-specific note filtering has been implemented so that each user can only see and manage their own notes.

## Changes Made

### 1. **New Auth Utility** (`backend/lib/auth.ts`)
Created a JWT verification utility that:
- Extracts and verifies the JWT token from the `Authorization` header
- Decodes the token to get the `userId`
- Returns `null` if the token is invalid or missing

### 2. **Updated Notes API** (`backend/pages/api/notes/index.ts`)
- ✅ **Authentication Required**: All requests now require a valid JWT token
- ✅ **GET /api/notes**: Returns only notes where `userId` matches the logged-in user
- ✅ **POST /api/notes**: Automatically adds `userId` to new notes when created
- ✅ **401 Unauthorized**: Returns error if no valid token is provided

**Database Query Changes:**
```typescript
// Before: Returns ALL notes
const notes = await db.collection("new-coll").find({}).toArray();

// After: Returns only user's notes
const notes = await db.collection("new-coll").find({ userId }).toArray();
```

### 3. **Updated Individual Note API** (`backend/pages/api/notes/[id].ts`)
- ✅ **Authentication Required**: All requests require a valid JWT token
- ✅ **Ownership Verification**: Checks that the note belongs to the logged-in user
- ✅ **GET /api/notes/:id**: Only returns the note if it belongs to the user
- ✅ **PUT /api/notes/:id**: Only updates if the user owns the note
- ✅ **DELETE /api/notes/:id**: Only deletes if the user owns the note
- ✅ **404 Not Found**: Returns error if note doesn't exist or user doesn't own it

**Security Check:**
```typescript
// Verify ownership before any operation
const note = await db.collection("new-coll").findOne({ 
  _id: new ObjectId(id), 
  userId 
});

if (!note) {
  return res.status(404).json({ message: "Note not found or access denied" });
}
```

### 4. **Updated Analytics Trigger** (`backend/pages/api/analytics/trigger.ts`)
- ✅ **Authentication Required**: Requires valid JWT token
- ✅ **Ownership Verification**: Verifies the user owns the note before triggering analytics

## How It Works

### Authentication Flow:

1. **User logs in** → `/api/auth/login` returns a JWT token
2. **Frontend stores token** in localStorage
3. **Frontend sends token** in every API request via `Authorization: Bearer <token>` header
4. **Backend verifies token** using `verifyToken()` function
5. **Backend extracts userId** from the decoded token
6. **Backend filters data** by userId

### Data Model:

**Note Document Structure:**
```json
{
  "_id": "ObjectId",
  "userId": "ObjectId",  // ← NEW: Links note to user
  "title": "string",
  "content": "string",
  "createdAt": "Date",
  "updatedAt": "Date"
}
```

## Security Benefits

✅ **Complete Isolation**: Users can only see their own notes
✅ **No Unauthorized Access**: Users cannot read, update, or delete other users' notes
✅ **Token Verification**: All protected routes verify JWT authenticity
✅ **Ownership Checks**: Double-checks ownership for individual note operations

## Testing the Implementation

### Test Case 1: Create Notes as Different Users
1. Login as User A → Create notes
2. Logout and login as User B → Create notes
3. User A should only see their notes
4. User B should only see their notes

### Test Case 2: Try to Access Another User's Note
1. Login as User A → Note the ID of one of their notes
2. Logout and login as User B
3. Try to GET/PUT/DELETE User A's note by ID
4. Should receive `404 Not Found` or `access denied`

### Test Case 3: Unauthorized Access
1. Clear localStorage (remove token)
2. Try to access `/api/notes`
3. Should receive `401 Unauthorized`

## Important Notes

⚠️ **Existing Notes**: Notes created before this implementation won't have a `userId` field. You have two options:
1. Delete all existing notes and create new ones
2. Run a migration script to add `userId` to existing notes

⚠️ **Frontend Already Configured**: The frontend already sends the token in the Authorization header via the axios interceptor in `frontend/lib/api.ts`

## Migration Script (Optional)

If you have existing notes without userId, you can assign them to a specific user:

```javascript
// Run this in MongoDB shell or create a migration script
db.getCollection('new-coll').updateMany(
  { userId: { $exists: false } },
  { $set: { userId: ObjectId('YOUR_USER_ID_HERE') } }
);
```

## Next Steps

1. ✅ Implementation complete
2. 🔄 Restart your backend server
3. 🧪 Test with multiple user accounts
4. 📊 (Optional) Migrate existing notes if needed

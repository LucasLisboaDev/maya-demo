# n8n Workflow Setup for Attachment Profile

## Overview
The webpage now automatically polls for profile results after a call ends. Your n8n workflow needs to:
1. Receive data from ElevenLabs when a call ends
2. Store the profile with a session_id
3. Respond to queries from the webpage

## How it Works

### 1. Webpage generates a session ID
- When the page loads, it creates a unique `session_id` (e.g., `session_1699564123_abc123`)
- This is stored in sessionStorage and logged to console

### 2. ElevenLabs sends webhook data
- When a call ends, ElevenLabs POSTs to your webhook
- Data includes `data_collection_results.eq_analysis.value` with the profile type
- Also includes `dynamic_variables.system__conversation_id`

### 3. Webpage polls for results
- After call ends, webpage POSTs every 5 seconds: `{ "session_id": "session_...", "action": "get_profile" }`
- Polls up to 60 times (5 minutes)
- When profile is found, displays it automatically

## n8n Workflow Structure

### Option A: Using n8n Database (Simplest)

```
Webhook Trigger (POST)
  ↓
IF node (check action field)
  ↓
├─ action == "get_profile"
│   ↓
│   Code node: Extract session_id from body
│   ↓
│   Database: SELECT profile FROM profiles WHERE session_id = ?
│   ↓
│   IF found → Return profile JSON
│   IF not found → Return 404
│
└─ No action (ElevenLabs webhook)
    ↓
    Code node: Extract profile_type from data_collection_results.eq_analysis.value
    ↓
    Code node: Parse incoming query params or body for session_id
    ↓
    Database: INSERT INTO profiles (session_id, profile_type, conversation_id, created_at)
    ↓
    Respond 200 OK
```

### Option B: Using Redis/Cache

Replace database with Redis SET/GET operations.

### Option C: Using n8n's built-in database

Use the "Postgres" node pointing to n8n's internal DB.

## Required n8n Nodes

### 1. Webhook Trigger
- Method: POST
- Path: `/webhook/41fac57f-ff3d-4dcf-a4ac-9c76d995502d`
- Authentication: HMAC with your secret
- Response Mode: "Respond to Webhook"

### 2. IF Node - Check Request Type
```javascript
// Expression
{{ $json.body.action === 'get_profile' }}
```

### 3a. Handle GET Profile (True branch)
```javascript
// Code node to query database
const sessionId = $json.body.session_id;

// Query your database (example with Postgres)
// SELECT profile_type, conversation_id 
// FROM profiles 
// WHERE session_id = sessionId
// LIMIT 1

// If found, return:
return [{
  data_collection_results: {
    eq_analysis: {
      value: JSON.stringify({ profile_type: 'Secure' }) // Replace with actual value
    }
  }
}];

// If not found, return empty to trigger 404
```

### 3b. Handle Store Profile (False branch - ElevenLabs webhook)
```javascript
// Code node to extract and store
const profileValue = $json.body.data_collection_results?.eq_analysis?.value;
const conversationId = $json.body.dynamic_variables?.system__conversation_id;

// Parse profile_type from the string
let profileType = null;
if (typeof profileValue === 'string') {
  const parsed = JSON.parse(profileValue.replace(/'/g, '"'));
  profileType = parsed.profile_type;
} else if (profileValue?.profile_type) {
  profileType = profileValue.profile_type;
}

// Get session_id from query params or referer header
// For now, you'll need to pass it from the widget somehow
// OR use a different approach (see Alternative below)

return [{
  session_id: sessionId,
  profile_type: profileType,
  conversation_id: conversationId,
  created_at: new Date().toISOString()
}];
```

### 4. Respond to Webhook
- Response Code: 200
- Response Body: `{{ $json }}`
- Headers:
  ```
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: POST, OPTIONS
  Access-Control-Allow-Headers: Content-Type, x-signature
  Content-Type: application/json
  ```

## IMPORTANT: Linking Session ID to Conversation

**Problem:** ElevenLabs doesn't know about our session_id when it sends the webhook.

**Solutions:**

### Solution 1: Pass session_id via ElevenLabs dynamic variables
- In your ElevenLabs agent config, set a dynamic variable
- The widget can pass custom data when starting a conversation
- This requires ElevenLabs widget config changes

### Solution 2: Time-based matching (hacky but works)
- Store profile with timestamp when ElevenLabs sends it
- When webpage polls with session_id, return the most recent profile (< 2 minutes old)
- Works for single-user testing, not production-ready

```javascript
// In the GET profile handler
// Instead of matching by session_id, return most recent:
SELECT profile_type 
FROM profiles 
WHERE created_at > NOW() - INTERVAL '2 minutes'
ORDER BY created_at DESC 
LIMIT 1
```

### Solution 3: Use localStorage to store conversation_id
- When widget starts, it might expose the conversation_id
- Store it in localStorage
- Send it with the polling request
- Match profiles by conversation_id instead of session_id

## Testing

1. Deploy the updated index.html
2. Open DevTools Console
3. Note the session_id logged on page load
4. Complete a call
5. Watch console for:
   - `"conversationEnded event fired"`
   - `"Starting to poll for results..."`
   - `"Poll attempt 1/60"`
   - `"Response status: 200"`
   - `"Received data: {...}"`
6. Profile should appear automatically

## Quick Test Without Full n8n Setup

Modify n8n to always return a dummy profile:

```javascript
// In webhook, always return this:
return [{
  data_collection_results: {
    eq_analysis: {
      value: JSON.stringify({ profile_type: 'Secure' })
    }
  }
}];
```

This will let you test the frontend polling logic immediately.


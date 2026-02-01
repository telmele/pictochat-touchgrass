# TouchDesigner WebSocket Connection Guide

## Connection Details

- **WebSocket URL:** `ws://YOUR_IP:8090/`
- **Protocol:** WebSocket (text-based, JSON messages)
- **Max Message Size:** 524288 bytes (512 KB)

## Connection Setup in TouchDesigner

### 1. Using the webSocketDAT Operator

1. Add a **webSocketDAT** operator to your network
2. Configure the parameters:
   - **Network Address:** `ws://YOUR_IP:8090/` (or `ws://localhost:8090/` locally)
   - **Active:** ON
   - **Auto Reconnect:** ON (recommended)

### 2. Required Handshake Protocol

After connecting, you **MUST** send a handshake message:

```python
# In a webSocketDAT callback or script
op('websocket1').sendText('handshake')
```

**Important:** The server expects this exact string `"handshake"` as the first message. Without it, your connection will timeout.

### 3. Keep-Alive (Ping/Pong)

The server sends `"ping"` every 30 seconds. You must respond with `"pong"`:

```python
# In webSocketDAT callbacks
def onReceiveText(dat, data):
    if data == 'ping':
        dat.sendText('pong')
    elif data != 'handshake':
        # Process actual message (JSON)
        import json
        try:
            msg = json.loads(data)
            # Handle message here
        except:
            pass
```

## Complete TouchDesigner Setup

### Method 1: Using webSocketDAT callbacks

Create a callback DAT with this code:

```python
import json

def onOpen(dat):
    print("WebSocket connected!")
    # Send handshake immediately
    dat.sendText('handshake')
    return

def onClose(dat):
    print("WebSocket closed")
    return

def onReceiveText(dat, data):
    # Handle ping/pong
    if data == 'ping':
        dat.sendText('pong')
        return
    
    # Handle JSON messages
    try:
        msg = json.loads(data)
        print("Received:", msg)
        
        # Store message in storage for access by other operators
        parent().store('lastMessage', msg)
        
    except json.JSONDecodeError:
        print("Received non-JSON:", data)
    return

def onReceiveBinary(dat, data):
    return

def onReceiveError(dat, error):
    print("WebSocket error:", error)
    return
```

### Method 2: Using Execute DAT monitoring

Create an Execute DAT monitoring the webSocketDAT:

```python
def onFrameStart(frame):
    ws = op('websocket1')
    
    # Check if connected and handshake not sent yet
    if ws.par.connected and not hasattr(parent(), 'handshakeSent'):
        ws.sendText('handshake')
        parent().handshakeSent = True
    
    # Handle incoming messages
    while ws.numInMessages > 0:
        msg = ws.getInMessage()
        if msg == 'ping':
            ws.sendText('pong')
        else:
            try:
                data = json.loads(msg)
                # Process your data here
                print("Received:", data)
            except:
                pass
```

## Sending Messages to the Server

The server expects JSON messages. Example message formats:

```python
import json

# Example: Send a message
message = {
    "type": "chat",
    "room": "room_a",
    "content": "Hello from TouchDesigner!"
}

op('websocket1').sendText(json.dumps(message))
```

## Network Setup

### Local Testing
- URL: `ws://localhost:8090/`
- Works only on the same machine

### Network Access (Docker or Modified settings.json)
- URL: `ws://192.168.x.x:8090/` (replace with your actual IP)
- Make sure firewall allows port 8090
- Server must be configured with `host: "0.0.0.0"` in settings.json

### Finding Your IP Address
Run in PowerShell:
```powershell
(Get-NetIPAddress | Where-Object {$_.AddressFamily -eq 'IPv4' -and $_.IPAddress -notlike '127.*'}).IPAddress
```

## Troubleshooting

### Connection Closes Immediately
- Make sure you send `"handshake"` as the first message
- Check that you're responding to `"ping"` with `"pong"`

### Can't Connect from Network
- Verify server is running with `host: "0.0.0.0"` (not `127.0.0.1`)
- Check Windows Firewall settings for port 8090
- Confirm you're using the correct IP address

### Messages Not Received
- Check webSocketDAT's **Connected** parameter is ON
- Verify callbacks are properly attached
- Enable **Print Incoming** in webSocketDAT for debugging

## Message Types Received from Server

### 1. `sv_nameVerified` - Username Validated
Sent after successful name verification.
```json
{
  "type": "sv_nameVerified",
  "player": {
    "name": "YourName",
    "color": 14013909
  }
}
```

### 2. `sv_roomIds` - Room List Update
List of available rooms and player counts.
```json
{
  "type": "sv_roomIds",
  "count": [2, 5, 0, 1],
  "ids": ["room_a", "room_b", "room_c", "room_d", "room_e"]
}
```

### 3. `sv_roomData` - Room Join Confirmation
Confirms you joined a room.
```json
{
  "type": "sv_roomData",
  "id": "room_a"
}
```

### 4. `sv_playerJoined` - Player Joined Room
When another player joins.
```json
{
  "type": "sv_playerJoined",
  "player": {
    "name": "OtherPlayer",
    "color": 16711680
  },
  "id": "room_a"
}
```

### 5. `sv_playerLeft` - Player Left Room
When a player leaves.
```json
{
  "type": "sv_playerLeft",
  "player": {
    "name": "OtherPlayer",
    "color": 16711680
  },
  "id": "room_a"
}
```

### 6. `sv_receivedMessage` - Chat Message (IMPORTANT!)
This is the main message type with drawings and text.
```json
{
  "type": "sv_receivedMessage",
  "message": {
    "player": {
      "name": "PlayerName",
      "color": 14013909
    },
    "textboxes": [
      {"x": 113, "y": 211, "text": "Hello!"},
      {"x": 27, "y": 227, "text": "Second line"}
    ],
    "lines": 2,
    "drawing": [
      {"x": 10, "y": 20, "type": 1},
      {"x": 15, "y": 25, "type": 0}
    ]
  }
}
```

**Drawing types:**
- `0`: Continue line
- `1`: Start new line
- `2`: Flood fill
- `3`: Clear/background

## Complete TouchDesigner Callback Implementation

```python
import json

# Global storage for messages
if not hasattr(parent(), 'messages'):
    parent().messages = []
if not hasattr(parent(), 'players'):
    parent().players = {}

def onOpen(dat):
    print("✓ WebSocket connected!")
    dat.sendText('handshake')
    return

def onClose(dat):
    print("✗ WebSocket closed")
    parent().messages = []
    parent().players = {}
    return

def onReceiveText(dat, data):
    # Handle ping/pong keep-alive
    if data == 'ping':
        dat.sendText('pong')
        return
    
    # Parse JSON message
    try:
        msg = json.loads(data)
        msgType = msg.get('type', '')
        
        # Room list update
        if msgType == 'sv_roomIds':
            counts = msg.get('count', [])
            rooms = msg.get('ids', [])
            print(f"Rooms: {list(zip(rooms, counts))}")
            parent().store('roomCounts', counts)
        
        # Name verified
        elif msgType == 'sv_nameVerified':
            player = msg.get('player', {})
            print(f"✓ Logged in as: {player.get('name')}")
            parent().store('myPlayer', player)
        
        # Joined a room
        elif msgType == 'sv_roomData':
            roomId = msg.get('id')
            print(f"✓ Joined room: {roomId}")
            parent().store('currentRoom', roomId)
        
        # Player joined
        elif msgType == 'sv_playerJoined':
            player = msg.get('player', {})
            name = player.get('name')
            parent().players[name] = player
            print(f"+ {name} joined")
        
        # Player left
        elif msgType == 'sv_playerLeft':
            player = msg.get('player', {})
            name = player.get('name')
            if name in parent().players:
                del parent().players[name]
            print(f"- {name} left")
        
        # CHAT MESSAGE - This is the important one!
        elif msgType == 'sv_receivedMessage':
            message = msg.get('message', {})
            player = message.get('player', {})
            textboxes = message.get('textboxes', [])
            drawing = message.get('drawing', [])
            
            # Extract all text
            fullText = ' '.join([tb.get('text', '') for tb in textboxes])
            
            # Store message
            parent().messages.append({
                'player': player.get('name'),
                'color': player.get('color'),
                'text': fullText,
                'textboxes': textboxes,
                'drawing': drawing,
                'lines': message.get('lines', 1)
            })
            
            # Keep only last 50 messages
            if len(parent().messages) > 50:
                parent().messages = parent().messages[-50:]
            
            # Print to console
            print(f"{player.get('name')}: {fullText}")
            
            # Store latest message for easy access
            parent().store('latestMessage', parent().messages[-1])
        
    except json.JSONDecodeError as e:
        print(f"JSON error: {e}")
        print(f"Data: {data}")
    
    return

def onReceiveBinary(dat, data):
    return

def onReceiveError(dat, error):
    print(f"WebSocket error: {error}")
    return
```

## Accessing Messages in Other Operators

### In a Text DAT - Display Latest Message
```python
# Read latest message text
if hasattr(parent(), 'latestMessage'):
    msg = parent().latestMessage
    return f"{msg['player']}: {msg['text']}"
else:
    return "No messages yet"
```

### In a Table DAT - Display All Messages
```python
# In an Execute DAT, update a table with all messages
def onFrameEnd(frame):
    table = op('table_messages')
    
    if hasattr(parent(), 'messages'):
        # Clear table
        table.clear()
        table.appendRow(['Player', 'Text', 'Color'])
        
        # Add all messages
        for msg in parent().messages:
            table.appendRow([
                msg['player'],
                msg['text'],
                str(msg['color'])
            ])
```

### In a CHOP - Use Message Data
```python
# In a Script CHOP or Execute DAT
def cook(scriptOp):
    if hasattr(parent(), 'latestMessage'):
        msg = parent().latestMessage
        
        # Output color as channels
        color = msg['color']
        r = (color >> 16) & 0xFF
        g = (color >> 8) & 0xFF
        b = color & 0xFF
        
        scriptOp.clear()
        scriptOp.numSamples = 1
        scriptOp.appendChan('r')
        scriptOp.appendChan('g')
        scriptOp.appendChan('b')
        
        scriptOp['r'][0] = r / 255.0
        scriptOp['g'][0] = g / 255.0
        scriptOp['b'][0] = b / 255.0
```

## Example: Complete TouchDesigner Network

1. **webSocketDAT** (`websocket1`)
   - Network Address: `ws://localhost:8090/`
   - Active: ON
   - Callbacks DAT: `callback1`

2. **DAT** (`callback1`) - Complete callbacks script (see above)

3. **Text DAT** (`text_latest`) - Display latest message
   ```python
   if hasattr(parent(), 'latestMessage'):
       msg = parent().latestMessage
       return f"{msg['player']}: {msg['text']}"
   return "Waiting for messages..."
   ```

4. **Table DAT** (`table_messages`) - Empty table for message history

5. **Execute DAT** - Update table on frame
   ```python
   def onFrameEnd(frame):
       if frame % 30 == 0:  # Update once per second at 30fps
           table = op('table_messages')
           if hasattr(parent(), 'messages'):
               table.clear()
               table.appendRow(['Player', 'Message'])
               for msg in parent().messages[-10:]:  # Last 10 messages
                   table.appendRow([msg['player'], msg['text']])
   ```

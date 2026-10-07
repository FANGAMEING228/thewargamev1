# FPV War Game - Multiplayer Soldier Mode Guide

## Overview

This guide covers the **P2P Multiplayer Soldier Mode** where players can play as ground soldiers fighting against FPV drones and each other in a shared online environment.

## Architecture

### 3-Tier System:
1. **Signaling Server** (WebSocket) - Handles initial P2P connection negotiation
2. **WebRTC P2P** - Direct peer-to-peer game state sync between players
3. **Game Clients** - Browser-based Three.js soldier players

```
┌─────────────────────────────────────────────────────────────┐
│           Signaling Server (WebSocket)                       │
│  - User announcements                                        │
│  - SDP offer/answer exchange                                 │
│  - ICE candidate relay                                       │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ↓                    ↓                    ↓
    ┌─────────┐          ┌─────────┐          ┌─────────┐
    │ Player 1│ ←──RTCp2p──→ │ Player 2│          │ Player 3│
    │ Browser │           │ Browser │ ←─RTCp2p──→ │ Browser │
    └─────────┘          └─────────┘          └─────────┘
```

## Setup Instructions

### 1. Install Server Dependencies

```bash
cd server
npm install ws
```

### 2. Start Signaling Server

```bash
node signalingServer.js
```

Server will be available at `ws://localhost:8080`

### 3. Enable Soldier Mode in HTML

Edit `index.htm` to add:

```html
<script src="src/soldierMultiplayer.js" type="module"></script>
<script src="src/soldierPlayer.js" type="module"></script>
```

### 4. Initialize Multiplayer in Your Game Code

In your main engine.js or game initialization:

```javascript
import { soldierMultiplayer } from './src/soldierMultiplayer.js';
import { createSoldierPlayer } from './src/soldierPlayer.js';

// During game mode 5 (soldier) initialization:
async function initSoldierMultiplayer() {
    const success = await soldierMultiplayer.init('ws://localhost:8080');
    
    if (success) {
        // Create player soldier
        const soldier = createSoldierPlayer(scene, { x: 0, y: 1, z: 0 });
        
        // Handle remote soldiers spawning
        soldierMultiplayer.onRemoteSoldierSpawned = (userId, soldierData) => {
            // Create visual representation for remote soldier
            createRemoteSoldierMesh(scene, userId, soldierData);
        };
        
        // Handle remote soldiers moving
        soldierMultiplayer.onRemoteSoldierMoved = (userId, soldierData) => {
            updateRemoteSoldierPosition(userId, soldierData);
        };
        
        // Handle remote soldiers firing
        soldierMultiplayer.onRemoteSoldierFired = (userId, fireData) => {
            createMuzzleFlash(scene, fireData.position, userId);
        };
        
        // Handle remote soldiers dying
        soldierMultiplayer.onRemoteSoldierDied = (userId, deathData) => {
            removeRemoteSoldier(scene, userId);
        };
        
        return soldier;
    }
}
```

## Game Modes

### Mode 5 - Multiplayer Soldier

- **Controls:**
  - WASD - Movement
  - Space - Jump
  - Mouse - Look around
  - Shift - Sprint
  - Left Click - Fire weapon

- **Game Rules:**
  - Health: 100 points
  - Ammo: 120 rounds
  - Fire rate: 100ms per shot
  - Damage per shot: 25 points
  - Fire range: 100 units

- **Objectives:**
  - Defend against FPV drones (mode 0)
  - Combat other soldiers
  - Survive as long as possible

## Data Flow

### Soldier Update Message (50ms interval)
```json
{
  "type": "soldier-update",
  "userId": "user123",
  "position": { "x": 10, "y": 1, "z": 20 },
  "rotation": { "x": -0.1, "y": 1.5, "z": 0 },
  "health": 85,
  "isAlive": true,
  "timestamp": 1633024800000
}
```

### Fire Event Message
```json
{
  "type": "soldier-fired",
  "userId": "user123",
  "position": { "x": 10, "y": 1.2, "z": 20 },
  "direction": { "x": 0, "y": -0.1, "z": -1 },
  "damage": 25,
  "timestamp": 1633024800000
}
```

### Death Event Message
```json
{
  "type": "soldier-died",
  "userId": "user123",
  "position": { "x": 10, "y": 1, "z": 20 },
  "killedByUserId": "user456",
  "timestamp": 1633024800000
}
```

## Network Performance

- **Update Frequency:** 50ms (20 updates/second)
- **Bandwidth per player:** ~1.5 KB/s
- **Latency impact:** ~50-200ms typical
- **Max players tested:** 8 players peer-to-peer
- **Recommended:** 2-4 players for optimal experience

## Advanced Configuration

### Adjust Soldier Parameters

In `soldierPlayer.js`:

```javascript
this.moveSpeed = 5;          // units/sec
this.sprintSpeed = 8;        // units/sec
this.fireRate = 100;         // ms between shots
this.damagePerShot = 25;     // health points
this.fireRangeMeters = 100;  // max shot distance
```

### Adjust Sync Rate

In `soldierMultiplayer.js`:

```javascript
const SOLDIER_SYNC_INTERVAL = 50; // ms between position updates
const P2P_HEARTBEAT = 5000;       // ms between connection checks
```

### Custom ICE Servers

In `soldierMultiplayer.js`, modify the RTCPeerConnection options:

```javascript
const peerConnection = new RTCPeerConnection({
    iceServers: [
        { urls: ['stun:stun.l.google.com:19302', 'stun:stun1.l.google.com:19302'] },
        { 
            urls: ['turn:your-turn-server.com:3478'],
            username: 'user',
            credential: 'pass'
        }
    ]
});
```

## Troubleshooting

### "Cannot init multiplayer: user not logged in"
- Make sure user is authenticated via auth.js
- Login through the UI before starting soldier mode

### "Connection to player failed"
- Check signaling server is running
- Verify firewall allows WebSocket connections
- Check browser console for errors

### High latency / Jittery players
- Reduce SOLDIER_SYNC_INTERVAL for more frequent updates
- Use turn-server for better connectivity
- Reduce player count

### Players not seeing each other
- Restart signaling server
- Check both players are in mode 5
- Verify no network issues between peers

## Future Enhancements

- [ ] Team-based gameplay
- [ ] Soldier skill classes (Rifleman, Sniper, Support)
- [ ] Squad system
- [ ] Voice chat integration
- [ ] Weapon variety and loadouts
- [ ] Environmental damage effects
- [ ] Advanced animation synchronization
- [ ] Spectator mode
- [ ] Recording/replay system

## Performance Tips

1. **Reduce draw calls** - Merge soldier meshes when distant
2. **Use LOD** - Lower detail for far-away players
3. **Limit shadow casting** - Only nearby soldiers cast shadows
4. **Optimize physics** - Use simple capsule colliders
5. **Network culling** - Don't send updates for distant players

## Security Considerations

⚠️ **Important:** This is a development version. For production:

1. **Validate all inputs** on server side
2. **Implement anti-cheat** measures
3. **Use TURN server** with authentication
4. **Encrypt signaling messages** with TLS
5. **Rate limit** message frequency
6. **Implement player reporting** system
7. **Use secure WebSocket (wss://)** in production
8. **Add server-side damage validation**

## License

Part of FPV War Game project. See LICENSE file for details.

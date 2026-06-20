<div align="center">
  <h1>⚡ BitSocket</h1>
  <p><strong>A high-performance, schema-driven, binary WebSocket framework for Node.js.</strong></p>
  <p>BitSocket provides the developer experience of Socket.io but with <b>Protobuf-level network compression</b>. By leveraging a strict Schema Engine, BitSocket drops JSON completely, stripping keys and formatting to deliver up to an 80% reduction in network payload size.</p>
</div>

<hr />

## 🚀 Why BitSocket?

Socket.io is built on top of Engine.io, which means it transmits stringified JSON. If you send an array of 100 user objects, the keys `"id"`, `"name"`, and `"email"` are transmitted 100 times. 

**BitSocket completely eliminates this overhead.** 
By defining Schemas on your server, BitSocket maps your JavaScript objects directly into strict `ArrayBuffers`. The keys are never transmitted over the network—only the pure, deeply compressed binary data.

### Features
- 🧬 **Schema Auto-Discovery**: Define your schemas on the server. The moment a client connects, the server pushes the schemas to the client during the handshake. Zero manual schema sharing required!
- 📦 **Extreme Binary Compression**: Drops all JSON overhead resulting in 40% to 80% smaller network payloads.
- 🔄 **Connection Multiplexing**: Share a single underlying TCP connection across multiple isolated Namespaces (e.g. `/user`, `/store`), exactly like Socket.io.
- 👥 **Room Broadcasting**: Full support for group communication (`socket.join('room')`, `io.to('room').emit(...)`).
- ♾️ **Recursive Data Types**: Native support for deeply nested objects and multi-dimensional arrays without losing compression.
- 🧩 **Dynamic MsgPack Fallbacks**: Need to send an arbitrary, unpredictable JSON dictionary? Define the field as `'object'` and BitSocket seamlessly drops down to MsgPack compression for that specific field.

---

## 🛠️ Installation

```bash
npm install bit-socket
```

---

## 📖 Quick Start

### 1. The Server
Define your schemas, attach them to a namespace, and start listening!

```javascript
import { BitSocketServer, Schema } from 'bit-socket';

const io = new BitSocketServer({ port: 5005 });

// 1. Define strict binary schemas
const UserSchemas = [
  new Schema('user:create', {
    name: 'string',
    age: 'uint8',
    tags: ['string'] // Support for strict arrays!
  }),
  new Schema('user:created', {
    id: 'uint32',
    success: 'boolean',
    metadata: 'object' // Dynamic arbitrary JSON falls back to MsgPack!
  })
];

// 2. Attach schemas to a Namespace
io.of('/user').schema(UserSchemas);

// 3. Handle Connections
io.of('/user').on('connection', (socket) => {
  console.log('User connected!', socket.id);

  socket.on('user:create', (payload) => {
    console.log("Received:", payload); // { name: 'Alice', age: 25, tags: ['admin'] }

    // Broadcast a response to everyone in a specific room
    socket.join('admin-room');
    socket.to('admin-room').emit('user:created', {
      id: 1045,
      success: true,
      metadata: { serverPing: 12 }
    });
  });
});
```

### 2. The Client
The client only needs to connect. **It automatically downloads the schemas during the connection handshake!**

```javascript
import { BitSocketClient } from 'bit-socket';

// Connect the root multiplexer
const root = new BitSocketClient('ws://localhost:5005');

// Open the /user channel (Re-uses the existing WebSocket connection!)
const userClient = root.of('/user');

userClient.on('connect', () => {
  // Emit the event using a normal JS object. 
  // BitSocket intercepts it, strips the keys, and compresses it using the downloaded Schema!
  userClient.emit('user:create', {
    name: "Alice",
    age: 25,
    tags: ['admin']
  });
});

userClient.on('user:created', (payload) => {
  console.log("Success:", payload); 
});
```

---

## 🧱 Supported Schema Types

BitSocket currently supports mapping your JavaScript data into the following strict memory representations:

| Schema Type | JavaScript Type | Byte Size |
|-------------|-----------------|-----------|
| `'uint8'`   | Number          | 1 byte    |
| `'boolean'` | Boolean         | 1 byte    |
| `'uint16'`  | Number          | 2 bytes   |
| `'uint32'`  | Number          | 4 bytes   |
| `'int32'`   | Number          | 4 bytes   |
| `'float64'` | Number          | 8 bytes   |
| `'string'`  | String          | 4 bytes (len) + utf8 bytes |
| `'bytes'`   | Uint8Array      | 4 bytes (len) + buffer |

### Advanced Types
- **Arrays**: Wrap a type in brackets. `['string']` or `[['uint8']]`.
- **Nested Objects**: Define a literal object. `{ profile: { age: 'uint8' } }`.
- **Dynamic Fallbacks**: Use `'object'`, `'array'`, or `'any'` to allow arbitrary JSON data. BitSocket will compress this specific field using MsgPack while preserving keys.

---

## 🌐 Multiplexing & Rooms

BitSocket matches the elegant routing API of Socket.io:

**Namespaces (Multiplexing)**  
Keep logic separated without opening multiple TCP connections.
```javascript
const chatNsp = root.of('/chat');
const gameNsp = root.of('/game');
```

**Rooms**  
Create isolated communication channels within a namespace.
```javascript
// Server Side
socket.join('lobby-1');
socket.leave('lobby-1');

// Emit to everyone in the room EXCEPT the sender
socket.broadcast.to('lobby-1').emit('message', data);

// Emit to everyone in the room INCLUDING the sender
io.of('/chat').to('lobby-1').emit('message', data);
```

---

## 📈 Performance vs Socket.io
See the `NETWORK_ANALYSIS.md` document for a fully quantified byte-for-byte breakdown. In summary:
- **Single Objects**: ~40% smaller payloads.
- **Large Arrays**: ~50% to 80% smaller payloads.
- **Continuous Metrics**: ~60% smaller payloads.

Because network latency (I/O) is the slowest bottleneck in any real-time system, BitSocket provides lower end-to-end latency for high-frequency applications.

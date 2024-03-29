# INSPScalability
## Problems

I have designed my system keeping single system in mind, So I created a RoomManager to manage the room, two global variables allPeers and allRooms where allPeers managing allPeers connected to any room of that particular server and allRooms keep instances of RoomManager of that particular server but things are failing at the moment the problem is


Let suppose 2 server instances is there and 10 peoples already connected to server 1 tehre communication going good but when a new peer connected with server 2 then it won't get any peerDetails anything because everybody is connected to othere server however they all have joined with same roomId , becasue I a managed with global variable so it is not working properly, In short i can say at the moment only the current ec2 managing its own state of info, and I don't know even if it is possible to exchange though i use redis adpater 

Firstly how to ensure that doesn't matter to which server my peer connected firstly he should get how many already people connected to that room even if they are on different server

```js
const redis = require('redis');
const { promisify } = require('util');

// Create a Redis client
const redisClient = redis.createClient();

// Promisify Redis commands for easier usage
const getAsync = promisify(redisClient.get).bind(redisClient);
const setAsync = promisify(redisClient.set).bind(redisClient);

class RoomManager {
  constructor() {
    this.rooms = new Map();
  }

  async joinRoom(roomId, socketId) {
    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, new Set());
    }
    this.rooms.get(roomId).add(socketId);
    await this.updateRoomInfo(roomId);
  }

  async updateRoomInfo(roomId) {
    const roomInfo = { peers: Array.from(this.rooms.get(roomId)) };
    await setAsync(`room:${roomId}`, JSON.stringify(roomInfo));
  }

  async getPeersInRoom(roomId) {
    const roomInfoStr = await getAsync(`room:${roomId}`);
    if (roomInfoStr) {
      const roomInfo = JSON.parse(roomInfoStr);
      return roomInfo.peers;
    }
    return [];
  }
}

const roomManager = new RoomManager();

const joinRoomPreviewSocketHandler = async (data, callback, socket, io) => {
  try {
    const { roomId } = data;
    if (roomId) {
      const allPeersInThisRoom = await roomManager.getPeersInRoom(roomId);

      // Join the room
      socket.join(roomId);

      // Add the current peer to the room
      await roomManager.joinRoom(roomId, socket.id);

      // Send response with peers in this room
      callback({ success: true, peers: allPeersInThisRoom });
    } else {
      callback({ success: false }); // No room id supplied
    }
  } catch (err) {
    console.log("Error in join room Preview Handler", err);
  }
};


```


## Global variables approach correct ?

Each server instance maintains its own state of connected clients and rooms, which is sufficient for handling local operations and interactions.

However, to ensure that the state is consistent across all server instances and that clients connected to different server instances can communicate seamlessly, you're using Redis as the adapter for Socket.IO. Redis acts as a centralized message broker, facilitating communication and synchronization of state between different server instances.

## Does single server setup is correct and how to do inter server communication ?

- Local Management: Each server instance manages its own local state, including details about connected peers, rooms, and any communication specific to those peers. This allows for efficient handling of local operations without the need to constantly synchronize with other server instances.

- Inter-Server Communication via Redis: When an event occurs that requires synchronization across server instances, such as a new peer joining a room, the relevant information is published to a Redis channel. Other server instances are subscribed to this channel and receive notifications about the event.

- Synchronization: Upon receiving the notification from Redis, each server instance updates its own local state accordingly. This ensures that all server instances are aware of the event and can take appropriate actions, such as updating their own lists of connected peers.

By following this approach, you achieve a balance between local efficiency and global consistency. Server instances only need to communicate with Redis for events that require synchronization, while still maintaining fast and efficient local operations for most tasks.




## Does local peer list within RoomManager need to hold other peers from other server


In a distributed system where you have multiple server instances managing rooms and peers, it's generally a good practice to have each server instance maintain its own local state of connected peers. This means that the RoomManager instance on each server only maintains information about the peers connected to that specific server instance.

When a new peer joins a room, it's typically added to the local peers list of the RoomManager instance where it connected. This ensures that each server instance operates independently and efficiently without constantly synchronizing state with other instances for every peer connection.

However, for scenarios where you need global visibility of peer connections across all server instances, such as broadcasting messages to all peers in a room regardless of which server they are connected to, you would use a mechanism like Redis pub/sub for inter-server communication and synchronization.

In summary:

Each RoomManager instance maintains its own local peers list for efficiency and independence.
Inter-server communication and synchronization are handled through Redis or a similar mechanism for scenarios that require global visibility or coordination across server instances.


## How to get fresh data about the room when any client get connected first to any server?

When a new peer joins a room on Server 2 and receives the updated information from Redis, they are now aware of the other peers in the room, even if those peers are connected to different servers.

If a socket.emit('newPeerJoined') event is triggered on Server 2 after the new peer joins, it will be emitted to all connected clients on Server 2, including the newly joined peer, because socket.emit sends the event to all connected clients on that particular server instance.

In summary, at the initial connection or when updates occur (such as a new peer joining), clients need to get fresh data from Redis to ensure they have the most up-to-date information about the room and its peers. Subsequent events or communications, such as socket.emit events, will then be handled locally within each server instance.



## How to use different redis function 
```js
const redis = require('redis');
const { promisify } = require('util');

// Create a Redis client
const redisClient = redis.createClient();

// Promisify Redis commands for easier usage
const hsetAsync = promisify(redisClient.hset).bind(redisClient);
const hgetAsync = promisify(redisClient.hget).bind(redisClient);
const hgetAllAsync = promisify(redisClient.hgetall).bind(redisClient);

// Update the list of peers in Redis
const updatePeersInRoom = async (roomId, peerId, peerDetails) => {
  try {
    // Add or update the peer's details in the hash for the room
    await hsetAsync(`room:${roomId}:peers`, peerId, JSON.stringify(peerDetails));
    console.log(`Peer ${peerId} added/updated in room ${roomId}.`);
  } catch (error) {
    console.error(`Error updating peers in room ${roomId}:`, error);
  }
};

// Retrieve the list of peers in a room from Redis
const getPeersInRoom = async (roomId) => {
  try {
    // Get all peer details for the room from the hash
    const peerDetails = await hgetAllAsync(`room:${roomId}:peers`);
    // Convert the peer details to an array of objects
    const peers = Object.values(peerDetails).map(JSON.parse);
    return peers;
  } catch (error) {
    console.error(`Error getting peers in room ${roomId}:`, error);
    return [];
  }
};

// Usage
const roomId = 'exampleRoomId';
const newPeerId = 'examplePeerId';
const newPeerDetails = {
  routerId: 'exampleRouterId',
  peerName: 'Example Peer',
};

updatePeersInRoom(roomId, newPeerId, newPeerDetails);
const peersInRoom = await getPeersInRoom(roomId);
console.log(peersInRoom);
```

## How to use redis 

```js
const redis = require('redis');
const { promisify } = require('util');

// Create a Redis client
const redisClient = redis.createClient();

// Promisify Redis commands for easier usage
const hsetAsync = promisify(redisClient.hset).bind(redisClient);
const hgetAsync = promisify(redisClient.hget).bind(redisClient);
const hgetAllAsync = promisify(redisClient.hgetall).bind(redisClient);

// Update the list of peers in Redis
const updatePeersInRoom = async (roomId, peerId, peerDetails) => {
  try {
    // Add or update the peer's details in the hash for the room
    await hsetAsync(`room:${roomId}:peers`, peerId, JSON.stringify(peerDetails));
    console.log(`Peer ${peerId} added/updated in room ${roomId}.`);
  } catch (error) {
    console.error(`Error updating peers in room ${roomId}:`, error);
  }
};

// Retrieve the list of peers in a room from Redis
const getPeersInRoom = async (roomId) => {
  try {
    // Get all peer details for the room from the hash
    const peerDetails = await hgetAllAsync(`room:${roomId}:peers`);
    // Convert the peer details to an array of objects
    const peers = Object.values(peerDetails).map(JSON.parse);
    return peers;
  } catch (error) {
    console.error(`Error getting peers in room ${roomId}:`, error);
    return [];
  }
};

// Usage
const roomId = 'exampleRoomId';
const newPeerId = 'examplePeerId';
const newPeerDetails = {
  routerId: 'exampleRouterId',
  peerName: 'Example Peer',
};

updatePeersInRoom(roomId, newPeerId, newPeerDetails);
const peersInRoom = await getPeersInRoom(roomId);
console.log(peersInRoom);
```


## How to distribute streams across server
To distribute producer streams across multiple mediasoup servers using Redis as a signaling mechanism, you'll need to implement a custom solution that involves coordinating between the different servers. Here's a high-level overview of how you can achieve this:

- Establish a Redis Pub/Sub Channel: Set up a Redis Pub/Sub channel to allow communication between the different mediasoup servers. Each server will subscribe to this channel to receive messages.

- Publish Producer Information: When a producer is created on one server, publish information about this producer (such as its ID and transport parameters) to the Redis channel.

- Subscribe and Consume Messages: Each mediasoup server will subscribe to the Redis channel and consume messages about producers created on other servers. When a message about a producer is received, the server will create a consumer for that producer.

- Handling Consumer Creation: When a server receives information about a producer from another server, it will use that information to create a consumer on its end. This consumer will consume the stream produced by the producer.

- Error Handling and Cleanup: Implement error handling and cleanup logic to handle scenarios like server disconnects, failures, or producer/consumer expiration.

Here's a simplified example of how you might implement this in Node.js using mediasoup and ioredis for Redis communication:


```js

const { Worker } = require('mediasoup');
const Redis = require('ioredis');

const redis = new Redis();

// Create a mediasoup worker
const worker = await Worker.createWorker();

// Subscribe to the Redis channel
redis.subscribe('mediasoup_producers');

// Handle messages received from the Redis channel
redis.on('message', async (channel, message) => {
  // Parse the message
  const { roomId, producerId, transportParameters } = JSON.parse(message);

  // Create a consumer for the received producer
  const consumer = await createConsumer(roomId, producerId, transportParameters);

  // Handle consumer creation
  // For example, add the consumer to a map for later reference
});

// Function to create a consumer for a producer
async function createConsumer(roomId, producerId, transportParameters) {
  // Lookup the room and transport based on roomId
  // Use roomId to determine which server instance should handle this consumer
  const room = getRoomFromId(roomId);
  const transport = getTransportFromParameters(transportParameters);

  // Create a consumer using the provided transport
  const consumer = await room.createConsumer({
    producerId,
    transport,
  });

  return consumer;
}

// Function to handle cleanup and error handling
function handleErrorsAndCleanup() {
  // Implement logic to handle errors and cleanup resources
  // For example, remove consumers from the map on disconnect
}

```

## How to use inter server stream exchange 
SERVER - 1
```js
const mediasoup = require('mediasoup');
const Redis = require('ioredis');

// Create a Redis client for signaling
const redis = new Redis();

// Function to create a PipeTransport for a router
async function createPipeTransport(router) {
  // Create a PipeTransport for the router
  const pipeTransport = await router.createPipeTransport({
    listenIp: '0.0.0.0', // Listen on all IPs
  });

  // Store the PipeTransport ID in Redis for signaling
  await redis.set(`router:${router.id}:pipeTransportId`, pipeTransport.id);

  // Store the PipeTransport IP and port in Redis for signaling
  await redis.set(`router:${router.id}:pipeTransportIp`, 'server2_ip'); // Assuming server2 IP
  await redis.set(`router:${router.id}:pipeTransportPort`, 'server2_port'); // Assuming server2 port

  return pipeTransport;
}

// Function to create a producer
async function createProducer(pipeTransport, kind) {
  // Create a producer on the PipeTransport
  const producer = await pipeTransport.produce({
    kind,
    /* Other producer options */
  });

  return producer;
}

// Example usage for Server 1
(async () => {
  // Connect to mediasoup worker
  const worker = await mediasoup.createWorker();

  // Create router
  const router = await worker.createRouter();

  // Create PipeTransport for the router
  const pipeTransport = await createPipeTransport(router);

  // Create producer on the PipeTransport
  const producer = await createProducer(pipeTransport, 'audio');
})();

```


SERVER - 2

```js
const mediasoup = require('mediasoup');
const Redis = require('ioredis');

// Create a Redis client for signaling
const redis = new Redis();

// Function to create a PipeTransport for a router
async function createPipeTransport(router) {
  // Create a PipeTransport for the router
  const pipeTransport = await router.createPipeTransport({
    listenIp: '0.0.0.0', // Listen on all IPs
  });

  // Store the PipeTransport ID in Redis for signaling
  await redis.set(`router:${router.id}:pipeTransportId`, pipeTransport.id);

  // Store the PipeTransport IP and port in Redis for signaling
  await redis.set(`router:${router.id}:pipeTransportIp`, 'server1_ip'); // Assuming server1 IP
  await redis.set(`router:${router.id}:pipeTransportPort`, 'server1_port'); // Assuming server1 port

  return pipeTransport;
}

// Function to create a consumer
async function createConsumer(pipeTransport, producerId) {
  // Create a consumer on the PipeTransport
  const consumer = await pipeTransport.consume({
    producerId,
  });

  return consumer;
}

// Example usage for Server 2
(async () => {
  // Connect to mediasoup worker
  const worker = await mediasoup.createWorker();

  // Create router
  const router = await worker.createRouter();

  // Create PipeTransport for the router
  const pipeTransport = await createPipeTransport(router);

  // Retrieve the producerId from Redis for signaling
  const producerId = await redis.get('producerId');

  // Create consumer on the PipeTransport
  const consumer = await createConsumer(pipeTransport, producerId);
})();

```



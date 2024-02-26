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
Chat Application Architecture at 100M+ Concurrent Users
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Full System Overview

                          ┌─────────────────────────────────┐
                          │         Clients                 │                                                                                                                                                   
                          │  Mobile / Web / Desktop         │
                          └────────────┬────────────────────┘                                                                                                                                                    
                                       │ WebSocket / HTTP
                          ┌────────────▼────────────────────┐                                                                                                                                                    
                          │      Global Load Balancer       │                                                                                                                                                   
                          │   (Anycast / GeoDNS routing)    │
                          └──┬──────────┬──────────┬────────┘                                                                                                                                                    
                             │          │          │
                ┌────────────▼──┐  ┌────▼──────┐  ┌▼───────────┐                                                                                                                                                 
                │  WS-Server-1  │  │WS-Server-2│  │WS-Server-N │  (~2000 servers)                                                                                                                                
                │  50k conns    │  │ 50k conns │  │ 50k conns  │                                                                                                                                                 
                └──────┬────────┘  └─────┬─────┘  └─────┬──────┘                                                                                                                                                 
                       │                 │               │                                                                                                                                                       
           ┌───────────▼─────────────────▼───────────────▼──────────┐
           │                   Pub/Sub Broker Layer                 │                                                                                                                                         
           │           (Kafka / NATS JetStream / Redis Streams)     │                                                                                                                                         
           │                                                        │                                                                                                                                         
           │   topic: room:123    topic: room:456    topic: room:789│                                                                                                                                         
           └───────────┬────────────────────────────────────────────┘                                                                                                                                           
                       │                                                                                                                                                                                         
           ┌───────────▼─────────────────────────────────────────────┐                                                                                                                                          
           │                    Service Layer                        │                                                                                                                                         
           │  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │
           │  │  Message   │  │  Presence  │  │  Room/Member     │   │                                                                                                                                           
           │  │  Service   │  │  Service   │  │  Service         │   │                                                                                                                                           
           │  └─────┬──────┘  └─────┬──────┘  └────────┬─────────┘   │                                                                                                                                           
           └────────┼───────────────┼──────────────────┼─────────────┘                                                                                                                                           
                    │               │                  │                                                                                                                                                         
           ┌────────▼───────────────▼──────────────────▼─────────────┐                                                                                                                                           
           │                    Storage Layer                        │                                                                                                                                         
           │  ┌──────────────┐  ┌────────────┐  ┌────────────────┐   │
           │  │  ScyllaDB /  │  │   Redis    │  │  PostgreSQL    │   │                                                                                                                                           
           │  │  Cassandra   │  │  Cluster   │  │  (metadata)    │   │                                                                                                                                           
           │  │  (messages)  │  │ (presence/ │  │  rooms, users  │   │                                                                                                                                           
           │  └──────────────┘  │  sessions) │  └────────────────┘   │                                                                                                                                           
           │                    └────────────┘                        │                                                                                                                                          
           └──────────────────────────────────────────────────────────┘                                                                                                                                          
                                                                                                                                                                                                                 
---             
Layer 1: WebSocket Connection Servers

At 100M concurrent users each server handles ~50k connections.

100M users ÷ 50k per server = 2000 WebSocket servers

Each server maintains only local state — no global lookup needed for message delivery:

// In-memory state per WebSocket server — no Redis for routing                                                                                                                                                 
type WSServer struct {                                                                                                                                                                                         
// userID → connection (for this server only)
connections map[string]*websocket.Conn

      // roomID → set of local userIDs in that room                                                                                                                                                              
      // This is the key structure — only local users
      roomMembers map[string]map[string]struct{}                                                                                                                                                                 
                  
      // roomID → Pub/Sub subscription handle                                                                                                                                                                    
      // Only one subscription per room, regardless of how many
      // local users are in that room                                                                                                                                                                            
      subscriptions map[string]Subscription                                                                                                                                                                      
  
      mu sync.RWMutex                                                                                                                                                                                            
}

Connection lifecycle

User B connects to WS-Server-7:

1. WebSocket handshake established
2. Auth token validated → userID extracted
3. Fetch User B's room list from Room Service (cached)
4. For each room B is in:                                                                                                                                                                                      
   a. Add B to roomMembers[roomID] locally                                                                                                                                                                     
   b. If first local user in room → subscribe to Pub/Sub topic "room:{id}"                                                                                                                                     
   c. If not first → reuse existing subscription (no extra cost)
5. Update presence: SET presence:userB "server-7" EX 60
6. Fetch undelivered messages (gap fill — see §5)

func (s *WSServer) OnConnect(conn *websocket.Conn, userID string) error {                                                                                                                                      
// Fetch rooms this user participates in                                                                                                                                                                   
rooms, err := s.roomService.GetUserRooms(ctx, userID)
if err != nil {                                                                                                                                                                                            
return fmt.Errorf("fetch rooms: %w", err)
}

      s.mu.Lock()                                                                                                                                                                                                
      s.connections[userID] = conn
                                                                                                                                                                                                                 
      for _, roomID := range rooms {
          // Track local membership
          if s.roomMembers[roomID] == nil {
              s.roomMembers[roomID] = make(map[string]struct{})                                                                                                                                                  
          }
                                                                                                                                                                                                                 
          isFirstLocal := len(s.roomMembers[roomID]) == 0                                                                                                                                                        
          s.roomMembers[roomID][userID] = struct{}{}
                                                                                                                                                                                                                 
          if isFirstLocal {
              // Subscribe ONCE per room per server — not once per user
              // If 10,000 local users are in room:123, still only 1 subscription                                                                                                                                
              sub := s.pubsub.Subscribe("room:" + roomID)                                                                                                                                                        
              s.subscriptions[roomID] = sub                                                                                                                                                                      
              go s.handleRoomMessages(roomID, sub)                                                                                                                                                               
          }                                                                                                                                                                                                      
      }           
      s.mu.Unlock()                                                                                                                                                                                              
                  
      s.presence.SetOnline(ctx, userID, s.serverID)                                                                                                                                                              
      return nil
}

func (s *WSServer) OnDisconnect(userID string) {
s.mu.Lock()
defer s.mu.Unlock()

      delete(s.connections, userID)
                                                                                                                                                                                                                 
      for roomID, members := range s.roomMembers {
          delete(members, userID)
                                                                                                                                                                                                                 
          // Last local user left this room → unsubscribe
          // Stops receiving pub/sub messages nobody here needs                                                                                                                                                  
          if len(members) == 0 {                                                                                                                                                                                 
              s.subscriptions[roomID].Unsubscribe()
              delete(s.subscriptions, roomID)                                                                                                                                                                    
              delete(s.roomMembers, roomID)
          }                                                                                                                                                                                                      
      }           

      s.presence.SetOffline(ctx, userID)
}

Why this is efficient

Without this model:                    With this model:                                                                                                                                                        
───────────────────────────────────────────────────────
Room has 1M members                    Same room, 1M members                                                                                                                                                   
Naive: 1M Redis lookups on each msg   1 Pub/Sub subscription per server
2000 servers × 1 sub = 2000 subscriptions                                                                                                                               
(regardless of member count)

Each WS server has 1000 users          Each server: 1 subscription to room:123                                                                                                                                 
in room:123                            Delivers to all 1000 local users from                                                                                                                                   
→ each needs the message               that one subscription — O(1) pub/sub cost
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Layer 2: Message Send Flow

User A (on WS-Server-1) sends "Hello!" to Room-123

Step 1: WS-Server-1 receives the message over WebSocket                                                                                                                                                        
Step 2: Assign message ID (ULID — time-ordered, sortable)
Step 3: Write to Kafka topic "room-messages" (durable log)                                                                                                                                                     
Step 4: Kafka consumer writes to ScyllaDB (async, reliable)                                                                                                                                                    
Step 5: WS-Server-1 publishes to Pub/Sub topic "room:123"                                                                                                                                                      
Step 6: WS-Server-2, WS-Server-7, WS-Server-N (subscribers) receive it                                                                                                                                         
Step 7: Each server fans out to its local members of Room-123                                                                                                                                                  
Step 8: Deliver over WebSocket to each connected user

func (s *WSServer) OnMessage(userID string, msg IncomingMessage) error {
// 1. Validate & build the message                                                                                                                                                                         
chatMsg := &Message{                                                                                                                                                                                       
ID:        ulid.New(),  // time-ordered ID — enables cursor pagination
RoomID:    msg.RoomID,                                                                                                                                                                                 
SenderID:  userID,                                                                                                                                                                                     
Content:   msg.Content,                                                                                                                                                                                
Timestamp: time.Now().UTC(),                                                                                                                                                                           
}

      // 2. Persist durably first — never publish before persisting                                                                                                                                              
      //    Kafka acts as the write-ahead log
      if err := s.kafka.Publish("room-messages", chatMsg); err != nil {                                                                                                                                          
          return fmt.Errorf("persist message: %w", err)
      }                                                                                                                                                                                                          
                  
      // 3. Publish to Pub/Sub for real-time delivery                                                                                                                                                            
      //    This is best-effort — persistence already handled by Kafka
      if err := s.pubsub.Publish("room:"+msg.RoomID, chatMsg); err != nil {                                                                                                                                      
          // Non-fatal: message is persisted, clients will get it on next sync                                                                                                                                   
          slog.Warn("pubsub publish failed", "room", msg.RoomID, "err", err)                                                                                                                                     
      }                                                                                                                                                                                                          
                                                                                                                                                                                                                 
      // 4. Deliver to sender immediately (echo for confirmation)                                                                                                                                                
      s.deliverToUser(userID, chatMsg)
                                                                                                                                                                                                                 
      return nil  
}

// Called when a pub/sub message arrives for a room this server subscribes to                                                                                                                                  
func (s *WSServer) handleRoomMessages(roomID string, sub Subscription) {
for msg := range sub.Messages() {                                                                                                                                                                          
s.mu.RLock()
members := s.roomMembers[roomID]                                                                                                                                                                       
// Snapshot the member set — avoid holding lock during delivery
local := make([]string, 0, len(members))                                                                                                                                                               
for uid := range members {
local = append(local, uid)                                                                                                                                                                         
}       
s.mu.RUnlock()

          // Fan out to all locally connected users in this room
          for _, uid := range local {                                                                                                                                                                            
              s.deliverToUser(uid, msg)
          }
      }
}

func (s *WSServer) deliverToUser(userID string, msg *Message) {
s.mu.RLock()                                                                                                                                                                                               
conn := s.connections[userID]
s.mu.RUnlock()

      if conn == nil {                                                                                                                                                                                           
          return // user disconnected since we last checked
      }                                                                                                                                                                                                          
                  
      if err := conn.WriteJSON(msg); err != nil {
          slog.Warn("delivery failed", "user", userID, "err", err)
          s.OnDisconnect(userID) // clean up stale connection                                                                                                                                                    
      }                                                                                                                                                                                                          
}
                                                                                                                                                                                                                 
---             
Layer 3: Pub/Sub Broker Comparison and Selection

                    Redis Pub/Sub    Redis Streams    Kafka          NATS JetStream
─────────────────────────────────────────────────────────────────────────────────                                                                                                                              
Persistence       NO               YES              YES            YES
Message replay    NO               YES (by ID)      YES            YES                                                                                                                                         
Consumer groups   NO               YES              YES            YES
Throughput        Very High        High             Very High      Very High                                                                                                                                   
Latency           ~0.1ms           ~1ms             ~5-10ms        ~0.5ms
Fan-out model     Broadcast        Pull/push         Pull           Push/pull                                                                                                                                  
Max scale         ~1M msg/s        ~500k msg/s      ~10M msg/s     ~5M msg/s                                                                                                                                   
Ops complexity    Low              Medium           High           Medium                                                                                                                                      
Best for          Presence/       Small-medium     Large scale    Real-time                                                                                                                                    
ephemeral        chat             chat           chat                                                                                                                                        
─────────────────────────────────────────────────────────────────────────────────

Recommended: NATS JetStream or Kafka depending on scale

// NATS JetStream: lowest latency, simple ops, good for chat
type NATSPubSub struct {                                                                                                                                                                                       
js nats.JetStreamContext                                                                                                                                                                                   
}

func (n *NATSPubSub) Publish(subject string, msg *Message) error {                                                                                                                                             
data, _ := proto.Marshal(msg) // protobuf: 5-10x smaller than JSON
_, err := n.js.Publish(subject, data)                                                                                                                                                                      
return err  
}

func (n *NATSPubSub) Subscribe(subject string) Subscription {
// Durable consumer: survives server restarts, replays missed messages
sub, _ := n.js.Subscribe(subject,                                                                                                                                                                          
nats.Durable("ws-server-"+serverID),                                                                                                                                                                   
nats.DeliverNew(),            // only new messages (existing in DB)                                                                                                                                    
nats.AckExplicit(),           // manual ack — prevents message loss                                                                                                                                    
nats.MaxAckPending(10000),    // backpressure: stop if 10k unacked                                                                                                                                     
)                                                                                                                                                                                                          
return &NATSSubscription{sub}                                                                                                                                                                              
}
                  
---
Layer 4: Fan-Out Strategy — Small vs Large Rooms

The hardest scaling problem is fan-out for large rooms (millions of members).

Room size         Strategy
──────────────────────────────────────────────────────────────                                                                                                                                                 
< 1,000 members   Direct Pub/Sub fan-out (push to all subscribers)
1k–100k members   Tiered fan-out: Pub/Sub → per-region brokers
> 100k members    Pull model: message stored, clients pull on demand                                                                                                                                           
+ push only to active viewers via sliding window

Tiered fan-out for large rooms

Message published to "room:megachannel"                                                                                                                                                                        
│                                                                                                                                                                                                     
▼
Kafka topic: room:megachannel  (1 message written)                                                                                                                                                             
│                                                                                                                                                                                                     
├── Region-US consumers (500 WS servers)
│         └── Each server fans out to local users                                                                                                                                                     
│                                                                                                                                                                                                     
├── Region-EU consumers (400 WS servers)
│         └── Each server fans out to local users                                                                                                                                                     
│                                                                                                                                                                                                     
└── Region-APAC consumers (300 WS servers)
└── Each server fans out to local users

// For massive rooms: fan-out service distributes across regions
type FanoutService struct {                                                                                                                                                                                    
kafka    *kafka.Producer
regional map[string]*kafka.Producer // one per region                                                                                                                                                      
}

func (f *FanoutService) Fanout(msg *Message, roomID string) error {                                                                                                                                            
room, _ := f.roomService.Get(roomID)

      if room.MemberCount < 1000 {                                                                                                                                                                               
          // Small room: direct pub/sub fan-out
          return f.kafka.Publish("room:"+roomID, msg)                                                                                                                                                            
      }                                                                                                                                                                                                          
  
      if room.MemberCount < 100_000 {                                                                                                                                                                            
          // Medium room: fan out per region based on where members are
          regions := f.getActiveRegions(roomID)                                                                                                                                                                  
          for _, region := range regions {
              f.regional[region].Publish("room:"+roomID, msg)                                                                                                                                                    
          }       
          return nil                                                                                                                                                                                             
      }           

      // Large room: write once, clients pull via cursor                                                                                                                                                         
      // Only push to users currently viewing the room (active window)
      f.kafka.Publish("messages", msg)  // global log                                                                                                                                                            
      activeViewers := f.getActiveViewers(roomID)  // users with room open right now                                                                                                                             
      return f.kafka.Publish("room:"+roomID+":active", msg)                                                                                                                                                      
}
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Layer 5: Message Gap Fill — Handling Missed Messages

WebSocket connections drop. Users reconnect. They need missed messages.

User B disconnects at t=100
Messages 101, 102, 103 sent to Room-123                                                                                                                                                                        
User B reconnects at t=200

Without gap fill: B misses 3 messages                                                                                                                                                                          
With gap fill: B sends last_seen_id=100, server returns 101,102,103

// Client sends cursor on reconnect
type ConnectRequest struct {                                                                                                                                                                                   
UserID    string            `json:"user_id"`                                                                                                                                                               
RoomCursors map[string]string `json:"room_cursors"` // roomID → last seen messageID
}

func (s *WSServer) OnReconnect(userID string, req ConnectRequest) error {                                                                                                                                      
// Subscribe to real-time first (before reading history)
// This prevents race: new messages arrive while reading history                                                                                                                                           
for roomID := range req.RoomCursors {                                                                                                                                                                      
s.subscribeRoom(userID, roomID)                                                                                                                                                                        
}

      // Then fetch missed messages per room                                                                                                                                                                     
      for roomID, lastSeenID := range req.RoomCursors {
          missed, err := s.msgStore.GetSince(ctx, roomID, lastSeenID, limit=200)                                                                                                                                 
          if err != nil {                                                                                                                                                                                        
              continue                                                                                                                                                                                           
          }                                                                                                                                                                                                      
          for _, msg := range missed {
              s.deliverToUser(userID, msg)                                                                                                                                                                       
          }       
      }
      return nil
}

// ScyllaDB schema — optimized for time-range queries
// Partition key: roomID + time bucket (prevents hot partitions)                                                                                                                                               
// Clustering key: messageID (ULID — time-ordered)

CREATE TABLE messages (                                                                                                                                                                                        
room_id    TEXT,                                                                                                                                                                                           
bucket     TEXT,       -- "2026-04-17" — limits partition size
message_id TEXT,       -- ULID: time-sortable, globally unique                                                                                                                                             
sender_id  TEXT,                                                                                                                                                                                           
content    TEXT,                                                                                                                                                                                           
type       TEXT,                                                                                                                                                                                           
created_at TIMESTAMP,                                                                                                                                                                                      
PRIMARY KEY ((room_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id ASC)                                                                                                                                                                    
AND compaction = {'class': 'TimeWindowCompactionStrategy',                                                                                                                                                   
'compaction_window_size': 1,                                                                                                                                                               
'compaction_window_unit': 'DAYS'};

-- Query missed messages efficiently
SELECT * FROM messages                                                                                                                                                                                         
WHERE room_id = 'room-123'
AND bucket = '2026-04-17'
AND message_id > '01HZK...' -- last seen ULID                                                                                                                                                                
LIMIT 200;
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Layer 6: Presence at Scale

Naive approach: Redis key per user. At 100M users → 100M Redis key ops just for heartbeats.

Problem

100M users × heartbeat every 30s = 3.3M Redis writes/sec
Not feasible on a single Redis cluster

Solution: Local aggregation + delta publishing

// Each WS server tracks local presence — only publishes CHANGES                                                                                                                                               
type PresenceAggregator struct {                                                                                                                                                                               
serverID string                                                                                                                                                                                            
local    map[string]time.Time // userID → last seen                                                                                                                                                        
redis    *redis.ClusterClient
mu       sync.Mutex                                                                                                                                                                                        
}

// User activity: update local state only (no Redis write per heartbeat)                                                                                                                                       
func (p *PresenceAggregator) Beat(userID string) {
p.mu.Lock()                                                                                                                                                                                                
p.local[userID] = time.Now()
p.mu.Unlock()                                                                                                                                                                                              
}

// Every 10 seconds: batch-write all local presence to Redis                                                                                                                                                   
// 2000 servers × 50k users = 100M entries, but written in server-level batches
func (p *PresenceAggregator) FlushLoop(ctx context.Context) {                                                                                                                                                  
ticker := time.NewTicker(10 * time.Second)
for {                                                                                                                                                                                                      
select {
case <-ticker.C:
p.mu.Lock()
snapshot := p.local
p.mu.Unlock()

              // Single pipeline: set all local user presences with TTL                                                                                                                                          
              pipe := p.redis.Pipeline()                                                                                                                                                                         
              for userID, lastSeen := range snapshot {                                                                                                                                                           
                  if time.Since(lastSeen) < 30*time.Second {
                      // Only write if recently active
                      pipe.Set(ctx,                                                                                                                                                                              
                          "presence:"+userID,
                          p.serverID,                                                                                                                                                                            
                          60*time.Second,  // TTL: auto-expire if server crashes                                                                                                                                 
                      )                                                                                                                                                                                          
                  }                                                                                                                                                                                              
              }                                                                                                                                                                                                  
              pipe.Exec(ctx)
          case <-ctx.Done():
              return
          }
      }
}

Room-level online count (approximation)

// For "142 people online in this room":                                                                                                                                                                       
// Use HyperLogLog for approximate counts — O(1) memory regardless of members                                                                                                                                  
// Exact counts require sorted sets — expensive at scale

// Approximate (HyperLogLog — ~0.8% error, fixed 12KB memory per room)                                                                                                                                         
redis.PFAdd(ctx, "online:room:123", userIDs...)                                                                                                                                                                
count, _ := redis.PFCount(ctx, "online:room:123").Result()

// Exact for small rooms (Sorted Set with timestamp as score)                                                                                                                                                  
redis.ZAdd(ctx, "online:room:123", &redis.Z{
Score:  float64(time.Now().Unix()),                                                                                                                                                                        
Member: userID,
})                                                                                                                                                                                                             
// Expire old entries
redis.ZRemRangeByScore(ctx, "online:room:123", "-inf",                                                                                                                                                         
strconv.FormatInt(time.Now().Add(-60*time.Second).Unix(), 10))
count, _ := redis.ZCard(ctx, "online:room:123").Result()
                  
---                                                                                                                                                                                                            
Layer 7: Multi-Device and Multi-Region

Multi-device: one user, multiple connections

// User logs in on phone AND laptop — both are in Room-123
// Message should reach BOTH connections

// Key: userID maps to multiple connections, potentially on multiple servers                                                                                                                                   
type UserSessions struct {
Sessions []Session                                                                                                                                                                                         
}

type Session struct {                                                                                                                                                                                          
ServerID string
DeviceID string
ConnectedAt time.Time
}

// Each WS server handles this locally for its own connections                                                                                                                                                 
func (s *WSServer) deliverToUser(userID string, msg *Message) {
s.mu.RLock()                                                                                                                                                                                               
conns := s.userConnections[userID] // slice — multiple devices
s.mu.RUnlock()

      for _, conn := range conns {                                                                                                                                                                               
          if err := conn.WriteJSON(msg); err != nil {
              s.removeConnection(userID, conn)
          }                                                                                                                                                                                                      
      }
}                                                                                                                                                                                                              
// Messages delivered to all devices via normal Pub/Sub flow
// No special handling needed — each device is just another connection

Multi-region: global chat

US-EAST region          EU-WEST region
┌────────────────┐      ┌────────────────┐                                                                                                                                                                     
│ WS servers     │      │ WS servers     │
│ Kafka cluster  │◄────►│ Kafka cluster  │                                                                                                                                                                     
│ ScyllaDB       │      │ ScyllaDB       │                                                                                                                                                                     
└────────────────┘      └────────────────┘
│ MirrorMaker2 / Kafka replication                                                                                                                                                                     
│                                                                                                                                                                                                      
Global message log (all regions)

// Messages written to local region's Kafka
// Cross-region replication via Kafka MirrorMaker2                                                                                                                                                             
// Each region is self-sufficient for local delivery
// Cross-region messages replicated with ~50-200ms lag

type RegionalRouter struct {                                                                                                                                                                                   
localKafka  *kafka.Producer                                                                                                                                                                                
globalKafka *kafka.Producer // for cross-region channels                                                                                                                                                   
}

func (r *RegionalRouter) Route(msg *Message, room *Room) error {                                                                                                                                               
if room.IsRegional {
// Local room: only write to local Kafka                                                                                                                                                               
return r.localKafka.Publish("room:"+room.ID, msg)
}                                                                                                                                                                                                          
// Global room: write to global topic, replicated to all regions
return r.globalKafka.Publish("global:room:"+room.ID, msg)                                                                                                                                                  
}
  
---                                                                                                                                                                                                            
Layer 8: Backpressure and Slow Consumers

A slow WS server (e.g., GC pause) shouldn't block message flow for everyone else.

// Each WS server has a per-room delivery queue with overflow protection
type RoomDeliveryQueue struct {                                                                                                                                                                                
roomID  string
queue   chan *Message                                                                                                                                                                                      
dropped int64  // metric: messages dropped for slow consumers
}

func NewRoomDeliveryQueue(roomID string) *RoomDeliveryQueue {                                                                                                                                                  
q := &RoomDeliveryQueue{
roomID: roomID,                                                                                                                                                                                        
queue:  make(chan *Message, 1000), // buffered
}                                                                                                                                                                                                          
go q.drain()
return q                                                                                                                                                                                                   
}

func (q *RoomDeliveryQueue) Enqueue(msg *Message) {
select {
case q.queue <- msg:
// delivered to queue
default:                                                                                                                                                                                                   
// Queue full: server is overloaded
// Drop message from this slow server — client will gap-fill on reconnect                                                                                                                              
atomic.AddInt64(&q.dropped, 1)                                                                                                                                                                         
slog.Warn("delivery queue full — dropping message",                                                                                                                                                    
"room", q.roomID,                                                                                                                                                                                  
"dropped_total", atomic.LoadInt64(&q.dropped),                                                                                                                                                     
)                                                                                                                                                                                                      
}
}

func (q *RoomDeliveryQueue) drain() {
for msg := range q.queue {
q.deliverToLocalMembers(msg)
}                                                                                                                                                                                                          
}
                                                                                                                                                                                                                 
---             
Complete Message Flow — End to End

User A (phone, US-EAST, WS-Server-5) sends "Hello!" to Room-123
────────────────────────────────────────────────────────────────

t=0ms:   WS-Server-5 receives message over WebSocket                                                                                                                                                           
t=1ms:   Assign message ID: 01JZKT4X2... (ULID)
t=2ms:   Write to Kafka topic "room-messages" (partition: hash(room-123))                                                                                                                                      
t=2ms:   Kafka acknowledges write (durable)                                                                                                                                                                    
t=3ms:   Publish to NATS topic "room:123"                                                                                                                                                                      
t=3ms:   WS-Server-5 delivers echo to User A (confirmation)

t=3ms:   NATS delivers to all subscribers of "room:123":                                                                                                                                                       
WS-Server-5  (has User A, User C locally)
WS-Server-7  (has User B locally)                                                                                                                                                                  
WS-Server-12 (has User D, User E locally)

t=4ms:   Each WS server fans out to local members                                                                                                                                                              
t=4ms:   User B receives "Hello!" on their phone

t=5ms:   Kafka consumer writes message to ScyllaDB (async)                                                                                                                                                     
Partition key: (room-123, 2026-04-17)
Clustering key: 01JZKT4X2...

t=50ms:  ScyllaDB write confirmed (message durable in storage)

Later:   User F reconnects after being offline                                                                                                                                                                 
Sends: {room_cursors: {"room-123": "01JZKT3..."}}
Server fetches messages since 01JZKT3... from ScyllaDB                                                                                                                                                
Returns: ["Hello!", ...] — gap filled
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Scaling Numbers

Metric                           Value             Notes
────────────────────────────────────────────────────────────────────                                                                                                                                           
Concurrent connections           100M
WS servers needed                2,000             50k conns each                                                                                                                                              
Pub/Sub subscriptions            ~20M total        avg 10 rooms/server                                                                                                                                         
2000 × 10,000 rooms                                                                                                                                         
Messages per second              ~500k/s           peak                                                                                                                                                        
Fan-out deliveries/sec           ~5M/s             avg 10 recipients                                                                                                                                           
Kafka throughput needed          ~2 GB/s           1KB avg message                                                                                                                                             
ScyllaDB write throughput        ~500k writes/s    1:1 with messages                                                                                                                                           
Redis presence ops               ~200k ops/s       batch heartbeats                                                                                                                                            
Network per WS server            ~10 Gbps          peak delivery                                                                                                                                               
Memory per WS server             ~50 GB            50k conns × 1MB each

Pub/Sub subscription cost:                                                                                                                                                                                     
Without model: 100M users × 10 rooms = 1B subscriptions                                                                                                                                                      
With model:    2000 servers × 10k unique rooms = 20M subscriptions                                                                                                                                           
Reduction:     50x fewer subscriptions
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Summary: Why the Pub/Sub Fanout Model Works

Property              Naive (Redis Lookup)     Pub/Sub Fanout
──────────────────────────────────────────────────────────────────                                                                                                                                             
Message routing       Query Redis per user     Subscribe per room per server
Routing cost          O(members)               O(1) publish + O(local members)                                                                                                                                 
Redis operations      N per message send       Batch heartbeats only
WS server coupling    Tight (needs global map) Loose (local state only)                                                                                                                                        
Subscription count    1 per user               1 per room per server                                                                                                                                           
New user join cost    Update global map        Subscribe locally if first in room                                                                                                                              
Server crash impact   All user sessions lost   Other servers unaffected                                                                                                                                        
Scale ceiling         Redis throughput         Kafka throughput (~10M msg/s)                                                                                                                                   
Gap fill              Hard (who missed what?)  Easy (cursor on Kafka/ScyllaDB)                                                                                                                                 
Multi-device          Requires coordination    Natural (multiple local conns)

The core insight: by shifting from "where is each user?" to "which servers care about this room?", you reduce the routing problem from O(users) to O(servers × rooms_per_server) — a number that stays bounded
regardless of how many users join a room.  
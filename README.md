# Truck_Platooning
# Truck_Platooning
Tasks 1 and 2 — first version Distributed Systems — Semester Project · FH Dortmund

Task 1 — Data, signals, events and protocol
1.1 Data and signals required for the communication

Sent once, when a truck joins

Name	Type	Bytes	Meaning
truck_id	uint16	2	Identifier of the truck
platoon_id	uint16	2	Identifier of the platoon
predecessor_id	uint16	2	The truck directly in front; 0 if this truck is the leader
successor_id	uint16	2	The truck directly behind; 0 if this truck is last
role	uint8	1	LEADER / FOLLOWER
vehicle_length	float	4	Needed to compute the gap
max_decel	float	4	Braking capability

Sent continuously

Name	Type	Bytes	Unit	Rate
speed	float	4	m/s	20 Hz
acceleration	float	4	m/s²	20 Hz
position_gps	2 x float	8	degrees	20 Hz
brake_active	bool	1	—	20 Hz
sensor_ok	bool	1	—	20 Hz

Measured locally, not communicated

Name	Source	Meaning
gap_measured	Radar / Lidar	Distance to the truck in front
own_speed	Wheel sensor	Own speed
own_accel	IMU	Own acceleration
1.2 Events
Event	Direction	Acknowledged
JOIN_REQUEST	follower → leader	yes
JOIN_ACCEPT	leader → follower	—
JOIN_REJECT	leader → follower	—
LEAVE_REQUEST	follower → leader	yes
LEAVE_ACK	leader → follower	—
GAP_OPEN_REQUEST	follower N → leader	yes
GAP_OPEN_COMMAND	leader → follower N+1	yes
GAP_CLOSE_COMMAND	leader → follower N+1	yes
EMERGENCY_BRAKE	any → all (broadcast)	no
1.3 Protocol
Message frame

Every message has the same fixed size of 32 bytes: a 13 byte header and a 19 byte payload. Messages with less content than 19 bytes are padded. This removes the need for a length field, lets the receiver read every field at a fixed offset, and makes the transmission time of every message identical.

+-----------+-------------+--------+---------+-----------+----------+----------+
| sender_id | platoon_id  |  type  | seq_num | timestamp | payload  | checksum |
|  2 byte   |   2 byte    | 1 byte | 2 byte  |  4 byte   | 19 byte  |  2 byte  |
+-----------+-------------+--------+---------+-----------+----------+----------+
<------------------- header, 13 byte -------------------><-- 19 --><--- 2 ---->
<--------------------------- total 32 byte ------------------------------------>

Header fields

Field	Type	Purpose
sender_id	uint16	Which truck sent the message
platoon_id	uint16	Which platoon the message belongs to
type	uint8	STATUS, JOIN_REQUEST, EMERGENCY_BRAKE, ...
seq_num	uint16	Detects lost messages and discards duplicates
timestamp	uint32	Time the message was created, in ms
checksum	uint16	Detects corrupted messages

Payload by message type

Type	Content	Used bytes
STATUS	speed, acceleration, position_gps, brake_active, sensor_ok	18
JOIN_REQUEST	vehicle_length, max_decel	8
JOIN_ACCEPT	platoon_id, predecessor_id, successor_id, role	7
GAP_OPEN_REQUEST	successor_id of the requesting truck	2
EMERGENCY_BRAKE	empty, the type alone carries the meaning	0
others	empty	0

The largest payload is STATUS with 18 bytes, so 19 bytes is enough for every message. At 32 bytes and 20 messages per second, one truck sends about 640 bytes per second.

Rules

Status messages carrying the continuous data are broadcast every 50 ms and are not acknowledged.
There is no separate heartbeat. Because the status message arrives on a fixed schedule, its absence is itself the signal that the link is gone.
If no message from the predecessor arrives within 300 ms, that is six missed status messages, and the link is declared lost.
Acknowledged events are retransmitted up to three times, then treated as failed.
EMERGENCY_BRAKE has the highest priority, is broadcast, and is acted on without waiting for an acknowledgement.
seq_num detects lost messages and discards duplicates. Data older than 200 ms is discarded.
The leader broadcasts to all followers. In addition, every truck sends its data to the truck named in its successor_id.
A gap for crossing traffic is requested by the follower that detects it, not by the leader. The leader cannot see a junction further back in the platoon. The leader grants the request and commands the truck behind the gap to open it.
1.4 State machine — communication

One instance runs in every truck. This machine describes platoon membership only. How the truck drives while a message is missing is described by the control state machine in Section 2.4.

                        +----------------+
              +-------->|   STANDALONE   |<---------------+
              |         +-------+--------+                |
              |                 | driver requests          |
              |                 | platooning               |
              |                 | / send JOIN_REQUEST      |
              |                 v                          |
              |         +----------------+                 |
              |         |    JOINING     |                 | LEAVE_ACK
  JOIN_REJECT |         +-------+--------+                 | received, or
  or 3 x      |                 | JOIN_ACCEPT              | silence > 5 s
  timeout     |                 v                          |
              |         +----------------+                 |
              +---------+   IN_PLATOON   +-----------------+
                        |                |
                        |  +----------+  |  driver requests exit
                        |  |  LEADER  |  |  / send LEAVE_REQUEST
                        |  +----------+  +--------------------> LEAVING
                        |  | FOLLOWER |  |
                        |  +----------+  |
                        +----------------+

Transitions

From	Event / Guard	To	Action
STANDALONE	driver requests platooning	JOINING	send JOIN_REQUEST
JOINING	JOIN_ACCEPT received	IN_PLATOON	store platoon_id, predecessor_id, successor_id, role
JOINING	JOIN_REJECT or 3 timeouts	STANDALONE	notify driver
IN_PLATOON	silence > 5 s	STANDALONE	forced decouple, notify driver
IN_PLATOON	driver requests exit	LEAVING	send LEAVE_REQUEST
LEAVING	LEAVE_ACK received	STANDALONE	relink neighbours, open gap, release control
1.5 Structure of the platoon

The platoon is modelled as a doubly linked list. The leader is the head. Every truck stores predecessor_id and successor_id, so no truck holds a position number that would have to be renumbered when another truck joins or leaves. A position can still be derived when needed by walking the list from the head.

   leader                follower              follower
  +--------+   next     +--------+   next     +--------+
  |   A    | ---------> |   B    | ---------> |   C    |
  |        | <--------- |        | <--------- |        |
  +--------+   prev     +--------+   prev     +--------+
  pred = 0                pred = A              pred = B
  succ = B                succ = C              succ = 0

Joining

A truck joins at the end of the list. The leader answers JOIN_REQUEST with a JOIN_ACCEPT containing predecessor_id and successor_id, and informs the current last truck of its new successor. Only two trucks change their stored data.

Leaving

Leaving is a linked list delete. The departing truck B tells its predecessor A to set successor_id = C, and tells its successor C to set predecessor_id = A. The remaining trucks are unaffected.

   before:   A --> B --> C
   after:    A --------> C

Repairing a broken chain

If a truck disappears without leaving properly, its successor stops receiving messages and decouples after 5 s, which leaves a gap in the list. That truck then sends a JOIN_REQUEST to the leader and is given a new predecessor_id. The leader keeps a copy of the whole list, so it can approve joins, approve gap requests, and repair the chain.

Task 2 — Control behaviour
2.1 How the distance to the preceding truck is guaranteed
d_desired = d_min + h * v_own
d_min = 5 m, the standstill gap
h = time headway, depends on the mode (Section 2.2)
v_own = own speed

Control law

e     = gap_measured - d_desired          (from the own radar)
a_fb  = Kp*e + Ki*INTEGRAL(e) + Kd*de/dt  (PID feedback)
a_ff  = accel_predecessor                 (feedforward, received)

link OK:     a_cmd = a_fb + a_ff
link lost:   a_cmd = a_fb

a_cmd = clamp(a_cmd, a_min, a_max)        (physical limits)
The PID term uses the radar, so the gap is held even without communication.
When the link is lost the feedforward term is simply not available, and the headway h is increased to compensate for the later reaction.
The feedforward term uses the received acceleration of the predecessor, so the follower brakes at the same moment instead of after it.
2.2 Operating modes
d_critical = v_own^2 / (2 * max_decel) + safety_margin
Mode	Trigger	h	Behaviour
NORMAL	link OK and sensors OK	0.5 s	feedforward used, up to 90 km/h
DEGRADED	no message for > 300 ms	1.5 s	feedforward unavailable, up to 70 km/h
EMERGENCY	gap < d_critical, or EMERGENCY_BRAKE	—	full braking
GAP_OPENING	GAP_OPEN_COMMAND received	3.0 s	feedforward used, up to 90 km/h
DECOUPLED	degraded > 5 s, or leave completed	—	manual driving
2.3 Communication failure — is the system robust?

Yes. Three reasons:

The gap is measured by the own radar, so no safety function depends on the radio.
On link loss the feedforward term is dropped and the headway is increased from 0.5 s to 1.5 s, with the speed capped at 70 km/h. The later reaction is compensated by the larger gap.
If the link does not return within 5 s, the truck leaves the platoon and returns to manual driving, so it never stays degraded indefinitely.

If the radar fails as well, this is detected through sensor_ok and leads directly to EMERGENCY_BRAKE and then DECOUPLED.

2.4 State machine — control behaviour
                           +---------------------+
                           |     GAP_OPENING     |
                           |      h = 3.0 s      |
                           +--^---------------+--+
        GAP_OPEN_COMMAND     |                 | GAP_CLOSE_COMMAND
                             |                 v
          +----------------------+
          |   NORMAL_FOLLOWING   |
          |      h = 0.5 s       |
          +--+----------------^--+
             | link lost      | link restored
             v                |
          +----------------------+
          |  DEGRADED_FOLLOWING  |
          |      h = 1.5 s       |
          +--+----------------+--+
             | timer > 5 s    |
             v                |
          +--------------+    |
          |  DECOUPLED   |    |
          +--------------+    |
                              |
   +--------------------------+-------------------+
   |  from ANY state:                             |
   |  gap < d_critical OR EMERGENCY_BRAKE received|
   |              v                               |
   |      +--------------------+                  |
   |      |  EMERGENCY_BRAKE   |--> broadcast to  |
   |      |  full deceleration |    trucks behind |
   |      +---------+----------+                  |
   |                | vehicle stopped / safe      |
   |                v                             |
   |          DECOUPLED                           |
   +----------------------------------------------+
2.5 Activity diagram — one control cycle (every 50 ms)
                 ( start cycle )
                       |
                       v
            [ read own sensors ]
            speed, gap_measured
                       |
                       v
           [ read message queue ]
      predecessor speed, acceleration, brake
                       |
                       v
        < status message within 300 ms? >
              | no              | yes
              v                 v
      [ mode = DEGRADED ]  [ mode = NORMAL ]
              |                 |
              +--------+--------+
                       v
        < gap < d_critical OR
          EMERGENCY_BRAKE received? >
              | yes             | no
              v                 |
   [ mode = EMERGENCY ]         |
   [ broadcast EMERG_BRAKE ]    |
   [ a_cmd = a_min ]            |
              |                 v
              |      [ compute d_desired ]
              |      [ e = gap - d_desired ]
              |      [ a_fb = PID(e) ]
              |      [ a_cmd = a_fb + a_ff if link OK ]
              |                 |
              +--------+--------+
                       v
         [ clamp a_cmd to physical limits ]
                       |
                       v
            [ send to actuators ]
                       |
                       v
          [ broadcast own status ]
      speed, acceleration, brake, sensor_ok
                       |
                       v
                 ( end cycle )

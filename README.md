# Multithreaded server in C++
This repository contains an autonomous robot control server written in c++ designed to remotely manage a fleet of robots. The robots are capable of independently connecting to the server, which then guides them towards the center of a coordinate system. As part of the testing process, each robot starts at random coordinates and attempts to reach the target coordinates [0,0]. Upon reaching the destination, the robot must retrieve a valuable secret.

## Detailed Specification

Communication between the server and robots is fully realized using a text-based protocol. Each command is terminated with a pair of special symbols "\a\b". (These are two characters '\a' and '\b'.) The server must strictly adhere to the communication protocol but should consider imperfect robot firmware (see the Special Situations section).

### Server Messages:
| Name                          | Message                                   | Description                                                                   |
|-------------------------------|-------------------------------------------|-------------------------------------------------------------------------------|
| SERVER_CONFIRMATION           | `<16-bit decimal number>\a\b`           | Message with a confirmation code. It can contain a maximum of 5 digits and the termination sequence \a\b. |
| SERVER_MOVE                   | 102 MOVE\a\b                            | Command to move one square forward.                                            |
| SERVER_TURN_LEFT              | 103 TURN LEFT\a\b                       | Command to turn left.                                                         |
| SERVER_TURN_RIGHT             | 104 TURN RIGHT\a\b                      | Command to turn right.                                                        |
| SERVER_PICK_UP                | 105 GET MESSAGE\a\b                     | Command to pick up a message.                                                 |
| SERVER_LOGOUT                 | 106 LOGOUT\a\b                          | Command to terminate the connection after successfully retrieving a message. |
| SERVER_KEY_REQUEST            | 107 KEY REQUEST\a\b                     | Server's request for a Key ID for communication.                              |
| SERVER_OK                     | 200 OK\a\b                              | Affirmative acknowledgment.                                                   |
| SERVER_LOGIN_FAILED           | 300 LOGIN FAILED\a\b                    | Authentication failed.                                                        |
| SERVER_SYNTAX_ERROR           | 301 SYNTAX ERROR\a\b                    | Incorrect message syntax.                                                     |
| SERVER_LOGIC_ERROR            | 302 LOGIC ERROR\a\b                     | Message sent in the wrong situation.                                          |
| SERVER_KEY_OUT_OF_RANGE_ERROR | 303 KEY OUT OF RANGE\a\b                | Key ID is out of the expected range.  

### Client Messages:
| Name                 | Message                                       | Description                                                                                                                               | Example           | Maximum Length |
|----------------------|-----------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------|---------------|
| CLIENT_USERNAME      | `<user name>\a\b`                            | Message with a username. The name can be any character sequence except for the termination sequence \a\b and will never match the content of the CLIENT_RECHARGING message.    | Umpa_Lumpa\a\b    | 20            |
| CLIENT_KEY_ID        | `<Key ID>\a\b`                               | Message containing the Key ID. It can only contain a whole number with a maximum of three digits.                                       | 2\a\b             | 5             |
| CLIENT_CONFIRMATION  | `<16-bit decimal number>\a\b`                | Message with a confirmation code. It can contain a maximum of 5 digits and the termination sequence \a\b.                                 | 1009\a\b          | 7             |
| CLIENT_OK            | OK `<x>` `<y>\a\b`                           | Confirmation of a movement, where x and y are integer coordinates of the robot after executing a movement command.                         | OK -3 -1\a\b      | 12            |
| CLIENT_RECHARGING    | RECHARGING\a\b                               | The robot started recharging and stopped responding to messages.                                                                          |                   | 12            |
| CLIENT_FULL_POWER    | FULL POWER\a\b                              | The robot has replenished energy and is now accepting commands.                                                                           |                   | 12            |
| CLIENT_MESSAGE       | `<text>\a\b`                                 | Text of the retrieved secret message. It can contain any characters except for the termination sequence \a\b and will never match the content of the CLIENT_RECHARGING message. | Haf!\a\b          | 100           |

### Time Constants:

| Name                 | Value [s] | Description                                                  |
|----------------------|-----------|--------------------------------------------------------------|
| TIMEOUT              | 1         | Both server and client expect a response within this interval.               |
| TIMEOUT_RECHARGING   | 5         | Time interval during which the robot must complete recharging.  |


Communication with robots can be divided into several phases: 

### Authentication

Both the server and the client are aware of five pairs of authentication keys (these are not public and private keys):
| Key ID | Server Key | Client Key |
|--------|------------|------------|
| 0      | 23019      | 32037      |
| 1      | 32037      | 29295      |
| 2      | 18789      | 13603      |
| 3      | 16443      | 29533      |
| 4      | 18189      | 21952      |

Each robot starts communication by sending its username (CLIENT_USERNAME message). The username can be any sequence of 18 characters, except for the termination sequence "\a\b". In the next step, the server prompts the client to send the Key ID (SERVER_KEY_REQUEST message), which is essentially the identifier of the key pair it wants to use for authentication. The client responds with the CLIENT_KEY_ID message, in which it sends the Key ID. After that, the server knows the correct key pair, so it can calculate the "hash" code from the username using the following formula:

Username: Mnau!

ASCII representation: 77 110 97 117 33

Resulting hash: ((77 + 110 + 97 + 117 + 33) * 1000) % 65536 = 40784

The resulting hash is a 16-bit decimal number. The server then adds its server key to the hash, and if the value overflows the 16-bit capacity, the value simply wraps around (the example below shows Key ID 0):

(40784 + 23019) % 65536 = 63803

The resulting server confirmation code is sent to the client as text in the SERVER_CONFIRM message. The client calculates the hash back from the received code and compares it with the expected hash, which it calculated from the username. If they match, the client creates its own confirmation code. The calculation of the client's confirmation code is similar to the server's, but it uses the client key (the example below shows Key ID 0):

(40784 + 32037) % 65536 = 7285

The client sends its confirmation code to the server in the CLIENT_CONFIRMATION message, and the server calculates the hash back from it and compares it with the original hash of the username. If both values match, the server sends the SERVER_OK message; otherwise, it responds with the SERVER_LOGIN_FAILED message and terminates the connection. The entire sequence is shown in the following diagram:

Client Server
​------------------------------------------
CLIENT_USERNAME --->
<--- SERVER_KEY_REQUEST
CLIENT_KEY_ID --->
<--- SERVER_CONFIRMATION
CLIENT_CONFIRMATION --->
<--- SERVER_OK
or
SERVER_LOGIN_FAILED
.
.
.

The server does not know the usernames in advance. Therefore, robots can choose any username, but they must know the sets of both client and server keys. The key pairs ensure mutual authentication and prevent the authentication process from being compromised by simple eavesdropping.

### Robot Movement towards the Goal

The robot can only move forward (SERVER_MOVE) and can perform a 90-degree turn to the right (SERVER_TURN_RIGHT) or left (SERVER_TURN_LEFT) in place. After each movement command, it sends a confirmation (CLIENT_OK) message, including its current coordinates. The server does not know the robot's initial position at the beginning of communication. Therefore, the server must determine the robot's position (position and direction) only from its responses. To prevent infinite wandering of the robot in space, each robot has a limited number of movements (only forward steps). The number of movements should be sufficient for the robot to move reasonably toward the goal. The communication example follows. The server first moves the robot forward twice to detect its current state and then guides it towards the target coordinates [0,0].

Client Server
​------------------------------------------
.
.
.
<--- SERVER_MOVE
or
SERVER_TURN_LEFT
or
SERVER_TURN_RIGHT
CLIENT_OK --->
<--- SERVER_MOVE
or
SERVER_TURN_LEFT
or
SERVER_TURN_RIGHT
CLIENT_OK --->
<--- SERVER_MOVE
or
SERVER_TURN_LEFT
or
SERVER_TURN_RIGHT
.
.
.

Immediately after authentication, the robot expects at least one movement command - SERVER_MOVE, SERVER_TURN_LEFT, or SERVER_TURN_RIGHT! It is not possible to try to pick up the secret during the first attempt. There are many obstacles on the way to the goal that robots must bypass. The following rules apply to obstacles:

An obstacle occupies a single coordinate.
It is guaranteed that each obstacle has all eight surrounding coordinates free (so it can always be easily bypassed).
It is guaranteed that the obstacle never occupies the coordinate [0,0].
If the robot hits an obstacle more than twenty times, it becomes damaged, and the connection is terminated.
The obstacle is detected when the robot receives a forward movement command (SERVER_MOVE), but there is no change in coordinates (the CLIENT_OK message contains the same coordinates as in the previous step). If the movement is not executed, the remaining steps of the robot are not decremented.

### Picking Up the Secret Message

After the robot reaches the target coordinates [0,0], it attempts to pick up the secret message (SERVER_PICK_UP message). If the robot is asked to pick up the message and is not at the target coordinates, it initiates self-destruction, and the communication with the server is terminated. When attempting to pick up the message at the target coordinates, the robot responds with the CLIENT_MESSAGE message. The server must respond to this message with the SERVER_LOGOUT message. (It is guaranteed that the secret message never matches the CLIENT_RECHARGING message. If the server receives this message after a request to pick up the message, it is always related to recharging.) After that, both the client and the server terminate the connection. Communication example with message pickup follows:

Client Server
​------------------------------------------
.
.
.
<--- SERVER_PICK_UP
CLIENT_MESSAGE --->
<--- SERVER_LOGOUT

### Charging

Each of the robots has a limited energy source. If the battery starts running out, the robot notifies the server and then starts recharging itself from the solar panel. During recharging, it does not respond to any messages. After recharging is complete, it informs the server and resumes its activity from where it left off before recharging. If the robot does not end the recharging within the TIMEOUT_RECHARGING time interval, the server terminates the connection.

Client Server
​------------------------------------------
CLIENT_USERNAME --->
<--- SERVER_CONFIRMATION
CLIENT_RECHARGING --->

Copy code
  ...
CLIENT_FULL_POWER --->
CLIENT_OK --->
<--- SERVER_OK
or
SERVER_LOGIN_FAILED
.
.
.

Another example:

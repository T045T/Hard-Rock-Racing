Hard Rock Racing communication protocol:
========================================

TRANSPORT
---------
The transport protocol used is line-based TCP on port 1993. The server accepts UNIX-style
line endings (\n) and windows-style line endings (\r\n). When reading from the
socket, make sure that you don't limit the size of your lines, as some packages
(e.g. the transmission of the map) may end up rather large.

CODEC & FORMAT
--------------
All communication between the server and the clients is done via JSON
(Description and a wide range of implementations at http://www.json.org/ ).

HANDSHAKE
---------
A connecting AI must send a handshake within 10 seconds of connecting, otherwise
the connection is dropped. It has the following form:

```json
{"message":"connect",
 "type":"player",
 "name":NAME, // The display name of the AI. Must be less than 12 letters
 "character":CHARACTER,
 "cartype":CARTYPE,
 "tracktiled":true/false
}
```
The tracktiled field is optional. If set to true, the map will not be transmitted
to you as one big image, but rather as a sequence of tiles, which the ai can
set together as it pleases.
In the "character" field, an AI can state a preference for which character it
wants to use. This determines which portrait the AI uses as well as the color of
the car. If the character is already used, a random one will be assigned. 
Neither the character nor the car has any effect on gameplay.
Available Characters are:
* Cyberhawk (Grey)
* Ivanzypher (Yellow)
* Jake Badlands (Black)
* Katarina Lyons (Green)
* Rip (Red)
* Snake Sanders (Blue)
* Tarquinn (Purple)
* Viper Mackay (Orange)
Available Car types are:
* Airblade
* Battle Trak
* Dirt Devil
* Marauder
* Havac
Note that the same car can be picked several times.
If the handshake was successful, the server will respond with

```json
{"message":"connect",
 "status":true
}
```
regardless of whether the requested character was available or not.
The handshake will fail if the chosen name is already in use.

GAMESTART
---------
Only four racers can race at a time. By connecting, you queue up and will race
when it is your turn. Only players who are currently racing and observers
receive gamestates and other updates on the race. But other players can still
send requests.
Before the race starts, a gamestart package is sent. It is also sent to any
player joining while the race is in progress.

```json
{"message":"gamestart",
 "players":[PLAYER1, PLAYER2, PLAYER3, PLAYER4],
 "laps":laps,
 "track":TRACK
}
```
PLAYER1 through PLAYER4 are the competing AIs (see below). If the name of your
AI is not in this list, it is not racing in this race. 
After the gamestart package is sent, there is a brief time for the AIs to
process the map data.

TRACK
-----

### Non-Tiled

The tiled track object looks as follows

```json
{"message":"track",
 "tiled":false,
 "width": WIDTH,
 "height":HEIGHT,
 "startdir":"UP/DOWN/LEFT/RIGHT",
 "data":[NUMBER, NUMBER, ... ]
}
```
The startdir field describes which direction the players will face when starting.
DOWN is defined as negative y, LEFT as negative x.
"data" is a one-dimensional array representing the track. It can be thought of
as an image. The point x, y on the track is in x + y * width in the data array. 
Note that this might be very long.
Areas outside the track have the value 0x000000
Areas inside the track that are not the finish line have the value 0xffffff
The finish line has the value 0x00ff00

### Tiled

The tiled track object looks as follows

```json
{"message":"track",
 "tiled":true,
 "tiles":[TILE, ...]
}
```

with the elements of "tiles" being tile objects

```json
{"message":"tile",
 "type":"Curve/Straight/FinishLine",
 "orientation":"UP/DOWN/LEFT/RIGHT"
}
```
The starting direction is the orientation of the finish tile. Other than that,
the finish tile is the same as a straight tile.

GAMESTATE
---------
The server will periodically (10-30 times per second) send a gamestate package
which describes the locations of all players and other dynamic objects
(currently only missiles). The sending of the first gamestate package marks the
start of the race.

```json
{"message":"gamestate",
 "time":SERVERTIME, // time since the race started, in seconds
 "cars":[CAR1, CAR2, ...],
 "missiles":[MISSILE1, ...],
 "mines":[MINE1, ...]
}
```
Note there will at most be 4 Cars on the track at any time.

PLAYER
------

```json
{"message":"player",
 "name":NAME,
 "character":CHARACTER,
 "cartype":CARTYPE,
}
```

CAR
---
```json
{"message":"car",
 "id":ID,
 "driver":playername,
 "hp":hitpoints,
 "locationX":locX,
 "locationY":locY,
 "speedX":speedX,
 "speedY":speedY,
 "facing":facing,	// in radians, 0 is towards positive x
 "missiles":missiles,
 "boosts":boosts,
 "mines":mines,
 "lapscomplete":lapscomplete,
 "accelerating":true/false,
 "turning":TURNING
}
```
TURNING is 0 if the car is not turning, 1 if it is turning right or -1 if it is
turning left

MISSILE
-------

```json
{"message":"missile",
 "locationX":locX,
 "locationY":locY,
 "speedX":speedX,
 "speedY":speedY,
 "shooter":playername
}
```

MINE
----

```json
{"message":"mine",
 "locationX":locX,
 "locationY":locY
}
```

LAPCOMPLETE
-----------
When a player completes a lap, all players are sent a lapcomplete package:
```json
{"message":"lapcomplete",
 "player":playername,
 "lapsleft":lapsleft //full laps the player has yet to complete
}
```

MISSILEHIT
----------
When a player is hit by another player's missile, all players are sent a
missilehit package:
```json
{"message":"missilehit",
 "missile":id,
 "target":playername,
 "shooter":playername
}
```

MINEHIT
-------
When a player drives into a mine, all players get a message
```json
{"message":"minehit",
 "mine":id,
 "target":playername
}
```

DESTROYED
---------
When a player's car is destroyed, all players are sent a destroyed package:
```json
{"message":"destroyed",
 "car":id
}
```
The destroyed car will no longer be in gamestates. When the player respawns with
a new car, the new one will be in the gamestates.

RACEOVER
--------
When all but one player have completed the race, a raceover package is sent to
all players:
```json
{"message":"raceover",
 "placement":[WINNER, SECOND, THIRD, LOSER]
}
```
WINNER etc. are the names of players

OBSERVER
--------
A client can connect to the server as observer. This must be done at the
handshake:
```json
{"message":"connect",
 "type":"observer"
}
```
An observer is sent the same gamestates and information as players, but does not
participate in the game.

ACTION
------
The ai communicates to the server that it wants its car to take an action in
the following format:
```json
{"message":"action",
 "type":type
}
```
type is one of:
- turnleft
- turnright
- accelerate
- stopaccelerate
- stopturning
- missile
- boost
- mine
This package is resent to all players (including the one who sent it) with a 
"player":name field added.

REQUEST
-------
An ai can request the value of a game constant at any time. A request takes the
format:
```json
{"message":"request",
 "key":key //string
}
```
the response:
```json
{"message":"constant",
 "name":name,
 "value":value //or null if not found
}
```
Note the value will be returned as a string
constants are (this list may not be complete):
- car.hitbox.width
- car.hitbox.height
- car.maxfriction
- car.minfriction
- car.acceleration
- car.topspeed
- car.turnspeed
- car.minturnspeed
- car.collision.rotation
- car.collision.bounce
- car.collision.mindamagespeed
- car.hp
- missile.hitbox.width
- missile.hitbox.height
- missile.speed
- missile.range
- missile.damage
- player.maxmissiles
- race.laps
- track.straight.legnth
- track.straight.width
- track.curve.innerradius
- track.curve.outerradius

---
layout: post
title: "The Making of Rift: Part 4, Lobbies"
---

## Introduction

In the last blog post, I talked about how I integrated multi-controller networking into my raycaster. Next, I will tell the tale of how I made a lobby over I2C.

## Starting Off

I really wanted my game to be playable with two, three, or four players.  The problem is, my current handshaking system had to know the number of players in advance. I didn't really want to have to set every device to 2/3/4 players every time you played. Futhermore, if you could ever use I2C in the emulator, Ardens, you wouldn't know how many people you'd have to play with. I really liked the design of the lobbies in Among Us, where you could see the number of players joined and could start at anytime.

## Handshaking

The initial handshaking code I wrote was exactly the same as the original. The only difference is that we don't have to keep track of the number of times we receive a message to know when to start, as that will be done inside the lobby.

```cpp
uint8_t id;
...
void setup_lobby() {
    ...
    for (id = I2C_MAX_PLAYERS - 1; id >= 0; ) {
        uint8_t dummy;
        I2C::read(I2C::getAddressFromId(id), &dummy);
        switch (I2C::getTWError()) {
        case TW_MR_SLA_NACK:
            I2C::setAddress(I2C::getAddressFromId(id), true);

            sprites[id].id = id;
            ...
            I2C::onReceive(handshake_on_receive);
            I2C::onRequest(handshake_on_request);
            state = LOBBY;
            return;
        case TW_SUCCESS:
            id--;
            break;
        }
    }
    ...
}
```

However, I realised I wanted to allow devices to join late, that is, if the game has started jump right in. I cleverly thought of a way to do this. You see, we're already reading from each device but throwing it away. What if instead we sent the `state`? Then, if the `state` was `GAME` or `GAME_INIT`, we could set our `state` to `GAME_INIT` and skip right through the lobby.

```cpp
uint8_t id;
...
void handshake_on_request() {
    I2C::transmit(&state);
}
...
void setup_lobby() {
    ...
    for (id = I2C_MAX_PLAYERS - 1; id >= 0; ) {
        uint8_t otherPlayerState;
        I2C::read(I2C::getAddressFromId(id), &otherPlayerState);
        switch (I2C::getTWError()) {
        case TW_MR_SLA_NACK:
            I2C::setAddress(I2C::getAddressFromId(id), true);

            sprites[id].id = id;
            ...
            I2C::onReceive(handshake_on_receive);
            I2C::onRequest(handshake_on_request);
            state = LOBBY;
            return;
        case TW_SUCCESS:
            id--;
            switch (otherPlayerState) {
            case GAME_INIT:
            case GAME:
                state = GAME_INIT;
                break;
            }
            break;
        }
    }
    ...
}
```
This worked rather well. However, there are two major bugs. If a player with the `id` of `3` was to abandon a player with the `id` of `2`, then if anyone joined again they would not see that the player with id `2` was in a game and so would start the lobby at the same time as the game, which causes major problems. The second bug was that id was a global `uint8_t`, but because the code was looping down, the end condition was less than zero. However, because it was unsigned it would underflow to 255 and loop forever.

The fix I found to loop upwards and talk to every device, regardless if we had already found a position yet. This way, if there was anyone regardless of id had started their game we would find them.

I also optimized away the global variable in the loop to prevent needless loads and stores.

```cpp
uint8_t id;
...
void handshake_on_request() {
    I2C::transmit(&state);
}
void setup_lobby() {
    ...
    id = nullId;
    for (uint8_t i = 0; i < I2C_MAX_PLAYERS; i++) {
        uint8_t otherPlayerState;
        I2C::read(I2C::getAddressFromId(i), &otherPlayerState);
        switch (I2C::getTWError()) {
        case TW_MR_SLA_NACK:
            id = i;
            I2C::setAddress(I2C::getAddressFromId(i), true);

            sprites[i].id = i;
            arduboy.readUnitName((char *)sprites[i].name);
            I2C::onReceive(handshake_on_receive);
            I2C::onRequest(handshake_on_request);
            break;
        case TW_SUCCESS:
            switch (otherPlayerState) {
            case GAME_INIT:
            case GAME:
                state = GAME_INIT;
                break;
            }
            break;
        }
    }
    if (id == nullId)
        // no more slots left
        state = TITLE;
    ...
}
```
Perfect!

## Running the Lobby

When in a lobby, you have to keep track of everyone who is there. This way, you can count the number of players so everyone knows how many players have joined. Unfortunately, you can't just send a message when you power off; the game will stop running! The next best thing is to send a message to remind everyone we're still there. If you don't hear from someone in a while, you assume they've left.

I decided to send the id as the message; that way, players can know who they've received the message from. `nullId` would be used to signify starting the game.

```cpp
void handshake_on_receive() {
    uint8_t *buffer = I2C::getBuffer();
    if (buffer[0] == nullId) {
        state = GAME_INIT;
    } else {
        sprites[buffer[0]].timeout = 1;
    }
}
uint8_t run_timeout() {
    uint8_t numPlayers = 1;
    for (uint8_t i = 0; i < I2C_MAX_PLAYERS; i++) {
        if (sprites[i].timeout) {
            sprites[i].timeout++;
            numPlayers++;
        }
    }
    return numPlayers;
}

void run_lobby() {
    uint8_t numPlayers = run_timeout();
    I2C::write(0x00, &id, false);
        
    font3x5.setCursor(0, 0);
    font3x5.print(numPlayers);
    font3x5.print(F(" of " STR(I2C_MAX_PLAYERS) " players have joined\nPress A to start\nId "));
    font3x5.print(id);
    if (arduboy.justPressed(A_BUTTON)) {
        uint8_t message = nullId;
        I2C::write(0x00, &message, true);
        state = GAME_INIT;
    }
    if (arduboy.pressed(B_BUTTON)) {
        stop_multiplayer();
        state = TITLE;
    }
}
```

The timeout logic is a bit tricky. When a message is sent, we set the id of the received message to one. Not being `0` lets us know that that player has joined. However, if they have joined, we increment it. Because it is a `uint8_t`, if it gets to 256 without a message it will wrap around to `0`. This means they haven't sent a message for too long and we no longer will consider them there.

## For the Rest of the Game

The timeout code continues to run for the duration of the game. That way, if a player leaves during the game everyone will know that they have left.

The timeout code also controls everything from drawing the sprites to the game over rankings at the end to make sure you won't see any non-existant players.
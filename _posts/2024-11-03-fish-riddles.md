---
title: "Post: Halloween Fish-formed Riddle-maker 🦇"
last_modified_at: 2024-11-05
categories:
  - Blog
tags:
  - ic
  - raspberry
  - halloween
  - ai
  - python
  - websockets
---

In early October 2024, my wife found an old singing fish from the early 2000s at a thrift store: the [Big Mouth Billy Bass](https://en.wikipedia.org/wiki/Big_Mouth_Billy_Bass). This fish can sing two songs and move its head, mouth, and tail in rhythm with the music, although the movement is infrequent.

This Fish Riddle Maker became my pet project for almost all evenings during the last two weeks of October 2024.

# Idea

Throughout the summer of 2024, I was reading _"The Art of Game Design"_ by _Jesse Schell_. The book doesn't cover the technical side of game development but rather the design and psychology of games. It was fascinating to read about how game mechanics and "engage-ability" can be achieved for different types of games. At some point, I thought, _"Why not make something engaging and fun to interact with at a Halloween party, even if it’s not a game per se? Many people have interacted with ChatGPT at least once, so would it make a 'wow' effect if I put AI in the fish's body?"_

## A Few Weeks Later

I had a Raspberry Pi 4B lying around, which I usually used as an emulator for NES/Sega Dreamcast. So, I decided to make a riddle-maker out of it. The idea was to have the fish ask a riddle, and if the player answered correctly, the fish would dance with its tail and play the [Shrekophone](https://knowyourmeme.com/memes/shrekophone).

> **Note:** I later added a feature where the fish would ask the player if they wanted a fish fact or another riddle to keep them engaged.

### Thought Process

From the player's point of view, it would look like this:

- The player approaches the fish, which recognizes them and greets them.
- The fish asks a riddle.
- The player answers or says something like, "I don't know."
- The fish reacts to the answer—either dancing or saying, "Would you like to hear a fish fact or maybe another riddle?"

That’s basically it. 

But there were numerous technical challenges:

- How to make the fish recognize the player?
- How to make the fish ask a riddle?
- How to make the fish react to the answer?
- How to make the fish dance?
- I live in the Netherlands, so how do I make the fish speak Dutch? What if the player speaks English? What if they suddenly switch to another language?
- What if the player says something inappropriate? How should the fish react?

To solve these challenges, I decided to tackle them step by step. Some concepts were quite new to me.

#### Fish Should Recognize the Player

I decided to use a Raspberry Pi camera for this. I needed a face recognition system that would recognize the player's face, assign it a random ID, and use this ID within the system.

> **Note:** Saving face patterns to the filesystem raised privacy concerns for me. Although it was only for the Halloween party, I wanted to make it clear to users that their face pattern would be saved. I included the phrase _"I can see you"_—spoken by the fish—to make it clear to the player that the fish could see them. The fish would say this phrase once at the beginning of the conversation for both new and returning players.

#### Fish Should Ask a Riddle

My first thought was to use the OpenAI Realtime API. I did a small proof of concept, but it was unstable for my use case—probably due to the microphone quality—and the costs were too high. For just a few requests to the Realtime API, I was charged around $0.10, which would have been too expensive during testing and the Halloween party.

I decided to switch from the Realtime API to ChatGPT, which meant I needed to handle:

- Voice recognition (speech-to-text) to transcribe what the player said.
- Riddle logic, using [ChatGPT 4 mini](https://platform.openai.com/docs/models#gpt-4o-mini). I didn’t need advanced reasoning, but I did need a server to handle user IDs from the face recognition system and return riddles.
- Text-to-speech for the fish to speak the riddle.

#### Fish Should Dance

Since the fish already had a dancing/singing IC, I needed to trace where the signals were coming from and make the Raspberry Pi send the same signals.

I decided to permanently add an Arduino. It would control the fish’s body when commanded by the Raspberry Pi and act as a "proxy" for the fish’s original IC when the Raspberry Pi was not in use. This way, I wouldn’t need to resolder components after the event, which would have been annoying.

Additionally, the fish IC runs on 5V while the Raspberry Pi runs on 3.3V, so I needed a level shifter to make it work.

#### Fish Should Speak

The Raspberry Pi wasn’t powerful enough to run text-to-speech software with low latency, which was necessary to avoid breaking the player's immersion. To solve this, I decided to offload this task to the server.

The server would receive text from ChatGPT and return an audio file. After some research, I chose [AllTalk TTS](https://github.com/erew123/alltalk_tts), which was fast enough (2-3 sentences) and produced good-quality audio.

The fish already had a speaker, but the sound from the Raspberry Pi’s mini-jack output was too quiet. I used an LM386 amplifier I had on hand, which required 12V.

I also needed to control the fish's mouth movements. Since I was already using an Arduino as a "proxy," I could send commands from the Raspberry Pi to the Arduino to control the mouth.

#### Fish Should Recognize What the Player Said

This was one of the most challenging parts. The Raspberry Pi couldn’t handle speech-to-text processing, so I offloaded this task to the server. The server would receive audio from the Raspberry Pi, process it, and return the result.

I initially tested various speech-to-text models and settled on [OpenAI Whisper](https://github.com/openai/whisper). However, I discovered that Whisper could loop indefinitely if the audio was noisy or the player spoke too quietly. I transitioned to [Faster Whisper](https://github.com/SYSTRAN/faster-whisper), which included a voice activity detection (VAD) filter and added noise reduction.

#### Fish Should Detect the Player's Language

I found that Whisper could detect the player's language. I used this feature to make the fish speak the player's language. However, incorrect language detection sometimes occurred due to the microphone's quality. After trial and error, I decided to ask the player, "English?" or "Nederlands?" during the initial greeting. If the player spoke any other language, the fish would repeat, "English?" or "Nederlands?" to prompt the correct response.

> **Note:** I added "parse language" logic to try to resolve language detection issues. If the player said something, which contained a prefix like "Eng" or "Ned" - the fish then "autocorrect" language used by the player.

#### Fish Should React to Inappropriate Language/Requests

Although the fish was made for the Halloween party, I wanted it to react to unusual requests. OpenAI’s default prompt could be modified to detect and respond to inappropriate language. If the player made an inappropriate request, the fish would respond with something like, _"I'm sorry, but I'm just a fish on the wall. Please don't ask me for life advice,"_ and then ask if they wanted a fish fact or another riddle.

> **Side Effect:** The fish would respond in the language chosen at the start of the conversation, but the player could speak any language since ChatGPT understands multiple languages. This led to funny interactions: when a player asked, _"Why do you keep talking to me in Dutch if I speak English with you?"_ the fish defended its choice by saying, _"You can speak any language with me, but I prefer Dutch because **I believe riddles are better in Dutch**"_ 😂.

# Implementation

## Architecture
- **Fish Body Control**:
  - C with the Arduino framework
  - PlatformIO plugin for VSCode as an IDE
  - API exposed to the Raspberry Pi
- **Raspberry Pi**:
  - **Face Recognition**: OpenCV and the `face_recognition` library.
  - **Voice Capture**: PyAudio.
- **Server**:
  - **Riddle Generation**: ChatGPT 4 mini.
  - **Speech-to-Text and Language Detection**: Faster Whisper.
  - **Text-to-Speech**: AllTalk TTS.

## Fish Proxy: Control of the Fish Body

I traced the signals from the fish IC to its body and configured the Arduino to send the same signals. I used a level shifter for compatibility between the 3.3V Arduino and 5V fish IC.

> **Note:** While testing, I discovered that the signals were actually PWM signals (thanks to the oscilloscope) and not just HIGH/LOW signals. This required rewiring the Arduino multiple times.

I eventually created an I2C peripheral that listens for commands from the Raspberry Pi. I developed a custom protocol for communication.

> **Note:** I tried different communication methods but settled on I2C due to its reliability. Previous versions using the serial port encountered corrupted commands (possibly due to the speaker's magnet) and synchronization issues.

> **Note 2:** Before the Raspberry Pi issues a command, there is "garbage" in the I2C registers. I added a "cleanup" command to stabilize the system.

## Riddle Client: Raspberry Pi

The Raspberry Pi’s responsibilities were defined as follows:

### Face Recognition

- Read the camera stream, detect faces/age groups, assign a random ID, and save it to the filesystem for retrieval.
- For returning players, retrieve the ID based on face similarity.

> **Note:** The age classification feature was meant to create different riddles for kids and adults. However, it struggled to recognize people outside due to jackets and glasses. Increasing the capture resolution might help, but I didn’t have time to test this.

### Voice Capture

- PyAudio captured the player's voice. Pre-recorded WAV files were played for language selection at the start.
- PyAudio recorded voice chunks up to 5 seconds and sent them to the server for processing.

### Fish Body Control

The client used the I2C bus to issue commands to the Arduino.

### Websockets

I used Websockets for communication between the Raspberry Pi and the server for reliable, asynchronous processing.

> **Note:** Refactoring synchronous code to be asynchronous took several evenings. The client part of the fish proxy and face recognition had to run in separate processes to prevent `asyncio` from timing out. I used `async` versions of Queues and Pipes for inter-process communication, and voice capture ran in a separate thread.

> **Note 2:** Some race conditions required additional locks. I’m aware of a potential issue where the system fails if the player speaks too long while the fish is asking another riddle, but this was not critical for the Halloween party.

### Fish Lip Sync

I developed an algorithm that calculates syllables in sentences and distributes them over time for believable lip-sync. The fish's motors are too slow for precise lip movements, so this method provided a realistic effect.

> **Note:** Initially, audio was transferred in the WebSocket payload. Due to payload size limits, I switched to sending a URL for the client to download the audio. This change didn’t impact performance.

### Metadata management

When the Riddle Processor greet the player first time, it will detect language used by the player and then issue `save metadata` command to the Riddle Client. This command will save the player's ID, language, and TTS voices which used by the player. Next interactions with Riddle Processor will include this metadata. Assigning random voices for the Players lead to the _wow_ effect during the Halloween party - as the fish could speak in different voices.

## Riddle Processor: Websocket-Based Server

The server was responsible for:
- Receiving payloads (player voice + ID) from the Riddle Client.
- Managing a "Riddle Registry" to prevent repeated riddles.
- Converting speech-to-text and text-to-speech, assign random voice from the TTS server.
- Sending processed data to the Riddle Client.
- Generating riddles using ChatGPT.
- Tracking player history for context.

> **Note:** I initially added a history cleanup for the most active players, but this was unnecessary for the party as the longest prompt was only 1.5k words.

### Details

The server handled commands over WebSocket, managed riddle processing, and checked the AllTalk TTS server's availability. Whisper models were preloaded for faster processing.

# Conclusion

The fish was a hit at the Halloween party. People were amazed that it could recognize them, speak, and even dance.

Feedback from friends revealed:
- The fish sometimes got stuck in the "greeting" phase due to an overlooked condition.
- It would talk over the player, which was annoying. Further text-to-speech and speech-to-text optimization might help.
- I spent total of $0.25 for the ChatGPT service during development and Halloween party 😀

# Acknowledgements
Special thanks to [my wife](https://www.instagram.com/olnorka/) for being my QA tester. Her help was invaluable. My friend, Mr. V, suggested implementing consistent language use, which I added. Mr. O and Mr. M suggested adding lip-sync, which was challenging but rewarding to implement.

# Demo

Here a short demo of a [Fish Riddle](https://youtu.be/fznJSKNcpw8).

# What's Next?

This project was a valuable learning experience. The knowledge I gained will be useful in future projects or even in my daily work.

Source code is available on [GitHub](https://github.com/PheonixS/FishRiddles).

Thank you for reading, and I wish you a pleasant day. 🦇

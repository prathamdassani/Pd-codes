Project Overview

MoodSync is an AI-powered chatbot that recommends movies or songs based on a user's real-time emotional state, rather than watch history or ratings. The pitch (from your own README) is that it's especially useful for people with alexithymia — i.e., people who struggle to identify/articulate their own emotions — since the bot walks them through simple guided questions instead of asking them to name a mood outright. Team name in the code comments is "Synclub" (JECRC University).

Design & Architecture

Flow: User opens chat → picks movie or song → answers a few guided mood questions (or reacts with emojis/words, or skips) → backend classifies the emotion via Gemini → backend hits OMDb (movies) or Spotify (songs) with emotion-mapped search terms → results saved to MongoDB and shown to the user.

Backend (Express, single entry server.js) — five route modules, mounted cleanly under /api/*:

POST /api/chat — sends the conversation to Gemini and gets back the bot's next guiding question/reply; saves each turn to a Session document.
POST /api/emotion — the real intelligence layer: it takes the collected free text + any emoji/word selections, builds a strict classifier prompt ("respond with ONLY one word from this list..."), and forces the output down to one of 7 emotions (joy, sadness, anger, fear, surprise, disgust, neutral), defaulting to neutral if Gemini returns anything unexpected.
POST /api/movies — maps the detected emotion to a small set of genre-flavored search terms (e.g. anger → "Batman/action/war"), picks one at random, queries OMDb with a randomized year/page for variety, shuffles and returns 4 results.
POST /api/songs — same idea but against Spotify's Client Credentials flow: gets an app token, searches mood-mapped phrases, then filters out lo-fi/remix/karaoke/instrumental noise with a keyword blocklist so you don't get junk results, and requires results be Latin/Devanagari-script only.
GET /api/history — returns the last 3 completed sessions for a device.

Identity without login: there's no user auth. Instead, a deviceId middleware reads (or generates) a device UUID from the x-device-id header on every request, and everything — sessions, history — is scoped to that ID. The frontend generates and persists this UUID in localStorage.

Data model (MongoDB, one Session collection): each session stores deviceId, recommendationType (movie/song), the full messages[] transcript, detectedEmotion, the raw self-expression inputs (selectedEmojis, selectedWords, skipped), the final recommendations[], and timestamps. A static method keepLatestThree prunes older sessions per device so Mongo storage doesn't grow unbounded — a nice, deliberate storage-management touch for interviews.

Frontend (React + Vite, one big component MoodSync_ChatUI.jsx): state machine driven by a phase variable (welcome → question flow → self-expression → results), with dedicated small components for Avatar, TypingBubble, RecCard, SelfExpressionPanel, and HistoryPanel. It talks to the deployed Render backend over plain fetch.

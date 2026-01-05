---
title: AI Resume Analyzer
published: 2024-04-01
description: "How to use this blog template."
image: ./resume.png
tags: ["Fuwari", "Blogging", "Customization"]
category: Project
github: https://github.com/sheet848/admin-dashboard
live: https://admin-dashboard-one-ecru-48.vercel.app/
draft: false
---

# Building an AI Resume Analyzer with Serverless Tech

Job hunting often fails silently—not because of skill gaps, but because resumes don’t align with Applicant Tracking Systems (ATS). To solve this, I built an **AI Resume Analyzer** that compares resumes against job descriptions and provides actionable feedback, ATS scores, and improvement tips.

What makes this project different is the architecture: **no backend code, no infrastructure setup, and zero hosting cost**. Everything runs on the frontend using **Puter.js**, a platform that brings authentication, storage, databases, and AI directly to the browser.

---

## What I Built

The app allows users to:

- Upload PDF resumes via drag-and-drop
- Match resumes against job descriptions
- Generate ATS scores (0–100)
- Receive AI feedback across multiple categories

**Tech Stack**

- React 19 + React Router v7
- TypeScript
- Tailwind CSS v4
- Puter.js (auth, storage, DB, AI)
- Claude Sonnet for analysis

---
## Puter.js

Traditionally, building this app would require OAuth setup, databases, file storage, backend APIs, and deployment—easily 40+ hours before shipping features.

With **one script tag**, Puter.js provided:

- OAuth authentication
- Cloud file storage
- Key-value database
- Free AI models

All without writing backend code. This fundamentally changed how I think about “full-stack” development.

---
## Key Engineering Decisions

### 1. Wrapping Puter.js with Zustand

To integrate Puter cleanly with React, I built a Zustand-based wrapper that:

- Centralized auth, storage, and AI access
- Eliminated prop drilling
- Managed loading and auth state cleanly
- Improved type safety and testability

This reinforced when and _why_ wrapping third-party libraries is worth the effort.

---

### 2. Reliable AI Output with Structured Prompts

Raw AI responses are unreadable for code. I enforced **strict JSON-only responses**, defined exact schemas, and added defensive parsing.  
**Lesson**: AI works best when you treat it like an unreliable API—validate everything.

---

### 3. PDF Handling with PDF.js + Canvas

PDFs were converted to images client-side for previews and processing.  
This improved UX and kept file sizes manageable, while deepening my understanding of Canvas and async flows.

---

### 4. React Router v7 Data Loaders

Moving data fetching into route loaders removed loading boilerplate, simplified state management, and improved error handling—cutting nearly a third of my component state code.

---

### 5. UX Polish Matters

AI analysis takes time, so I added:

- Step-by-step status updates
- File validation and previews
- Clear error states

Users don’t mind waiting if they know _what’s happening_.

---

## Key Takeaways

- Serverless frontend platforms can eliminate entire backend layers
- Zustand is a powerful, simpler alternative to Redux for many apps
- Structured AI prompting is non-negotiable for production use
- TypeScript pays off massively in medium-to-large projects
- Polish and UX details separate “working” from “professional”

---

## Closing Thoughts

This project reshaped how I approach modern web development. The backend isn’t always necessary, AI is a multiplier—not magic—and the best portfolio projects solve real problems you personally care about.

The future of web development is already here: **frontend-first, serverless, and AI-powered**.

---
## Resources

- **Puter.js Docs**: https://docs.puter.com
- **React Router v7**: https://reactrouter.com
- **Tailwind CSS**: https://tailwindcss.com

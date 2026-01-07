---
title: AI Resume Analyzer
published: 2025-07-22
description: Serverless AI tool providing ATS scores and resume feedback
image: ./resume.png
category: Projects
github: https://github.com/sheet848/ai-resume-analyzer
live: https://ai-resume-analyzer-she12.vercel.app/
draft: false
---

# Building an AI Resume Analyzer with Serverless Technology

## Introduction

Job hunting often fails silently—not because of skill gaps, but because resumes don’t align with Applicant Tracking Systems (ATS). To solve this, I built an **AI Resume Analyzer** that compares resumes against job descriptions and provides actionable feedback, ATS scores, and improvement tips.

What makes this project different is the architecture: **no backend code, no infrastructure setup, and zero hosting cost**. Everything runs on the frontend using **Puter.js**, a platform that brings authentication, storage, databases, and AI directly to the browser.

**Project credit:** Based on the work and teaching of [@Adrian Hajdin / JavaScript Mastery](https://www.youtube.com/@javascriptmastery) on YouTube.

<div class="link-card block btn-regular px-6 py-4 rounded-2xl" style="display: block">
  <p class="font-bold text-2xl">Discover this Project</p>
  <p><strong>GitHub Link:</strong> <a href="https://github.com/sheet848/ai-resume-analyzer" target="_blank">https://github.com/sheet848/ai-resume-analyzer</a></p>
  <p><strong>See Live:</strong> <a href="https://ai-resume-analyzer-she12.vercel.app/" target="_blank">https://ai-resume-analyzer-she12.vercel.app/</a></p>
</div>

## Project Overview: What Did I Build?

The AI Resume Analyzer is a full-stack application that:

- Accepts PDF resume uploads with drag-and-drop
- Matches resumes against job descriptions
- Generates comprehensive ATS scores (0-100)
- Provides AI feedback across 4 categories
- Stores all data securely in the cloud

**Tech Stack:**

- **React 19** with **React Router v7**
- **TypeScript** for type safety
- **Tailwind CSS v4** for styling
- **Puter.js** for backend services
- **Claude Sonnet** for AI analysis

## Discovering Puter.js

### The Traditional Backend Problem

Before discovering Puter.js, building an app like this meant; setting up authentication, database setup, file storage, AI integration, backend API. That's easily 40-60 hours of work **before** writing any frontend code.

### The Puter.js Revolution

With Puter.js, I added ONE script tag:

```html
<script src="https://js.puter.com/v2/"></script>
```

And suddenly I had access to:

- OAuth authentication (no configuration)
- Cloud file storage (unlimited)
- Key-value database (instant)
- Multiple AI models (free)
- All without a single line of backend code

This wasn't just convenient—it fundamentally changed my development approach.

## Building the Puter Wrapper: A Master Class in State Management

### The Challenge

While Puter.js works perfectly with vanilla JavaScript, integrating it into React required careful consideration. Questions arose:

- How do I ensure Puter loads before my components try to use it?
- How do I share authentication state across components?
- How do I avoid prop drilling for Puter functions?
- How do I handle loading states elegantly?

### The Solution: Zustand + Puter

I created a custom wrapper using Zustand that became the heart of the application:

```typescript
// lib/puter.ts
export const usePuterStore = create<PuterStore>((set, get) => ({
  isLoading: true,
  auth: { isAuthenticated: false, user: null },
  
  init: async () => {
    const puter = getPuter();
    if (puter) {
      await get().checkAuthStatus();
      set({ isLoading: false });
    }
  },
  
  signIn: async () => {
    const puter = getPuter();
    await puter.auth.signIn();
  },
  
  fs: {
    upload: async (files) => {
      const puter = getPuter();
      return await puter.fs.upload(files);
    }
  }
  // ... more functions
}));
```

### The Learning Curve

Creating this wrapper taught me:

**When to wrap external libraries**: Not every library needs wrapping. But when you need:

- Shared state across components
- Loading state management
- Type safety improvements
- Consistent error handling

A wrapper like this is invaluable.

**Zustand vs Redux**: Zustand is a simpler alternatives. No actions, no reducers—just functions that update state.

## PDF Processing: The Technical Deep Dive

### The Challenge

Displaying PDFs on the web is harder than it sounds. I needed to:

1. Accept PDF uploads
2. Display thumbnails on the homepage
3. Allow full PDF viewing
4. Keep file sizes reasonable

### Solution: PDF.js + Canvas API

I built a custom conversion function:

```typescript
const convertPDFToImage = async (file: File): Promise<File> => {
  // Load PDF
  const arrayBuffer = await file.arrayBuffer();
  const pdf = await pdfjsLib.getDocument(arrayBuffer).promise;
  
  // Get first page
  const page = await pdf.getPage(1);
  
  // Render to canvas
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  
  const viewport = page.getViewport({ scale: 2 });
  canvas.width = viewport.width;
  canvas.height = viewport.height;
  
  await page.render({ canvasContext: context, viewport }).promise;
  
  // Convert to PNG
  return new Promise((resolve) => {
    canvas.toBlob((blob) => {
      const imageFile = new File([blob], 'resume.png', { 
        type: 'image/png' 
      });
      resolve(imageFile);
    }, 'image/png');
  });
};
```

**What I Learned:**

**Canvas API is powerful**: Pixel manipulation. Render anything, convert to image, done.

**PDF.js handles complexity**: Loading multi-page PDFs, different formats, compression; all handled by the library.

**Promises can nest**: This function uses Promises within Promises. Understanding async flow is crucial.

## React Router v7: The Data Loading Revolution

### What's New?

React Router v7 introduced a game-changer: route-level data loading.

**Before (Manual Loading):**

```typescript
function ResumePage() {
  const [resume, setResume] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    loadResume().then(setResume).finally(() => setLoading(false));
  }, [id]);
  
  if (loading) return <Spinner />;
  return <ResumeView resume={resume} />;
}
```

**After (Router Loading):**

```typescript
// Just define the loader
export async function loader({ params }) {
  return await loadResume(params.id);
}

// Component receives data as props
function ResumePage({ data }) {
  return <ResumeView resume={data} />;
}
```

**Benefits:**

1. **No loading state boilerplate**: Router handles it
2. **Parallel data fetching**: Multiple loaders run simultaneously
3. **Error boundaries**: Built-in error handling
4. **Type safety**: TypeScript knows the data shape

This pattern eliminated about 30% of my state management code.

## Working with TypeScript

Coming from JavaScript, TypeScript felt like unnecessary overhead:

- Writing interfaces for everything
- Dealing with type errors
- More code for the same functionality

### The Conversion Experience

This project changed my mind completely. Here's why:

**Caught bugs before runtime:**

```typescript
// TypeScript caught this immediately
const score = feedback.overalScore; // Typo!
// Property 'overalScore' does not exist on type 'Feedback'
// Did you mean 'overallScore'?
```

**Self-documenting code:**

```typescript
interface Feedback {
  overallScore: number;
  ats: ATSFeedback;
  toneAndStyle: CategoryFeedback;
}

// I know exactly what this function expects and returns
function analyzeResume(resume: File): Promise<Feedback>
```

**Refactoring confidence**: When I restructured the Feedback interface, TypeScript showed me every place that needed updating. No hunting through files.

**Editor intelligence**: My IDE knew what properties existed, what functions were available, and what types were expected.

**Key Insight**: TypeScript is about writing better code and not more code.

## Key Takeaways

- Serverless frontend platforms can eliminate entire backend layers
- Zustand is a powerful, simpler alternative to Redux for many apps
- Structured AI prompting is non-negotiable for production use
- TypeScript pays off massively in medium-to-large projects
- Polish and UX details separate “working” from “professional”

## Conclusion

This project reshaped how I approach modern web development. The backend isn’t always necessary, AI is a multiplier and the best portfolio projects solve real problems you personally care about.

The future of web development is already here: **frontend-first, serverless, and AI-powered**.

---
## Resources

- **Puter.js Docs**: https://docs.puter.com
- **React Router v7**: https://reactrouter.com
- **Tailwind CSS**: https://tailwindcss.com

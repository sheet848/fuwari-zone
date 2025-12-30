---
title: AI Resume Analyzer
published: 2025-07-01
tags: [Markdown, Blogging, Demo]
category: Project
draft: false
---

# Building an AI Resume Analyzer with Serverless Technology: A Journey into the Future of Web Development

## Introduction

Job hunting is brutal. You send out resume after resume, and most of the time? Silence. No replies, no rejections—just crickets. The problem isn't always your skills; it's that your resume isn't speaking the language that Applicant Tracking Systems (ATS) understand.

I recently built an AI Resume Analyzer that solves this exact problem. It analyzes your resume against specific job descriptions and provides detailed, actionable feedback—all powered by free AI models and serverless technology.

But here's what makes this project truly special: **zero backend code, zero infrastructure costs, and zero configuration**. How? Puter.js—a revolutionary platform that brings serverless OAuth, cloud storage, databases, and AI directly to your frontend.

In this post, I'll walk you through the entire journey: the challenges I faced, the innovative solutions I discovered, and the valuable lessons that changed how I think about web development.

## Project Overview: What Did I Build?

The AI Resume Analyzer is a full-stack application that:

- Accepts PDF resume uploads with drag-and-drop
- Matches resumes against job descriptions
- Generates comprehensive ATS scores (0-100)
- Provides AI feedback across 4 categories
- Stores all data securely in the cloud
- Works entirely for free (user-pays model)

**Tech Stack:**

- React 19 with React Router v7
- TypeScript for type safety
- Tailwind CSS v4 for styling
- Puter.js for backend services
- Claude Sonnet for AI analysis

## The "Aha!" Moment: Discovering Puter.js

### The Traditional Backend Problem

Before discovering Puter.js, building an app like this meant:

1. **Setting up authentication**: Configure OAuth providers, manage sessions, handle tokens
2. **Database setup**: Choose between SQL/NoSQL, configure connections, write schemas
3. **File storage**: Set up AWS S3 or similar, manage buckets, handle uploads
4. **AI integration**: Get API keys, manage rate limits, handle billing
5. **Backend API**: Write routes, middleware, error handling
6. **Deployment**: Configure servers, manage environment variables, monitor costs

That's easily 40-60 hours of work **before** writing any frontend code.

### The Puter.js Revolution

With Puter.js, I added ONE script tag:

```html
<script src="https://js.puter.com/v2/"></script>
```

And suddenly I had access to:

- ✅ OAuth authentication (no configuration)
- ✅ Cloud file storage (unlimited)
- ✅ Key-value database (instant)
- ✅ Multiple AI models (free)
- ✅ All without a single line of backend code

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

**Why This Approach Is Brilliant:**

1. **Single Source of Truth**: All Puter functionality accessible from one hook
2. **Type Safety**: TypeScript knows exactly what's available
3. **Loading States**: Built-in loading management
4. **No Prop Drilling**: Access anywhere with `usePuterStore()`
5. **Testable**: Can mock the entire Puter API in tests

### The Learning Curve

Creating this wrapper taught me:

**When to wrap external libraries**: Not every library needs wrapping. But when you need:

- Shared state across components
- Loading state management
- Type safety improvements
- Consistent error handling

A wrapper like this is invaluable.

**Zustand vs Redux**: Before this project, I defaulted to Redux for state management. Zustand opened my eyes to simpler alternatives. No actions, no reducers—just functions that update state.

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

**Canvas API is powerful**: Before this, Canvas seemed intimidating. But it's just pixel manipulation—render anything, convert to image, done.

**PDF.js handles complexity**: Loading multi-page PDFs, different formats, compression—all handled by the library.

**Promises can nest**: This function uses Promises within Promises. Understanding async flow is crucial.

## AI Integration: Structured Output is King

### The Problem with Raw AI

Initially, I just sent the resume and job description to Claude:

```typescript
const feedback = await puter.ai.chat('claude-sonnet-3-5', [
  { role: 'user', content: `Analyze this resume...` }
]);
```

The response? A beautiful, human-readable paragraph. Useless for code.

### The Solution: Structured Prompting

I created a detailed format specification:

```typescript
const AI_RESPONSE_FORMAT = `
Return ONLY valid JSON with this exact structure:
{
  "overallScore": number (0-100),
  "ats": {
    "score": number,
    "tips": string[]
  },
  "toneAndStyle": {
    "score": number,
    "whatWentWell": string[],
    "whatToImprove": string[],
    "tips": string[]
  }
  // ... more categories
}

Do NOT include any text outside this JSON.
No markdown code blocks, no explanations.
`;
```

Then I could reliably parse the response:

```typescript
const feedbackText = feedbackMessage.content;
const feedback = JSON.parse(feedbackText);
```

**Key Lessons:**

**Be extremely specific**: AI will do exactly what you ask—no more, no less. Vague prompts get vague responses.

**Request JSON explicitly**: Saying "return JSON format" isn't enough. Show the exact structure.

**Handle parsing errors**: Even with perfect prompts, AI occasionally adds extra text. Always try-catch JSON parsing.

**Test with multiple models**: What works for Claude might fail for GPT-4. Test your prompts across models.

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

## File Upload UX: The Devil in the Details

### The Simple Approach

Initially, I used a basic file input:

```typescript
<input type="file" accept=".pdf" onChange={handleUpload} />
```

Functional? Yes. User-friendly? Absolutely not.

### The Polished Approach: React Dropzone

I implemented react-dropzone with extensive customization:

```typescript
const { getRootProps, getInputProps, isDragActive } = useDropzone({
  onDrop,
  accept: { 'application/pdf': ['.pdf'] },
  maxSize: 20 * 1024 * 1024, // 20MB
  multiple: false
});
```

**UX Enhancements I Added:**

1. **Visual feedback**: Different states for hover, drag, error
2. **File preview**: Show PDF thumbnail immediately
3. **Size validation**: Clear error for oversized files
4. **Format restriction**: Only accept PDFs
5. **One-click removal**: Easy to change files

**Learning:**

Small UX touches matter enormously. Users judge applications by these details. The difference between "works" and "delightful" is in the polish.

## TypeScript: From Reluctance to Advocacy

### My Initial Resistance

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

**Key Insight**: TypeScript isn't about writing more code—it's about writing better code with confidence.

## Tailwind CSS v4: Utility-First Revelation

### The Mental Shift

Before Tailwind, my CSS workflow:

1. Write HTML structure
2. Create CSS file
3. Name classes (the hardest part)
4. Jump between files constantly
5. Deal with specificity issues

With Tailwind:

```tsx
<button className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors">
  Analyze Resume
</button>
```

Everything in one place. No naming. No specificity wars.

### Custom Design System

I extended Tailwind with custom utilities:

```css
/* app.css */
.primary-button {
  @apply bg-gradient-to-r from-blue-500 to-purple-500 
         text-white px-6 py-3 rounded-full 
         hover:scale-105 transition-transform;
}
```

This gave me:

- Consistency across components
- One place to update styles
- Reusable patterns
- Cleaner JSX

### Animations Made Easy

Tailwind v4 + tailwindcss-animate made animations trivial:

```tsx
<div className="animate-in fade-in duration-1000">
  Content fades in beautifully
</div>
```

No CSS keyframes, no complex configurations.

## The Wipe Feature: Development Tools Matter

### The Problem

During development, I created dozens of test resumes. The database got cluttered. I needed a way to reset everything.

### The Solution

I created a hidden `/wipe` route:

```typescript
// routes/wipe.tsx
async function wipeAppData() {
  // Delete all files
  const files = await puter.fs.list();
  for (const file of files) {
    await puter.fs.delete(file.path);
  }
  
  // Clear KV store
  const keys = await puter.kv.list('resume:*');
  for (const key of keys) {
    await puter.kv.delete(key);
  }
}
```

**Why This Matters:**

Development isn't just about features—it's about workflow. Small tools like this save hours of manual data cleanup.

I learned to build developer conveniences as I go, not retrofit them later.

## Deployment: Puter App Store Experience

### Traditional Deployment Pain

Normally, deployment means:

- Configure environment variables
- Set up CI/CD
- Configure domain and SSL
- Manage server costs
- Monitor performance

### Puter Deployment

1. Set `ssr: false` in React Router config
2. Run `npm run build`
3. Drag contents of `build/client/` to Puter
4. Click "Deploy"

**Done.** Live in 60 seconds.

### The User-Pays Model

Here's what blew my mind: **I pay nothing for hosting**.

Each user of my app uses their own Puter quota:

- 10GB storage per user
- Free AI usage per user
- No bandwidth costs

If 1,000 people use my app, my costs remain **zero**. Users cover their own usage.

This is revolutionary for indie developers and students.

## Key Challenges and Solutions

### Challenge 1: Authentication State Management

**Problem**: Puter authentication happens on a separate page. How do I know when a user returns?

**Solution**:

- Store intended destination in URL query params
- Check auth status on app init
- Redirect back to original page after login

```typescript
// Redirect to auth with next parameter
navigate(`/auth?next=/upload`);

// After auth, redirect to next page
useEffect(() => {
  if (isAuthenticated) {
    const next = new URLSearchParams(location.search).get('next');
    navigate(next || '/');
  }
}, [isAuthenticated]);
```

### Challenge 2: AI Response Inconsistency

**Problem**: Sometimes Claude returned JSON with markdown code blocks, sometimes without.

**Solution**: Aggressive parsing with fallbacks:

```typescript
let feedbackText = response.content;

// Remove markdown code blocks if present
feedbackText = feedbackText.replace(/```json\n?/g, '');
feedbackText = feedbackText.replace(/```\n?/g, '');

// Parse cleaned text
try {
  const feedback = JSON.parse(feedbackText);
} catch (error) {
  // Fallback: extract JSON from mixed content
  const jsonMatch = feedbackText.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    feedback = JSON.parse(jsonMatch[0]);
  }
}
```

### Challenge 3: File Size Management

**Problem**: Some users uploaded multi-page, high-resolution PDFs (50MB+).

**Solution**: Client-side optimization:

```typescript
const optimizePDF = async (file: File): Promise<File> => {
  if (file.size > 20 * 1024 * 1024) {
    throw new Error('File size exceeds 20MB limit');
  }
  // Additional: Could compress PDFs using pdf-lib
  return file;
};
```

Future enhancement: Automatic compression for large files.

### Challenge 4: Loading State UX

**Problem**: AI analysis takes 15-30 seconds. Users got impatient.

**Solution**: Detailed status updates:

```typescript
const [statusText, setStatusText] = useState('');

// Update status throughout process
setStatusText('Uploading resume...');
await uploadFile();

setStatusText('Converting to image...');
await convertPDF();

setStatusText('Analyzing with AI...');
await analyzeResume();

setStatusText('Analysis complete! Redirecting...');
```

Users appreciate knowing what's happening.

## What I'd Do Differently

### 1. Progressive Resume Building

Current approach: Upload complete resume, get feedback.

Better approach: Interactive builder that provides real-time suggestions as users create their resume.

### 2. Resume Templates

Provide ATS-optimized templates users can fill out, ensuring better scores from the start.

### 3. Comparison View

Allow users to upload multiple versions and compare scores side-by-side.

### 4. Skills Gap Analysis

Show specific skills mentioned in job descriptions that aren't in the resume, with learning resources.

### 5. Historical Tracking

Chart score improvements over time as users iterate on their resumes.

## Skills Gained

This project transformed my understanding of:

### 1. Serverless Architecture

- When to use serverless vs traditional backends
- The user-pays model advantages
- Serverless limitations and workarounds

### 2. AI Integration

- Crafting effective prompts for structured output
- Handling AI response variability
- Cost management with AI APIs

### 3. State Management

- Zustand for lightweight state
- When to use global vs local state
- Custom hooks for reusability

### 4. Type Safety

- TypeScript for large applications
- Type declarations for external libraries
- Generic types and utility types

### 5. Modern React

- React Router v7 data loading
- Concurrent features
- Suspense boundaries

### 6. Developer Experience

- Building with AI assistants (Juny)
- Creating dev tools alongside features
- Documentation-driven development

## The Bigger Picture: Lessons for Your Career

### 1. Embrace New Platforms Early

I discovered Puter.js when it was relatively new. Being an early adopter means:

- Less competition in the ecosystem
- Better community support opportunities
- Resume differentiation
- Real-world experience with cutting-edge tech

### 2. Build What You Need

This project solved my own problem: optimizing resumes for job applications. The best portfolio projects solve real problems.

### 3. AI is a Tool, Not Magic

AI didn't write this project for me. It helped with:

- Boilerplate code
- Utility functions
- Format conversions
- Initial layouts

But the architecture, logic, and problem-solving? That was all me.

### 4. Polish Matters

The difference between a good portfolio project and a great one is polish:

- Smooth animations
- Clear error messages
- Loading states
- Responsive design

These details show professionalism.

## Performance Metrics

After deployment, here are the numbers:

- **Bundle size**: 487KB (gzipped)
- **Initial load**: < 2s on 4G
- **AI analysis**: 15-30s average
- **Storage used**: ~5MB per resume
- **Cost per user**: $0 (user-pays model)

## Conclusion

Building this AI Resume Analyzer taught me that modern web development is undergoing a massive shift. The backend is becoming optional for many applications. With platforms like Puter.js, the barrier to building full-stack applications has never been lower.

But more importantly, this project taught me:

- **Start with the problem**: The best projects solve real needs
- **Embrace new technologies**: Don't wait for the "perfect" tool
- **Polish matters**: Small UX details separate good from great
- **AI is a multiplier**: Use it to accelerate, not replace, your skills
- **Build in public**: Share your work and learning process

If you're looking to build your next project, consider:

1. What problem do you face regularly?
2. Could a serverless approach simplify development?
3. How can AI enhance the user experience?
4. What will make this portfolio piece stand out?

The future of web development is here. It's serverless, AI-powered, and more accessible than ever.

---

## Resources

- **Live Demo**: [Your Puter App URL]
- **Source Code**: [Your GitHub Repo]
- **Puter.js Docs**: https://docs.puter.com
- **React Router v7**: https://reactrouter.com
- **Tailwind CSS**: https://tailwindcss.com

---

_Have you built with serverless platforms? What challenges did you face? Share your experiences in the comments!_

---
title: Video Annotation Tool
published: 2024-04-01
description: How to use this blog template.
image: ./video.png
tags: ["Fuwari", "Blogging", "Customization"]
category: Project
github: "https://github.com/sheet848/admin-dashboard"
live: "https://admin-dashboard-one-ecru-48.vercel.app/"
draft: false
---
# Building a Video Annotation Tool: From YouTube API to Redux State Management

## Introduction

Ever tried to take notes while watching a video tutorial? You pause, write the timestamp, type your note, resume playback—it's tedious. I wanted a better solution, so I built a video annotation tool that automatically captures timestamps and lets you jump to any moment with a single click.

In this post, I'll walk you through building a production-ready video annotation application using Vite, React, and Redux. More importantly, I'll share the architectural decisions, the challenges I faced, and why certain patterns made all the difference.

## The Vision

**What I wanted to build:**

- Annotate both YouTube videos and uploaded files
- Automatic timestamp capture
- Click-to-seek functionality
- Real-time synchronization between video and notes
- Clean, intuitive interface
- Exportable annotations

**Why this project matters:** Video annotation is used in education, content creation, research, and accessibility. Understanding how to build it teaches you state management, external API integration, file handling, and real-time UI updates—skills that transfer to countless other applications.

## Tech Stack: Why These Choices?

### Vite Over Create React App

I chose Vite because:

- **Instant dev server start** (< 500ms vs 10+ seconds)
- **Lightning-fast HMR** (changes reflect instantly)
- **Optimized production builds** (better tree-shaking)
- **Modern by default** (ES modules, native TypeScript)

Create React App felt like using dial-up internet after experiencing Vite's speed.

### Redux Toolkit Over useState

With annotations, video time, playback state, and multiple video sources, state management could spiral out of control. Redux Toolkit provided:

**Predictable State Flow:**

```javascript
User clicks "Add Annotation"
  ↓
Dispatch action
  ↓
Reducer updates state
  ↓
Components re-render
  ↓
UI updates
```

**Time Travel Debugging:** Redux DevTools let me scrub through state changes. When annotations weren't syncing correctly, I could see exactly when and why state updated.

**Single Source of Truth:** Video time, annotations, and player state all live in one store. No prop drilling. No state inconsistencies.

### YouTube IFrame API Integration

The YouTube IFrame API was both powerful and frustrating. Here's what I learned:

**The Good:**

- Programmatic playback control
- Event listeners for player state
- Quality and speed control
- Thumbnail generation

**The Challenging:**

- Asynchronous initialization
- Cross-origin restrictions
- API loading race conditions
- Documentation gaps

I'll show you how I solved these problems.

## Architecture: The Foundation

### Redux Slice Design

I structured state around the annotation workflow:

```javascript
const initialState = {
  // Video source management
  videoSource: {
    type: null,        // 'youtube' | 'upload'
    url: null,         // YouTube URL
    file: null         // Uploaded file
  },
  
  // Annotation data
  annotations: [],     // Array of { id, timestamp, text }
  
  // Playback state
  currentTime: 0,      // Current video position
  isPlaying: false,    // Is video playing?
  
  // UI state
  selectedAnnotation: null,  // Currently selected annotation
  isEditMode: false,         // Is user editing?
  playerReady: false         // Is player initialized?
};
```

**Why this structure?**

1. **Separation of concerns**: Video source separate from annotations separate from playback
2. **Extensibility**: Easy to add new video sources (Vimeo, local streaming)
3. **UI state isolation**: Edit mode doesn't affect video playback

### Component Hierarchy

```
App
├── Header
├── VideoSourceSelector
│   ├── YouTube URL Input
│   └── File Upload Button
├── VideoPlayer
│   ├── YouTube IFrame
│   └── HTML5 Video
├── AnnotationsPanel
│   ├── Annotation List
│   │   └── Annotation Item
│   │       ├── Timestamp
│   │       ├── Text
│   │       └── Actions (Edit/Delete)
│   └── Add Button
└── AddAnnotationModal
    ├── Timestamp Display
    ├── Text Input
    └── Save/Cancel
```

Each component has a single responsibility. Want to change how annotations render? Edit `AnnotationsPanel`. Want to add Vimeo support? Modify `VideoPlayer`. Clean, maintainable architecture.

## YouTube API Integration: The Details

### Loading the API

The YouTube IFrame API must be loaded before use. I created a utility function:

```javascript
const loadYouTubeAPI = () => {
  return new Promise((resolve) => {
    if (window.YT && window.YT.Player) {
      resolve();
      return;
    }

    window.onYouTubeIframeAPIReady = () => {
      resolve();
    };

    const tag = document.createElement('script');
    tag.src = 'https://www.youtube.com/iframe_api';
    document.head.appendChild(tag);
  });
};
```

**Why a Promise?**

The API loads asynchronously. Trying to create a player before it's ready causes cryptic errors. Wrapping in a Promise lets us `await` readiness.

### Creating the Player

```javascript
useEffect(() => {
  const initPlayer = async () => {
    await loadYouTubeAPI();
    
    const player = new window.YT.Player('youtube-player', {
      videoId: extractVideoId(videoUrl),
      events: {
        onReady: handlePlayerReady,
        onStateChange: handleStateChange
      }
    });
    
    playerRef.current = player;
  };
  
  if (videoSource.type === 'youtube' && videoSource.url) {
    initPlayer();
  }
}, [videoSource]);
```

**Critical Details:**

1. **playerRef**: Stored in a ref, not state. Player instance doesn't need to trigger re-renders.
2. **Conditional initialization**: Only create player when YouTube URL exists
3. **Event handlers**: Critical for time synchronization

### Time Synchronization

The hardest part was keeping Redux state in sync with video time:

```javascript
useEffect(() => {
  if (!isPlaying) return;
  
  const interval = setInterval(() => {
    if (playerRef.current?.getCurrentTime) {
      const time = playerRef.current.getCurrentTime();
      dispatch(updateCurrentTime(time));
    }
  }, 100);
  
  return () => clearInterval(interval);
}, [isPlaying]);
```

**Why 100ms intervals?**

- Frequent enough for smooth UI updates
- Not so frequent it causes performance issues
- Matches standard video frame rates

I tried 16ms (60fps) initially. It caused Redux to choke on update volume. 100ms was the sweet spot.

## File Upload: Local Video Support

### Handling File Input

```javascript
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  
  if (!file.type.startsWith('video/')) {
    alert('Please upload a valid video file');
    return;
  }
  
  dispatch(setVideoSource({
    type: 'upload',
    file: file
  }));
};
```

### Creating Object URLs

To display uploaded videos, I created object URLs:

```javascript
useEffect(() => {
  if (videoSource.type !== 'upload') return;
  
  const url = URL.createObjectURL(videoSource.file);
  videoRef.current.src = url;
  
  return () => URL.revokeObjectURL(url);
}, [videoSource]);
```

**Memory Management:**

Object URLs create memory leaks if not revoked. The cleanup function in `useEffect` prevents this.

### HTML5 Video API

For uploaded files, I used the standard HTML5 video element:

```javascript
<video
  ref={videoRef}
  controls
  onTimeUpdate={(e) => dispatch(updateCurrentTime(e.target.currentTime))}
  onPlay={() => dispatch(setPlaying(true))}
  onPause={() => dispatch(setPlaying(false))}
/>
```

**Why not use a library?**

Native HTML5 video works great for basic playback. No need for video.js or similar libraries unless you need advanced features.

## Annotation Management: CRUD Operations

### Creating Annotations

```javascript
const handleAddAnnotation = () => {
  const annotation = {
    id: crypto.randomUUID(),
    timestamp: currentTime,
    text: annotationText,
    createdAt: Date.now()
  };
  
  dispatch(addAnnotation(annotation));
  dispatch(closeModal());
};
```

**Automatic timestamp capture** was crucial. When users click "Add," the current video time is automatically saved. No manual input needed.

### Editing Annotations

```javascript
const handleEditAnnotation = (id, newText) => {
  dispatch(updateAnnotation({
    id,
    text: newText
  }));
};
```

**Design decision:** Timestamps are immutable. You can edit text but not the timestamp. Why? Changing timestamps would break the annotation's context in the video.

### Deleting Annotations

```javascript
const handleDeleteAnnotation = (id) => {
  if (confirm('Delete this annotation?')) {
    dispatch(deleteAnnotation(id));
  }
};
```

Simple confirm dialog prevents accidental deletions.

### Click-to-Seek

The most satisfying feature to implement:

```javascript
const handleAnnotationClick = (timestamp) => {
  if (playerRef.current?.seekTo) {
    playerRef.current.seekTo(timestamp);
  } else if (videoRef.current) {
    videoRef.current.currentTime = timestamp;
  }
  
  dispatch(setSelectedAnnotation(id));
};
```

**Unified interface:** Same click handler works for both YouTube and uploaded videos. Different APIs, same user experience.

## Real-Time UI Updates

### Highlighting Active Annotation

As the video plays, the current annotation highlights:

```javascript
const isActive = (annotation) => {
  return currentTime >= annotation.timestamp &&
         (nextAnnotation ? currentTime < nextAnnotation.timestamp : true);
};

return (
  <div className={cn(
    'annotation',
    isActive(annotation) && 'active'
  )}>
    {/* Annotation content */}
  </div>
);
```

**Visual feedback** is critical. Users need to see which annotation corresponds to what they're currently watching.

### Smooth Scrolling

When an annotation becomes active, scroll it into view:

```javascript
useEffect(() => {
  const activeElement = document.querySelector('.annotation.active');
  if (activeElement) {
    activeElement.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }
}, [currentTime]);
```

Prevents users from losing their place in long annotation lists.

## Copy to Clipboard Feature

Users wanted to export annotations. I added a "Copy All" button:

```javascript
const copyToClipboard = () => {
  const text = annotations
    .map(a => `[${formatTimestamp(a.timestamp)}] ${a.text}`)
    .join('\n');
  
  navigator.clipboard.writeText(text);
  
  toast.success('Copied to clipboard!');
};

const formatTimestamp = (seconds) => {
  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  const s = Math.floor(seconds % 60);
  
  return `${h}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
};
```

**Format:** `[0:01:23] This is my annotation text`

Clean, readable, easy to paste into notes.

## Challenges and Solutions

### Challenge 1: YouTube API Race Conditions

**Problem:** Sometimes player would initialize before API loaded, causing crashes.

**Solution:** Promise-based API loading with await before player creation.

### Challenge 2: State Update Frequency

**Problem:** Updating Redux every 16ms (60fps) caused performance issues.

**Solution:** Reduced to 100ms intervals. Still smooth, no performance hit.

### Challenge 3: Memory Leaks with Object URLs

**Problem:** Creating object URLs for uploaded videos without cleanup caused memory leaks.

**Solution:** `useEffect` cleanup function to revoke URLs:

```javascript
return () => URL.revokeObjectURL(url);
```

### Challenge 4: Cross-Origin Video Issues

**Problem:** Some uploaded videos had CORS issues.

**Solution:** All video processing happens client-side. No server uploads needed.

### Challenge 5: Modal State Management

**Problem:** Modal open/close state tied to annotation data caused bugs.

**Solution:** Separated UI state (isModalOpen) from data state (annotations).

## Performance Optimization

### Memoization

Annotation list re-rendered on every time update. Fixed with `React.memo`:

```javascript
const AnnotationItem = React.memo(({ annotation, isActive, onClick }) => {
  return (
    <div className={isActive ? 'active' : ''} onClick={onClick}>
      {annotation.text}
    </div>
  );
}, (prevProps, nextProps) => {
  return prevProps.isActive === nextProps.isActive &&
         prevProps.annotation.id === nextProps.annotation.id;
});
```

Reduced re-renders by 90%.

### Redux Selector Optimization

```javascript
const selectSortedAnnotations = createSelector(
  state => state.annotations,
  annotations => [...annotations].sort((a, b) => a.timestamp - b.timestamp)
);
```

Sorting only happens when annotations change, not on every render.

## What I'd Do Differently

### 1. Add TypeScript

JavaScript was fine for prototyping, but TypeScript would've caught several bugs earlier. Type-safe Redux actions would've been especially helpful.

### 2. Implement Undo/Redo

Redux makes this trivial. Should've added it from the start.

### 3. Add Annotation Categories

Let users tag annotations (question, important, note). Makes searching easier.

### 4. Persist to Local Storage

Annotations disappear on page refresh. Should've added localStorage persistence or a backend.

### 5. Better Mobile Support

Works on mobile, but controls are cramped. Needs responsive design improvements.

## Key Learnings

1. **Redux Isn't Overkill**: For complex state, Redux simplifies everything. The boilerplate pays off.
    
2. **External APIs Need Error Handling**: YouTube API can fail. Always have fallbacks.
    
3. **Performance Matters**: 100ms timer updates vs 16ms makes huge difference.
    
4. **User Feedback Is Essential**: Loading states, success messages, error handling—users need to know what's happening.
    
5. **Vite Is Fast**: Development speed improved 10x compared to Create React App.
    

## Conclusion

Building this video annotation tool taught me about:

- Complex state management with Redux
- External API integration (YouTube)
- File handling and object URLs
- Real-time UI synchronization
- Performance optimization
- Clean component architecture

The most valuable lesson? **Start with architecture.** I spent time designing the Redux store structure upfront. That investment paid off as features grew. Adding new capabilities never required refactoring the core.

If you're building anything with video, time-based data, or complex state, these patterns will serve you well.

---

## Try It Yourself

**Live Demo:** https://video-annotate.vercel.app **Source Code:** https://github.com/sheet848/video-annotate

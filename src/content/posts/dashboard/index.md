---
title: React Admin Dashboard
published: 2024-04-01
description: How to use this blog template.
image: ./dasboard.png
tags: ["Fuwari", "Blogging", "Customization"]
category: Project
github: https://github.com/sheet848/admin-dashboard
live: https://admin-dashboard-one-ecru-48.vercel.app/
draft: false
---

## Introduction

Recently, I completed a comprehensive React admin dashboard project that transformed my understanding of building enterprise-level applications. 

This wasn't just another tutorial project—it was an intensive dive into production-grade packages, architectural patterns, and best practices used by professional development teams.

In this post, I'll share my journey building this dashboard from scratch, the challenges I encountered, the solutions I discovered, and the valuable lessons that will shape my future projects.

👉 **Project credit:** Based on the work and teaching of [@EdRoh](#) on YouTube.

<div class="link-card block btn-regular px-6 py-4 rounded-2xl" style="display: block">
  <p class="font-bold text-2xl">Discover this Project</p>
  <p><strong>GitHub Link:</strong> <a href="https://github.com/sheet848/admin-dashboard" target="_blank">https://github.com/sheet848/admin-dashboard</a></p>
  <p><strong>See Live:</strong> <a href="https://admin-dashboard-one-ecru-48.vercel.app/" target="_blank">https://admin-dashboard-one-ecru-48.vercel.app/</a></p>
</div>

## Project Overview

The admin dashboard is a fully-featured application with:

- Light and dark theme support
- Six different data visualization charts
- Three types of advanced data tables
- Form validation system
- Interactive calendar
- Collapsible sidebar navigation

## The Tech Stack: Why These Choices Matter

### Material-UI: More Than Just Components

Before this project, I viewed UI libraries as simple component collections. Material-UI changed that perspective entirely. I learned that MUI is actually a complete design system with:

- **`Box`** simplifies layout without jumping between files. I could write styles directly on components using props like `display="flex"` or use the `sx` prop for more complex styling.
- **Theme Provider** helps create reusable design tokens. I could maintain visual consistency while easily switching between light and dark modes. The `useTheme` hook made accessing these values trivial throughout the application.

### React Router: Navigation Architecture

Implementing React Router v6 taught me proper routing patterns. 
- structure pages with parent/child routes
- integrate navigation seamlessly with MUI
- avoid page reloads while still keeping clarity

### Material-UI DataGrid: Enterprise-Level Tables

The DataGrid component was eye-opening. Coming from basic HTML tables, seeing built-in features like:

- Column sorting and filtering
- Row selection with checkboxes
- Export to CSV functionality
- Custom cell rendering
- Column visibility toggles

I learned that production applications need these features out of the box. Building them from scratch would take weeks, but DataGrid provided them with just configuration.

## Formik and Yup: Form Validation Done Right

### The Problem with Manual Validation

Before using Formik, I was handling forms with useState for every field, writing onChange handlers, and manually checking validation rules. It was repetitive and error-prone.

### The Formik Solution

Formik completely changed my approach:

**Initial Values Object**: Instead of multiple useState calls, one object held all form state. Cleaner, more organized.

**Validation Schema with Yup**: Yup's declarative validation felt like writing documentation. For an email field:

```javascript
email: yup.string()
  .email("Invalid email")
  .required("Required")
```

This approach separated validation logic from UI components, making both easier to test and maintain.

**Touched State**: The `touched` state was brilliant. Error messages only show after a user interacts with a field, preventing overwhelming users with errors before they've even started typing.

**Integration with Material-UI**: Formik works seamlessly with MUI TextField components. Simply passing `onBlur={handleBlur}`, `onChange={handleChange}`, and `value={values.fieldName}` connected everything. With React Hook Form, I'd need controller components and more boilerplate.

## Calendar Implementation: FullCalendar Integration

Building the calendar taught me about event-driven architecture. FullCalendar provides:

**Multiple View Modes**: Month, week, day, and list views came pre-built. Understanding how to configure the toolbar showed me how flexible third-party libraries can be when properly documented.

**Event Handling**: I implemented two key handlers:

1. `handleDateClick` - for creating events when users click dates
2. `handleEventClick` - for deleting events when clicked

**State Synchronization**: Keeping the sidebar event list in sync with calendar changes taught me about derived state and how to properly update React state when external libraries modify data.

## Data Visualization with Nivo Charts

### Why Nivo Over Other Libraries

Nivo stood out because:

- Built on D3 but with React-friendly APIs
- Beautiful defaults requiring minimal configuration
- Responsive by default
- Extensive theme customization

### The Learning Curve

**Understanding Data Structures**: Each chart type expects data in specific formats. I learned to structure mock data correctly:

- Bar charts need arrays of objects with x/y coordinates
- Pie charts need arrays with id and value properties
- Line charts need arrays of series, each containing data points

**Theme Customization**: Making charts match the application theme required understanding Nivo's theme object structure. I learned to customize:

- Axis colors and fonts
- Legend text colors
- Tooltip backgrounds
- Grid line colors

**Responsive Design**: Creating both dashboard widgets and full-page chart views taught me about the `isDashboard` prop pattern. The same component renders differently based on context—smaller legends, fewer tick marks, adjusted sizing for dashboard widgets.

## Theme System: Building for Light and Dark Mode

### The Architecture

The theme system taught me about proper state management for global UI concerns:

**Color Tokens Function**: Creating a function that returns different color palettes based on mode was clever:

```javascript
export const tokens = (mode) => ({
  ...(mode === 'dark' ? darkColors : lightColors)
});
```

**React Context**: Using Context API for theme state showed me when Context is appropriate (truly global state that changes infrequently).

**useMode Custom Hook**: Building this hook taught me to combine useMemo, useState, and Material-UI's createTheme for optimal performance.

## File Architecture: The Ducks Pattern

### Organization Philosophy

The project structure taught me about feature-based organization:

**Scenes Folder**: Each page gets its own folder with an index.jsx. This makes finding page-specific code trivial. Need to modify the dashboard? Go to scenes/dashboard.

**Components Folder**: Shared components like charts and reusable widgets live here. The key insight: if a component is used in multiple places, it belongs in components.

**Avoiding Over-nesting**: I learned to keep folder structures flat. Instead of components/dashboard/widgets/StatBox, it's simply components/StatBox. Easier navigation, less cognitive load.

## React Pro Sidebar: Navigation Patterns

Setting up the sidebar taught me about third-party component customization:

**CSS Overriding**: React Pro Sidebar has internal CSS classes. Using parent selectors to target these classes:

```javascript
"& .pro-sidebar-inner": {
  background: colors.primary[400]
}
```

This pattern works for any library with CSS classes that need theming.

**Collapse Functionality**: Implementing collapse required managing state and conditionally rendering content. The hamburger menu toggles between two states, showing/hiding text and adjusting widths.

**Active State Management**: Tracking the selected menu item taught me about URL-based state. Highlighting the active page based on the current route creates better UX.

## Key Takeaways and Lessons Learned

### 1. Component Reusability is Everything

Building reusable components like StatBox, ProgressCircle, and chart components saved massive amounts of time. The dashboard uses StatBox four times with different data—write once, use everywhere.

### 2. Mock Data Matters

Creating realistic mock data helped visualize the final product and test edge cases. Understanding data structures before building components prevented rework.

### 3. Third-Party Libraries are Your Friends

Trying to build a data grid or calendar from scratch would take months. Using battle-tested libraries accelerated development while providing more features than I could build alone.

### 4. Theme Consistency Creates Polish

Small details matter: matching scrollbar colors to the theme, consistent border radius values, unified color palettes. These seemingly minor touches separate amateur from professional applications.

## What I'd Do Differently

### 1. Mobile-First Design

I built for desktop, then realized mobile responsiveness needed significant rework. Starting mobile-first would have saved time and created a better product.

### 2. Component Documentation

Creating a component library with usage examples would help future me (and team members) understand how to use components like StatBox and ProgressCircle.

### 3. TypeScript

PropTypes helped catch errors, but TypeScript would provide better IDE support and catch more bugs at compile time.

### 4. Backend Integration Planning

I built with mock data, which was great for learning. But designing with API integration in mind from the start would make real data integration smoother.

### 5. Accessibility

I focused on functionality and aesthetics but didn't implement comprehensive ARIA labels, keyboard navigation, or screen reader support. Accessibility should be built in, not added later.

## Conclusion

Building this admin dashboard was transformative. The combination of Material-UI's design system, powerful libraries like Formik and Nivo, and proper architectural patterns taught me how professional developers build scalable applications.

For anyone building their own admin dashboard, I recommend:

1. Start with a solid design system (Material-UI, Chakra, etc.)
2. Plan your routes and folder structure before coding
3. Use established libraries for complex features
4. Focus on reusable components
5. Test with realistic data
6. Prioritize user experience in forms and interactions

This project is now a centerpiece of my portfolio and a reference point for future projects. The patterns, techniques, and tools I learned here will influence how I approach every React application going forward.

---

## Resources

- **Material-UI Documentation**: https://mui.com
- **Formik Documentation**: https://formik.org
- **Nivo Charts**: https://nivo.rocks
- **FullCalendar**: https://fullcalendar.io
- **React Router**: https://reactrouter.com

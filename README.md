# NoEats

A lightweight intermittent fasting tracker built with Vue 3 and Vite.

## Features

- **Duration mode** — set a fasting window by hours and minutes, with quick presets for common protocols (12h, 16h, 18h, 20h, 24h, 36h)
- **End date/time mode** — pick an exact datetime when the fast should end
- **Live progress** — real-time progress bar, elapsed time, and remaining time updated every second
- **Persistent state** — fasting session survives page reloads via cookies (no account required)

## Tech Stack

- [Vue 3](https://vuejs.org/) (Composition API)
- [Vite](https://vitejs.dev/)
- Deployed via [Azure Static Web Apps](https://learn.microsoft.com/azure/static-web-apps/)

## Getting Started

### Prerequisites

- Node.js 18+

### Install & run locally

```bash
npm install
npm run dev
```

### Build for production

```bash
npm run build
```

Preview the production build locally:

```bash
npm run preview
```

## Project Structure

```
src/
  App.vue       # Main application component
  main.js       # Vue app entry point
index.html
vite.config.js
staticwebapp.config.json  # Azure Static Web Apps routing config
```

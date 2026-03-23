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

## Deployment

The project deploys automatically to Azure Static Web Apps via GitHub Actions on every push to `main`.

### First-time setup

1. Create an Azure Static Web App in the [Azure Portal](https://portal.azure.com) (choose **Other** as the deployment method when prompted).
2. Once created, go to the resource → **Manage deployment token** and copy the token.
3. In your GitHub repository, go to **Settings → Secrets and variables → Actions → New repository secret** and add:
   - **Name:** `AZURE_STATIC_WEB_APPS_API_TOKEN`
   - **Value:** the token copied from step 2

After that, any push to `main` triggers the workflow in `.github/workflows/azure-static-web-apps.yml`, which:
- Installs dependencies and runs `npm run build`
- Deploys the `dist/` output to Azure Static Web Apps

Pull requests also get an automatic staging preview URL posted as a PR comment, which is torn down when the PR is closed.

> `GITHUB_TOKEN` is automatically provided by GitHub for every workflow run — no setup needed.

## Project Structure

```
src/
  App.vue       # Main application component
  main.js       # Vue app entry point
index.html
vite.config.js
staticwebapp.config.json  # Azure Static Web Apps routing config
```

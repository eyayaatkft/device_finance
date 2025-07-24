# device_finance
# Help Support Widget

A reusable React Help & Support Widget with chatbot functionality, styled with Tailwind CSS.

## Installation

```bash
npm install kft-ai-assistant
```

## Usage

```tsx
"use client";

import React from "react";
import 'kft-ai-assistant/dist/widget.css';
import {HelpSupportWidget}  from "kft-ai-assistant"

export default function ExperimentPage() {
  return (
            <HelpSupportWidget url="https://chegebeya.com/" />    
  );
} 
              

```

## Setting the Website URL

Pass the URL of your website to the widget using the `url` prop.  
The widget will automatically scrape your site, create a knowledge base, and extract support themes for the chat assistant.

## Styling

This widget bundles its own styles, including all required Tailwind CSS utilities and custom classes.  
You do **not** need to install or configure Tailwind CSS in your host app.  
If your app already uses Tailwind, the widget's styles are namespaced and preflight is disabled to avoid conflicts.  
**You must import `widget.css` from the package for proper styling.**

## Props

| Prop Name | Type   | Description                                                                 |
|-----------|--------|-----------------------------------------------------------------------------|
| url       | string | The main site URL. Used to scrape and create the knowledge base for this instance. |

## Backend Requirements

The widget expects a backend compatible with its endpoints for knowledge base and chat functionality.  
Required endpoints may include:

- `/knowledge/list`
- `/scrape-url`
- `/help-themes`
- `/chat`

See the source code for details.

## Development

- **Build:** `npm run build`
- **Build CSS:** `npm run build:css`
- **Entry:** `src/index.ts`

## License

MIT
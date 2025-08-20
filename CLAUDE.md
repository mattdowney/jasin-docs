# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the official documentation site for Jasin (https://getjasin.com) - a tool that transforms Amazon product URLs into beautiful, embeddable product cards. The documentation is built using Mintlify and deployed to https://docs.getjasin.com.

**Key Point**: Jasin is NOT an API service. Users interact through a dashboard at getjasin.com to create embeds and get script tags - no API endpoints exist for public use.

## Development Commands

```bash
# Start development server with auto-reload
npm run dev            # Runs: mint dev

# Production build
npm run build          # Runs: mint build

# Format code
npm run format         # Format all files with Prettier
npm run format:mdx     # Format only MDX files
npm run format:check   # Check formatting without making changes

# Check for broken links
mint broken-links

# Update Mint CLI
npm install -g mint@latest
```

## Architecture & Structure

### Documentation Structure
- **Root level**: `introduction.mdx`, `quickstart.mdx` - main landing pages
- **essentials/**: Core functionality documentation (`authentication.mdx`)
- **features/**: Feature-specific docs (`embed-cards.mdx`, `analytics.mdx`, `themes.mdx`)
- **images/**: Brand assets (logos, hero images)
- **mint.json**: Mintlify configuration (navigation, colors, branding)
- **styles.css**: Custom styling with Geist fonts and comprehensive code font overrides

### Technology Stack
- **Mintlify**: Documentation framework with MDX support
- **Geist fonts**: Primary font family for text and Geist Mono for code
- **Tailwind CSS**: Built-in utility classes via Mintlify
- **Custom CSS**: Extensive font overrides in styles.css

### Configuration Files
- **mint.json**: Controls site metadata, navigation structure, colors (#8b5cf6 primary), and external links
- **package.json**: Development dependencies include Prettier with XML and Tailwind plugins
- **styles.css**: Aggressive font overrides ensuring Geist/Geist Mono usage throughout

## Content Guidelines

### What Jasin Actually Does
Document the dashboard-driven workflow, NOT API functionality:
- Users paste Amazon URLs in dashboard at getjasin.com
- Users customize appearance and get script tags
- Analytics viewed in dashboard, not via API
- No authentication required for embedding

### URLs and Branding
- **Correct**: `https://getjasin.com` (main app), `https://docs.getjasin.com` (docs)
- **Avoid**: Old domains like `jasin.app` or `docs.jasin.app`

### Mintlify Components
Use these components in MDX files:
- `<Card>`, `<CardGroup>` for organized content
- `<Steps>` for sequential instructions
- `<CodeGroup>` for multi-language code examples
- `<AccordionGroup>`, `<Accordion>` for FAQs
- `<Info>`, `<Warning>` for callouts

## Development Workflow

1. **Local development**: `npm run dev` (auto-reload at localhost:3000)
2. **Format before commit**: `npm run format`
3. **Test locally**: Verify all links and formatting work
4. **Deploy**: Auto-deploys to docs.getjasin.com when pushed to master branch

## Important Notes

- No test framework is configured (test script returns error)
- Deployment is automatic via GitHub Actions on master branch push
- All content uses MDX format supporting React components
- Custom Jasin CSS classes available for branded styling
- Logo size is customized to h-5 (20px) instead of default h-7
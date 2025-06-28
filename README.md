# Jasin Documentation

Official documentation for [Jasin](https://getjasin.com) - Transform Amazon product URLs into
beautiful, embeddable product cards.

📖 **Live Documentation**: [docs.getjasin.com](https://docs.getjasin.com)

## What is Jasin?

Jasin is a simple tool that transforms Amazon product URLs into beautiful, embeddable product cards
for your website or blog. Users paste an Amazon link in the dashboard, customize the appearance, and
get a simple `<script>` tag to embed anywhere.

### Key Features

- **No API required** - Simple dashboard interface
- **Script tag embeds** - Just copy and paste
- **Real-time analytics** - Track views and clicks
- **Multiple themes** - Light, dark, and auto themes
- **Freemium model** - 5 lifetime credits free, 25/month for Pro

## Documentation Structure

```
jasin-docs/
├── introduction.mdx          # Main landing page
├── quickstart.mdx           # Getting started guide
├── mint.json               # Mintlify configuration
├── essentials/
│   └── authentication.mdx  # Account & dashboard guide
└── features/
    ├── embed-cards.mdx     # Creating and using embeds
    ├── analytics.mdx       # Analytics and tracking
    └── themes.mdx          # Theme customization
```

## Local Development

### Prerequisites

- Node.js 16+ and npm
- Git

### Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/your-username/jasin-docs.git
   cd jasin-docs
   ```

2. **Install Mint CLI**

   ```bash
   npm install -g mint@latest
   ```

3. **Start development server**

   ```bash
   mint dev
   ```

4. **Open in browser**
   - Visit `http://localhost:3000`
   - Documentation will auto-reload when you make changes

### Development Workflow

1. **Start development server**

   ```bash
   npm run dev
   ```

   This runs Mint dev server with built-in Tailwind CSS support.

2. **Edit documentation** - Modify `.mdx` files, `mint.json`, or `styles.css`
3. **Auto-reload** - Changes appear instantly at `localhost:3000`
4. **Test thoroughly** - Ensure all links and formatting work
5. **Commit and push** - Changes auto-deploy via GitHub Actions

### Available Commands

```bash
# Development
npm run dev              # Runs Mint dev server

# Production build
npm run build           # Build documentation for production
```

## Deployment

### Automatic Deployment

Documentation automatically deploys to [docs.getjasin.com](https://docs.getjasin.com) when changes
are pushed to the `main` branch via GitHub Actions.

### Manual Deployment

If needed, you can manually trigger a deployment in the Mintlify dashboard by clicking the "Refresh"
button.

## Writing Documentation

### File Format

All documentation files use MDX format (`.mdx`), which supports:

- Standard Markdown syntax
- React components
- Mintlify-specific components

### Mintlify Components

Common components used in this documentation:

````mdx
# Cards

<Card title="Title" icon="icon-name" href="/link">
  Description text
</Card>

# Card Groups

<CardGroup cols={2}>
  <Card title="Card 1" icon="icon">
    Content
  </Card>
  <Card title="Card 2" icon="icon">
    Content
  </Card>
</CardGroup>

# Code Groups

<CodeGroup>

```html HTML
<script src="https://getjasin.com/embed.js"></script>
```
````

```javascript JavaScript
console.log("Hello world");
```

</CodeGroup>

# Steps

<Steps>
  <Step title="Step 1">Do this first</Step>
  <Step title="Step 2">Then do this</Step>
</Steps>

# Accordions

<AccordionGroup>
  <Accordion title="Question">
    Answer content here
  </Accordion>
</AccordionGroup>

# Info/Warning boxes

<Info>Helpful information</Info> <Warning>Important warning</Warning>

````

### Tailwind CSS Integration

This documentation uses Tailwind CSS for custom styling via Mintlify's built-in support. You can use both Tailwind utility classes and custom Jasin components:

```mdx
# Custom Jasin Components
<div className="jasin-card">
  <h3 className="text-lg font-semibold mb-2">Card Title</h3>
  <p className="text-gray-600">Card content</p>
  <button className="jasin-button">Action Button</button>
</div>

# Tailwind Utility Classes
<div className="flex items-center space-x-4 p-6 bg-blue-50 rounded-lg">
  <span className="jasin-badge">Status</span>
  <span className="text-sm text-gray-500">Additional info</span>
</div>

# Alert Components
<div className="jasin-alert jasin-alert-info">
  <strong>Info:</strong> This is an informational message.
</div>

<div className="jasin-alert jasin-alert-warning">
  <strong>Warning:</strong> This is a warning message.
</div>

<div className="jasin-alert jasin-alert-success">
  <strong>Success:</strong> This is a success message.
</div>

# Responsive Grid
<div className="jasin-grid">
  <div className="jasin-card">Item 1</div>
  <div className="jasin-card">Item 2</div>
  <div className="jasin-card">Item 3</div>
</div>
````

#### Available Custom Classes

- `.jasin-card` - Styled card component with hover effects
- `.jasin-button` - Branded button with Jasin colors
- `.jasin-badge` - Small status/label badges
- `.jasin-grid` - Responsive grid (1/2/3 columns)
- `.jasin-alert`, `.jasin-alert-info`, `.jasin-alert-warning`, `.jasin-alert-success` - Alert
  components

#### Custom Colors

- `jasin-50` through `jasin-700` - Custom blue color palette matching the brand

### Cursor IDE Setup

For the best MDX development experience in Cursor:

1. **Install extensions** (run once):

   ```bash
   ./setup-cursor.sh
   ```

2. **Or install manually** in Cursor's Extensions panel:

   - `unifiedjs.vscode-mdx` - MDX syntax highlighting
   - `esbenp.prettier-vscode` - Code formatting
   - `bradlc.vscode-tailwindcss` - Tailwind CSS IntelliSense

3. **Cursor AI Benefits**:
   - Ask Cursor to help write MDX components
   - Get AI suggestions for Tailwind classes
   - Auto-complete JSX elements and props
   - Smart refactoring of documentation content

### Style Guide

- **Use clear, concise language** - Avoid technical jargon
- **Include examples** - Show real code snippets and use cases
- **Structure with headings** - Use proper heading hierarchy
- **Add visual elements** - Use cards, accordions, and code groups
- **Test all links** - Ensure internal and external links work

## Configuration

### mint.json

The `mint.json` file configures:

- Site metadata and branding
- Navigation structure
- Theme colors
- External links

Key sections:

```json
{
  "name": "Jasin Documentation",
  "navigation": [
    {
      "group": "Get Started",
      "pages": ["introduction", "quickstart", "essentials/authentication"]
    },
    {
      "group": "Features",
      "pages": ["features/embed-cards", "features/analytics", "features/themes"]
    }
  ]
}
```

## URLs and Branding

### Correct URLs

- **Main app**: `https://getjasin.com`
- **Documentation**: `https://docs.getjasin.com`
- **Support**: Community support via documentation, Pro users get priority email support

### Incorrect URLs (Do Not Use)

- ❌ `jasin.app` (old domain)
- ❌ `docs.jasin.app` (old docs domain)
- ❌ `support@jasin.app` (old email)

## What Jasin Actually Does

**Important**: Jasin is NOT an API service. The documentation should reflect that users:

1. **Use the dashboard** at getjasin.com to create embeds
2. **Get a script tag** to embed on their websites
3. **No API calls** or authentication required for embedding
4. **Analytics are viewed** in the dashboard, not via API

### What NOT to Document

- API endpoints (they don't exist for public use)
- Authentication for API access
- Programmatic embed creation
- Complex developer integrations

### What TO Document

- Dashboard workflow
- Script tag usage
- Theme options
- Analytics viewing
- Account management

## Contributing

### Making Changes

1. **Fork the repository**
2. **Create a feature branch**
   ```bash
   git checkout -b update-documentation
   ```
3. **Make your changes**
4. **Test locally** with `mint dev`
5. **Submit a pull request**

### Content Guidelines

- **Accuracy first** - Ensure all information reflects actual functionality
- **User-focused** - Write from the user's perspective
- **Clear examples** - Include practical, working examples
- **Consistent tone** - Maintain friendly, helpful tone throughout

## Useful Commands

```bash
# Start local development
mint dev

# Check for broken links
mint broken-links

# Update Mint CLI
npm install -g mint@latest

# Check git status
git status

# Commit changes
git add .
git commit -m "Update documentation"
git push origin main
```

## Support

- **Documentation Issues**: Create an issue in this repository
- **Product Support**: Pro users get priority email support, free users use community documentation
- **Mintlify Help**: [Mintlify Documentation](https://mintlify.com/docs)

## License

This documentation is part of the Jasin project. All rights reserved.

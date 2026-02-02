# Advanced Astro Patterns

This document covers advanced patterns, optimization techniques, and real-world scenarios.

---

## Content Collections (Type-Safe CMS)

### Schema Definition

```typescript
// src/content/config.ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  type: 'content', // or 'data' for JSON/YAML
  schema: z.object({
    title: z.string(),
    description: z.string(),
    publishDate: z.coerce.date(), // Coerce string to Date
    author: z.string().default('Anonymous'),
    tags: z.array(z.string()).default([]),
    image: z.object({
      src: z.string(),
      alt: z.string(),
    }).optional(),
    draft: z.boolean().default(false),
    featured: z.boolean().default(false),
  }),
});

const authors = defineCollection({
  type: 'data',
  schema: z.object({
    name: z.string(),
    email: z.string().email(),
    avatar: z.string().url(),
    bio: z.string(),
    social: z.object({
      twitter: z.string().optional(),
      github: z.string().optional(),
    }),
  }),
});

export const collections = { blog, authors };
```

---

### Query Content Collections

```astro
---
// src/pages/blog/index.astro
import { getCollection } from 'astro:content';

// Get all blog posts
const allPosts = await getCollection('blog');

// Filter published posts
const publishedPosts = await getCollection('blog', ({ data }) => {
  return data.draft !== true;
});

// Sort by date (newest first)
const sortedPosts = publishedPosts.sort((a, b) => {
  return b.data.publishDate.getTime() - a.data.publishDate.getTime();
});

// Get featured posts
const featuredPosts = await getCollection('blog', ({ data }) => {
  return data.featured === true && data.draft !== true;
});
---

<h1>Blog Posts</h1>

<!-- Featured Section -->
{featuredPosts.length > 0 && (
  <section>
    <h2>Featured</h2>
    {featuredPosts.map((post) => (
      <article>
        <a href={`/blog/${post.slug}`}>
          <h3>{post.data.title}</h3>
          <p>{post.data.description}</p>
        </a>
      </article>
    ))}
  </section>
)}

<!-- All Posts -->
<section>
  <h2>All Posts</h2>
  {sortedPosts.map((post) => (
    <article>
      <a href={`/blog/${post.slug}`}>
        <h3>{post.data.title}</h3>
        <time datetime={post.data.publishDate.toISOString()}>
          {post.data.publishDate.toLocaleDateString()}
        </time>
        <p>{post.data.description}</p>
      </a>
    </article>
  ))}
</section>
```

---

### Render Collection Entry

```astro
---
// src/pages/blog/[slug].astro
import { getCollection } from 'astro:content';

// Generate static paths for all posts
export async function getStaticPaths() {
  const posts = await getCollection('blog');
  
  return posts.map((post) => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;

// Render Markdown/MDX to HTML
const { Content } = await post.render();
---

<article>
  <header>
    <h1>{post.data.title}</h1>
    <p>By {post.data.author}</p>
    <time datetime={post.data.publishDate.toISOString()}>
      {post.data.publishDate.toLocaleDateString()}
    </time>
    
    {post.data.tags.length > 0 && (
      <ul>
        {post.data.tags.map((tag) => (
          <li><a href={`/tags/${tag}`}>{tag}</a></li>
        ))}
      </ul>
    )}
  </header>
  
  {post.data.image && (
    <img src={post.data.image.src} alt={post.data.image.alt} />
  )}
  
  <!-- Rendered Markdown/MDX content -->
  <Content />
</article>
```

---

## Image Optimization

### Built-in Image Component

```astro
---
import { Image } from 'astro:assets';
import heroImage from '../assets/hero.jpg'; // Local import
---

<!-- ✅ Optimized: auto WebP, responsive sizes -->
<Image 
  src={heroImage} 
  alt="Hero image"
  width={1200}
  height={630}
  format="webp"
  quality={80}
/>

<!-- Remote images -->
<Image 
  src="https://example.com/image.jpg"
  alt="Remote image"
  width={800}
  height={600}
  inferSize
/>
```

---

### Picture Component (Art Direction)

```astro
---
import { Picture } from 'astro:assets';
import desktopImage from '../assets/hero-desktop.jpg';
import mobileImage from '../assets/hero-mobile.jpg';
---

<Picture 
  src={desktopImage}
  formats={['avif', 'webp']}
  alt="Responsive hero"
  widths={[400, 800, 1200]}
  sizes="(max-width: 768px) 100vw, 1200px"
  fallbackFormat="jpg"
/>
```

---

## Server-Side Rendering (SSR)

### Dynamic Routes with SSR

```astro
---
// src/pages/products/[id].astro
export const prerender = false; // Enable SSR for this page

const { id } = Astro.params;

// Fetch data on each request (not at build time)
const response = await fetch(`https://api.example.com/products/${id}`);
const product = await response.json();

if (!product) {
  return Astro.redirect('/404');
}
---

<article>
  <h1>{product.name}</h1>
  <p>{product.description}</p>
  <p>Price: ${product.price}</p>
</article>
```

---

### API Endpoints (Server-Side)

```typescript
// src/pages/api/search.json.ts
import type { APIRoute } from 'astro';

export const GET: APIRoute = async ({ request }) => {
  const url = new URL(request.url);
  const query = url.searchParams.get('q');
  
  if (!query) {
    return new Response(JSON.stringify({ error: 'Missing query' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json' },
    });
  }
  
  // Fetch from database or external API
  const results = await searchDatabase(query);
  
  return new Response(JSON.stringify({ results }), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
};

export const POST: APIRoute = async ({ request }) => {
  const data = await request.json();
  
  // Process data (save to DB, send email, etc.)
  const result = await saveToDatabase(data);
  
  return new Response(JSON.stringify({ success: true, id: result.id }), {
    status: 201,
    headers: { 'Content-Type': 'application/json' },
  });
};
```

---

### Cookies & Headers (SSR)

```astro
---
// src/pages/dashboard.astro
export const prerender = false;

// Read cookies
const token = Astro.cookies.get('auth-token');

if (!token) {
  return Astro.redirect('/login');
}

// Set cookies
Astro.cookies.set('last-visit', new Date().toISOString(), {
  httpOnly: true,
  secure: true,
  maxAge: 60 * 60 * 24 * 7, // 7 days
});

// Read headers
const userAgent = Astro.request.headers.get('user-agent');
const clientIp = Astro.clientAddress;
---

<h1>Dashboard</h1>
<p>Token: {token.value}</p>
<p>User Agent: {userAgent}</p>
<p>IP: {clientIp}</p>
```

---

## Hybrid Rendering (Static + SSR)

```javascript
// astro.config.mjs
export default defineConfig({
  output: 'hybrid', // Default to static, opt-in to SSR
  adapter: vercel(),
});
```

```astro
---
// src/pages/about.astro
// This page is static (pre-rendered at build time)
export const prerender = true; // Explicit (default in hybrid mode)
---

---
// src/pages/dashboard.astro
// This page is SSR (rendered on each request)
export const prerender = false; // Opt-in to SSR
---
```

---

## Middleware (Request Interception)

```typescript
// src/middleware.ts
import { defineMiddleware } from 'astro:middleware';

export const onRequest = defineMiddleware(async (context, next) => {
  // Add custom headers
  context.response.headers.set('X-Custom-Header', 'Hello');
  
  // Check authentication
  const token = context.cookies.get('auth-token');
  
  if (context.url.pathname.startsWith('/admin') && !token) {
    return context.redirect('/login');
  }
  
  // Log request
  console.log(`${context.request.method} ${context.url.pathname}`);
  
  // Continue to page/endpoint
  const response = await next();
  
  // Modify response (optional)
  return response;
});
```

---

## Progressive Enhancement with Forms

### Form with JavaScript (Island)

```astro
---
// src/components/ContactForm.astro
---

<form id="contact-form" action="/api/contact" method="POST">
  <input type="text" name="name" required />
  <input type="email" name="email" required />
  <textarea name="message" required></textarea>
  <button type="submit">Send</button>
</form>

<script>
  // Progressive enhancement (works without JS too)
  const form = document.getElementById('contact-form') as HTMLFormElement;
  
  form?.addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const formData = new FormData(form);
    const data = Object.fromEntries(formData);
    
    try {
      const response = await fetch('/api/contact', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      
      if (response.ok) {
        alert('Message sent!');
        form.reset();
      } else {
        alert('Error sending message');
      }
    } catch (error) {
      console.error(error);
      // Fallback: submit form normally
      form.submit();
    }
  });
</script>
```

---

## Custom Integrations

```typescript
// integrations/my-integration.ts
import type { AstroIntegration } from 'astro';

export function myIntegration(): AstroIntegration {
  return {
    name: 'my-integration',
    hooks: {
      'astro:config:setup': ({ config, updateConfig, injectScript }) => {
        // Add custom Vite config
        updateConfig({
          vite: {
            plugins: [/* custom Vite plugins */],
          },
        });
        
        // Inject script into every page
        injectScript('page', 'console.log("Integration loaded");');
      },
      'astro:build:start': ({ buildConfig }) => {
        console.log('Build started:', buildConfig.outDir);
      },
      'astro:build:done': ({ dir, pages }) => {
        console.log(`Build complete: ${pages.length} pages in ${dir}`);
      },
    },
  };
}
```

```javascript
// astro.config.mjs
import { myIntegration } from './integrations/my-integration';

export default defineConfig({
  integrations: [myIntegration()],
});
```

---

## Pagination

```astro
---
// src/pages/blog/[page].astro
import { getCollection } from 'astro:content';

export async function getStaticPaths({ paginate }) {
  const posts = await getCollection('blog');
  const sortedPosts = posts.sort((a, b) => 
    b.data.publishDate.getTime() - a.data.publishDate.getTime()
  );
  
  return paginate(sortedPosts, {
    pageSize: 10, // Posts per page
  });
}

const { page } = Astro.props;
---

<h1>Blog - Page {page.currentPage}</h1>

<!-- Posts for current page -->
{page.data.map((post) => (
  <article>
    <a href={`/blog/${post.slug}`}>
      <h2>{post.data.title}</h2>
    </a>
  </article>
))}

<!-- Pagination controls -->
<nav>
  {page.url.prev && <a href={page.url.prev}>Previous</a>}
  <span>Page {page.currentPage} of {page.lastPage}</span>
  {page.url.next && <a href={page.url.next}>Next</a>}
</nav>
```

---

## RSS Feed Generation

```typescript
// src/pages/rss.xml.ts
import rss from '@astrojs/rss';
import { getCollection } from 'astro:content';

export async function GET(context) {
  const posts = await getCollection('blog');
  
  return rss({
    title: 'My Blog',
    description: 'A blog about web development',
    site: context.site,
    items: posts.map((post) => ({
      title: post.data.title,
      description: post.data.description,
      pubDate: post.data.publishDate,
      link: `/blog/${post.slug}`,
    })),
    customData: '<language>en-us</language>',
  });
}
```

---

## Sitemap Generation

```bash
npx astro add sitemap
```

```javascript
// astro.config.mjs
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://example.com', // Required for sitemap
  integrations: [sitemap()],
});
```

---

## Environment Variables

```env
# .env
PUBLIC_API_URL=https://api.example.com
SECRET_API_KEY=your-secret-key
```

```astro
---
// Client-side (public)
const apiUrl = import.meta.env.PUBLIC_API_URL;

// Server-side only (secret)
const apiKey = import.meta.env.SECRET_API_KEY;
---

<script>
  // ✅ Works (public)
  console.log(import.meta.env.PUBLIC_API_URL);
  
  // ❌ Undefined (not public)
  console.log(import.meta.env.SECRET_API_KEY);
</script>
```

---

## TypeScript Helpers

### Infer Props from Component

```astro
---
// src/components/Card.astro
interface Props {
  title: string;
  description?: string;
  href: string;
}

const { title, description, href } = Astro.props;
---

<a href={href}>
  <h3>{title}</h3>
  {description && <p>{description}</p>}
</a>
```

```typescript
// Usage with type safety
import type { ComponentProps } from 'astro/types';
import Card from './components/Card.astro';

type CardProps = ComponentProps<typeof Card>;

const props: CardProps = {
  title: 'My Card',
  href: '/page',
};
```

---

## Performance Optimization

### Lazy Load Images Below Fold

```astro
---
import { Image } from 'astro:assets';
import heroImage from '../assets/hero.jpg';
import belowFoldImage from '../assets/below-fold.jpg';
---

<!-- Above fold: eager loading -->
<Image 
  src={heroImage} 
  alt="Hero" 
  loading="eager"
  fetchpriority="high"
/>

<!-- Below fold: lazy loading -->
<Image 
  src={belowFoldImage} 
  alt="Below fold" 
  loading="lazy"
/>
```

---

### Preload Critical Assets

```astro
<head>
  <link rel="preload" as="image" href="/hero.jpg" />
  <link rel="preload" as="font" href="/fonts/inter.woff2" type="font/woff2" crossorigin />
</head>
```

---

### Bundle Size Analysis

```bash
npm run build -- --verbose
```

Look for:
- Client bundle sizes
- Unused dependencies
- Duplicate code

---

## Error Handling

### Custom 404 Page

```astro
---
// src/pages/404.astro
---
<html>
  <head>
    <title>Page Not Found</title>
  </head>
  <body>
    <h1>404 - Page Not Found</h1>
    <a href="/">Go Home</a>
  </body>
</html>
```

---

### Error Boundaries (React Islands)

```jsx
// src/components/ErrorBoundary.jsx
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

```astro
---
import ErrorBoundary from '../components/ErrorBoundary.jsx';
import BuggyComponent from '../components/BuggyComponent.jsx';
---

<ErrorBoundary client:load>
  <BuggyComponent client:load />
</ErrorBoundary>
```

---

## Testing Strategies

### Unit Testing Astro Components

```typescript
// tests/components/Card.test.ts
import { experimental_AstroContainer as AstroContainer } from 'astro/container';
import { expect, test } from 'vitest';
import Card from '../../src/components/Card.astro';

test('Card renders with title', async () => {
  const container = await AstroContainer.create();
  const result = await container.renderToString(Card, {
    props: { title: 'Test', href: '/test' },
  });
  
  expect(result).toContain('Test');
});
```

---

### E2E Testing with Playwright

```typescript
// tests/e2e/homepage.spec.ts
import { test, expect } from '@playwright/test';

test('homepage loads and has title', async ({ page }) => {
  await page.goto('/');
  
  await expect(page).toHaveTitle(/My Site/);
  await expect(page.locator('h1')).toBeVisible();
});

test('navigation works', async ({ page }) => {
  await page.goto('/');
  
  await page.click('a[href="/blog"]');
  await expect(page).toHaveURL(/\/blog/);
});
```

---

## Resources

- **Content Collections**: https://docs.astro.build/en/guides/content-collections/
- **Image Optimization**: https://docs.astro.build/en/guides/images/
- **SSR Guide**: https://docs.astro.build/en/guides/server-side-rendering/
- **Middleware**: https://docs.astro.build/en/guides/middleware/
- **Custom Integrations**: https://docs.astro.build/en/reference/integrations-reference/

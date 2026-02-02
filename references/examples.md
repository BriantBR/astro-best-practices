# Astro Examples & Recipes

Real-world examples and common patterns for Astro projects.

---

## Recipe 1: Blog with Content Collections

### Directory Structure

```
src/
├── content/
│   ├── config.ts
│   └── blog/
│       ├── first-post.md
│       ├── second-post.mdx
│       └── third-post.md
├── layouts/
│   └── BlogLayout.astro
└── pages/
    ├── blog/
    │   ├── index.astro
    │   ├── [slug].astro
    │   └── tags/
    │       └── [tag].astro
    └── rss.xml.ts
```

---

### Blog Schema

```typescript
// src/content/config.ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  schema: ({ image }) => z.object({
    title: z.string(),
    description: z.string(),
    publishDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    author: z.string(),
    tags: z.array(z.string()),
    image: image().optional(),
    draft: z.boolean().default(false),
  }),
});

export const collections = { blog };
```

---

### Blog Index Page

```astro
---
// src/pages/blog/index.astro
import { getCollection } from 'astro:content';
import Layout from '../../layouts/Layout.astro';

const posts = await getCollection('blog', ({ data }) => !data.draft);
const sortedPosts = posts.sort((a, b) => 
  b.data.publishDate.getTime() - a.data.publishDate.getTime()
);

// Get all unique tags
const allTags = [...new Set(posts.flatMap(post => post.data.tags))];
---

<Layout title="Blog">
  <h1>Blog Posts</h1>
  
  <!-- Tag filter -->
  <nav>
    <a href="/blog">All</a>
    {allTags.map(tag => (
      <a href={`/blog/tags/${tag}`}>{tag}</a>
    ))}
  </nav>
  
  <!-- Posts list -->
  <section>
    {sortedPosts.map(post => (
      <article>
        <a href={`/blog/${post.slug}`}>
          <h2>{post.data.title}</h2>
          <time datetime={post.data.publishDate.toISOString()}>
            {post.data.publishDate.toLocaleDateString('en-US', {
              year: 'numeric',
              month: 'long',
              day: 'numeric',
            })}
          </time>
          <p>{post.data.description}</p>
        </a>
      </article>
    ))}
  </section>
</Layout>
```

---

### Single Blog Post

```astro
---
// src/pages/blog/[slug].astro
import { getCollection } from 'astro:content';
import { Image } from 'astro:assets';
import BlogLayout from '../../layouts/BlogLayout.astro';

export async function getStaticPaths() {
  const posts = await getCollection('blog');
  return posts.map(post => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content, headings } = await post.render();
---

<BlogLayout 
  title={post.data.title}
  description={post.data.description}
  publishDate={post.data.publishDate}
  author={post.data.author}
>
  <article>
    <header>
      {post.data.image && (
        <Image 
          src={post.data.image} 
          alt={post.data.title}
          width={1200}
          height={630}
        />
      )}
      
      <h1>{post.data.title}</h1>
      
      <div class="meta">
        <time datetime={post.data.publishDate.toISOString()}>
          {post.data.publishDate.toLocaleDateString()}
        </time>
        <span>By {post.data.author}</span>
      </div>
      
      <ul class="tags">
        {post.data.tags.map(tag => (
          <li><a href={`/blog/tags/${tag}`}>{tag}</a></li>
        ))}
      </ul>
    </header>
    
    <!-- Table of contents -->
    {headings.length > 0 && (
      <aside class="toc">
        <h2>Table of Contents</h2>
        <ul>
          {headings.map(heading => (
            <li class={`level-${heading.depth}`}>
              <a href={`#${heading.slug}`}>{heading.text}</a>
            </li>
          ))}
        </ul>
      </aside>
    )}
    
    <!-- Rendered content -->
    <Content />
  </article>
</BlogLayout>
```

---

### Blog Layout

```astro
---
// src/layouts/BlogLayout.astro
import BaseLayout from './BaseLayout.astro';

interface Props {
  title: string;
  description: string;
  publishDate: Date;
  author: string;
}

const { title, description, publishDate, author } = Astro.props;
---

<BaseLayout title={title} description={description}>
  <slot />
  
  <!-- Share buttons (static, no JS) -->
  <aside class="share">
    <h3>Share this post</h3>
    <a 
      href={`https://twitter.com/intent/tweet?text=${encodeURIComponent(title)}&url=${Astro.url}`}
      target="_blank"
      rel="noopener"
    >
      Twitter
    </a>
    <a 
      href={`https://www.linkedin.com/sharing/share-offsite/?url=${Astro.url}`}
      target="_blank"
      rel="noopener"
    >
      LinkedIn
    </a>
  </aside>
</BaseLayout>

<style>
  article {
    max-width: 65ch;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .meta {
    display: flex;
    gap: 1rem;
    color: #666;
  }
  
  .tags {
    display: flex;
    gap: 0.5rem;
    list-style: none;
    padding: 0;
  }
  
  .toc {
    background: #f5f5f5;
    padding: 1rem;
    border-radius: 8px;
    margin: 2rem 0;
  }
  
  .toc .level-2 { margin-left: 1rem; }
  .toc .level-3 { margin-left: 2rem; }
</style>
```

---

## Recipe 2: Multi-Page Form with React

### Step 1: Form Component (React Island)

```jsx
// src/components/MultiStepForm.jsx
import { useState } from 'react';

export default function MultiStepForm() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    company: '',
    message: '',
  });
  const [status, setStatus] = useState('');

  const handleChange = (e) => {
    setFormData(prev => ({
      ...prev,
      [e.target.name]: e.target.value,
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus('Sending...');

    try {
      const response = await fetch('/api/contact', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData),
      });

      if (response.ok) {
        setStatus('Message sent successfully!');
        setFormData({ name: '', email: '', company: '', message: '' });
        setStep(1);
      } else {
        setStatus('Error sending message.');
      }
    } catch (error) {
      setStatus('Network error.');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {step === 1 && (
        <div>
          <h2>Step 1: Your Info</h2>
          <input
            type="text"
            name="name"
            value={formData.name}
            onChange={handleChange}
            placeholder="Name"
            required
          />
          <input
            type="email"
            name="email"
            value={formData.email}
            onChange={handleChange}
            placeholder="Email"
            required
          />
          <button type="button" onClick={() => setStep(2)}>
            Next
          </button>
        </div>
      )}

      {step === 2 && (
        <div>
          <h2>Step 2: Company Info</h2>
          <input
            type="text"
            name="company"
            value={formData.company}
            onChange={handleChange}
            placeholder="Company"
          />
          <button type="button" onClick={() => setStep(1)}>
            Back
          </button>
          <button type="button" onClick={() => setStep(3)}>
            Next
          </button>
        </div>
      )}

      {step === 3 && (
        <div>
          <h2>Step 3: Message</h2>
          <textarea
            name="message"
            value={formData.message}
            onChange={handleChange}
            placeholder="Your message"
            required
          />
          <button type="button" onClick={() => setStep(2)}>
            Back
          </button>
          <button type="submit">Submit</button>
        </div>
      )}

      {status && <p>{status}</p>}
    </form>
  );
}
```

---

### Step 2: API Endpoint

```typescript
// src/pages/api/contact.json.ts
import type { APIRoute } from 'astro';

export const POST: APIRoute = async ({ request }) => {
  const data = await request.json();

  // Validation
  if (!data.name || !data.email || !data.message) {
    return new Response(
      JSON.stringify({ error: 'Missing required fields' }),
      { status: 400 }
    );
  }

  // Send email (example with Resend)
  try {
    await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${import.meta.env.RESEND_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        from: 'noreply@example.com',
        to: 'hello@example.com',
        subject: `New contact from ${data.name}`,
        html: `
          <p><strong>Name:</strong> ${data.name}</p>
          <p><strong>Email:</strong> ${data.email}</p>
          <p><strong>Company:</strong> ${data.company || 'N/A'}</p>
          <p><strong>Message:</strong></p>
          <p>${data.message}</p>
        `,
      }),
    });

    return new Response(JSON.stringify({ success: true }), { status: 200 });
  } catch (error) {
    console.error(error);
    return new Response(
      JSON.stringify({ error: 'Failed to send email' }),
      { status: 500 }
    );
  }
};
```

---

### Step 3: Page Integration

```astro
---
// src/pages/contact.astro
import Layout from '../layouts/Layout.astro';
import MultiStepForm from '../components/MultiStepForm.jsx';
---

<Layout title="Contact Us">
  <h1>Contact Us</h1>
  <MultiStepForm client:idle />
</Layout>
```

---

## Recipe 3: Dark Mode Toggle

### Theme Toggle Component (Vanilla JS)

```astro
---
// src/components/ThemeToggle.astro
---

<button id="theme-toggle" aria-label="Toggle dark mode">
  <span class="sun">☀️</span>
  <span class="moon">🌙</span>
</button>

<script>
  const theme = (() => {
    if (typeof localStorage !== 'undefined' && localStorage.getItem('theme')) {
      return localStorage.getItem('theme');
    }
    if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
      return 'dark';
    }
    return 'light';
  })();

  if (theme === 'dark') {
    document.documentElement.classList.add('dark');
  } else {
    document.documentElement.classList.remove('dark');
  }

  window.localStorage.setItem('theme', theme);

  const handleToggle = () => {
    const element = document.documentElement;
    element.classList.toggle('dark');

    const isDark = element.classList.contains('dark');
    localStorage.setItem('theme', isDark ? 'dark' : 'light');
  };

  document
    .getElementById('theme-toggle')
    ?.addEventListener('click', handleToggle);
</script>

<style>
  #theme-toggle {
    background: transparent;
    border: none;
    cursor: pointer;
    font-size: 1.5rem;
  }

  .sun { display: none; }
  .moon { display: block; }

  :global(.dark) .sun { display: block; }
  :global(.dark) .moon { display: none; }
</style>
```

---

### Global Styles with Dark Mode

```css
/* src/styles/global.css */
:root {
  --bg: #ffffff;
  --text: #000000;
  --accent: #3b82f6;
}

.dark {
  --bg: #1a1a1a;
  --text: #ffffff;
  --accent: #60a5fa;
}

body {
  background: var(--bg);
  color: var(--text);
}

a {
  color: var(--accent);
}
```

---

## Recipe 4: Authentication with Cookies

### Login Page

```astro
---
// src/pages/login.astro
import Layout from '../layouts/Layout.astro';

const error = Astro.url.searchParams.get('error');
---

<Layout title="Login">
  <h1>Login</h1>
  
  {error && <p class="error">{error}</p>}
  
  <form action="/api/auth/login" method="POST">
    <input type="email" name="email" placeholder="Email" required />
    <input type="password" name="password" placeholder="Password" required />
    <button type="submit">Login</button>
  </form>
</Layout>
```

---

### Login API Endpoint

```typescript
// src/pages/api/auth/login.ts
import type { APIRoute } from 'astro';

export const POST: APIRoute = async ({ request, cookies, redirect }) => {
  const formData = await request.formData();
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  // Validate credentials (example - use proper auth library in production)
  if (email === 'user@example.com' && password === 'password') {
    // Set auth cookie
    cookies.set('auth-token', 'your-secure-token', {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 60 * 60 * 24 * 7, // 7 days
      path: '/',
    });

    return redirect('/dashboard');
  }

  return redirect('/login?error=Invalid credentials');
};
```

---

### Protected Page

```astro
---
// src/pages/dashboard.astro
export const prerender = false; // Enable SSR

const token = Astro.cookies.get('auth-token');

if (!token) {
  return Astro.redirect('/login');
}

// Fetch user data
const user = await getUserFromToken(token.value);
---

<Layout title="Dashboard">
  <h1>Welcome, {user.name}!</h1>
  
  <form action="/api/auth/logout" method="POST">
    <button type="submit">Logout</button>
  </form>
</Layout>
```

---

### Logout Endpoint

```typescript
// src/pages/api/auth/logout.ts
import type { APIRoute } from 'astro';

export const POST: APIRoute = async ({ cookies, redirect }) => {
  cookies.delete('auth-token', { path: '/' });
  return redirect('/');
};
```

---

## Recipe 5: SEO with Dynamic Metadata

### SEO Component

```astro
---
// src/components/SEO.astro
interface Props {
  title: string;
  description: string;
  image?: string;
  article?: boolean;
  publishDate?: Date;
  author?: string;
}

const {
  title,
  description,
  image = '/default-og.jpg',
  article = false,
  publishDate,
  author,
} = Astro.props;

const canonicalURL = new URL(Astro.url.pathname, Astro.site);
const imageURL = new URL(image, Astro.site);
---

<!-- Primary Meta Tags -->
<title>{title}</title>
<meta name="title" content={title} />
<meta name="description" content={description} />
<link rel="canonical" href={canonicalURL} />

<!-- Open Graph / Facebook -->
<meta property="og:type" content={article ? 'article' : 'website'} />
<meta property="og:url" content={canonicalURL} />
<meta property="og:title" content={title} />
<meta property="og:description" content={description} />
<meta property="og:image" content={imageURL} />

{article && publishDate && (
  <meta property="article:published_time" content={publishDate.toISOString()} />
)}
{article && author && (
  <meta property="article:author" content={author} />
)}

<!-- Twitter -->
<meta property="twitter:card" content="summary_large_image" />
<meta property="twitter:url" content={canonicalURL} />
<meta property="twitter:title" content={title} />
<meta property="twitter:description" content={description} />
<meta property="twitter:image" content={imageURL} />

<!-- Additional Meta -->
<meta name="robots" content="index, follow" />
<meta name="googlebot" content="index, follow" />
```

---

### Usage

```astro
---
import SEO from '../components/SEO.astro';
---

<html>
  <head>
    <SEO 
      title="My Blog Post"
      description="An amazing blog post about Astro"
      image="/blog/post-image.jpg"
      article={true}
      publishDate={new Date('2024-01-15')}
      author="John Doe"
    />
  </head>
  <body>
    <!-- Content -->
  </body>
</html>
```

---

## Recipe 6: Internationalization (i18n)

### Directory Structure

```
src/
├── pages/
│   ├── index.astro           # Default (English)
│   ├── es/
│   │   └── index.astro       # Spanish
│   └── fr/
│       └── index.astro       # French
└── i18n/
    ├── en.json
    ├── es.json
    └── fr.json
```

---

### Translation Files

```json
// src/i18n/en.json
{
  "nav.home": "Home",
  "nav.about": "About",
  "nav.contact": "Contact",
  "hero.title": "Welcome to My Site",
  "hero.subtitle": "Building amazing web experiences"
}
```

```json
// src/i18n/es.json
{
  "nav.home": "Inicio",
  "nav.about": "Acerca de",
  "nav.contact": "Contacto",
  "hero.title": "Bienvenido a Mi Sitio",
  "hero.subtitle": "Construyendo experiencias web increíbles"
}
```

---

### Translation Helper

```typescript
// src/i18n/utils.ts
const translations = {
  en: () => import('./en.json').then(m => m.default),
  es: () => import('./es.json').then(m => m.default),
  fr: () => import('./fr.json').then(m => m.default),
};

export async function getTranslations(lang: keyof typeof translations) {
  return await translations[lang]();
}

export function getLangFromURL(url: URL) {
  const [, lang] = url.pathname.split('/');
  if (lang in translations) return lang as keyof typeof translations;
  return 'en';
}
```

---

### Multilingual Page

```astro
---
// src/pages/es/index.astro
import { getTranslations } from '../../i18n/utils';

const t = await getTranslations('es');
---

<html lang="es">
  <head>
    <title>{t['hero.title']}</title>
  </head>
  <body>
    <nav>
      <a href="/es">{t['nav.home']}</a>
      <a href="/es/about">{t['nav.about']}</a>
      <a href="/es/contact">{t['nav.contact']}</a>
    </nav>
    
    <main>
      <h1>{t['hero.title']}</h1>
      <p>{t['hero.subtitle']}</p>
    </main>
  </body>
</html>
```

---

## Resources

- **Astro Themes**: https://astro.build/themes
- **Community Examples**: https://github.com/withastro/astro/tree/main/examples
- **Astro Discord**: https://astro.build/chat

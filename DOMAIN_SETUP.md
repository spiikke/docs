# Domain Setup Instructions for twinodds.io/docs

This guide explains how to set up your domain so that `twinodds.io/docs` seamlessly routes to your Mintlify documentation site.

## Recommended Approach: Subdomain with Redirect

This is the easiest and most reliable method. It involves:
1. Hosting Mintlify docs on `docs.twinodds.io`
2. Redirecting `/docs` requests from your main app to the subdomain

### Step 1: Configure Mintlify Custom Domain

1. Go to [Mintlify Dashboard](https://dashboard.mintlify.com)
2. Select your project
3. Navigate to **Settings** → **Domain**
4. Click **Add Custom Domain**
5. Enter: `docs.twinodds.io`
6. Copy the DNS verification record provided

### Step 2: Configure DNS

Add a CNAME record in your DNS provider:

```
Type: CNAME
Name: docs
Value: [Mintlify-provided domain]
TTL: Auto or 3600
```

**For Cloudflare:**
- Go to DNS settings for `twinodds.io`
- Add CNAME: `docs` → `[Mintlify domain]`
- Proxy status: Proxied (recommended) or DNS only

### Step 3: Verify Domain in Mintlify

1. Back in Mintlify dashboard, click **Verify Domain**
2. Wait for SSL certificate provisioning (automatic, ~5-10 minutes)
3. Test by visiting `https://docs.twinodds.io`

### Step 4: Configure Redirect in Your Web App

Since you're using Next.js (based on your `web-frontend` structure), add redirects:

**Option A: Using `next.config.js`** (Recommended)

Edit `web-frontend/next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  async redirects() {
    return [
      {
        source: '/docs',
        destination: 'https://docs.twinodds.io',
        permanent: true, // 308 permanent redirect
      },
      {
        source: '/docs/:path*',
        destination: 'https://docs.twinodds.io/:path*',
        permanent: true,
      },
    ];
  },
};

module.exports = nextConfig;
```

**Option B: Using Vercel** (If deploying on Vercel)

Create/edit `vercel.json` in your project root:

```json
{
  "redirects": [
    {
      "source": "/docs",
      "destination": "https://docs.twinodds.io",
      "permanent": true
    },
    {
      "source": "/docs/:path*",
      "destination": "https://docs.twinodds.io/:path*",
      "permanent": true
    }
  ]
}
```

### Step 5: Deploy and Test

1. Commit and push your changes
2. Deploy your Next.js application
3. Visit `https://twinodds.io/docs` - should redirect to `https://docs.twinodds.io`
4. Test internal navigation within docs

## Alternative: Reverse Proxy (Advanced)

If you want `/docs` to serve Mintlify content directly without redirects, use a reverse proxy. This is more complex but provides seamless subdirectory integration.

### Using Next.js API Route

Create `web-frontend/app/docs/[[...slug]]/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { slug?: string[] } }
) {
  const path = params.slug?.join('/') || '';
  const searchParams = request.nextUrl.search;
  const mintlifyUrl = `https://docs.twinodds.io/${path}${searchParams}`;
  
  try {
    const response = await fetch(mintlifyUrl, {
      headers: {
        'User-Agent': request.headers.get('user-agent') || '',
        'Accept': request.headers.get('accept') || 'text/html',
      },
    });
    
    const contentType = response.headers.get('content-type') || '';
    const body = await response.text();
    
    // Rewrite URLs in HTML to maintain /docs prefix
    let modifiedBody = body;
    if (contentType.includes('text/html')) {
      modifiedBody = body
        .replace(/href="\//g, 'href="/docs/')
        .replace(/src="\//g, 'src="/docs/')
        .replace(/action="\//g, 'action="/docs/');
    }
    
    return new NextResponse(modifiedBody, {
      status: response.status,
      headers: {
        'Content-Type': contentType,
        'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
      },
    });
  } catch (error) {
    console.error('Failed to fetch documentation:', error);
    return NextResponse.json(
      { error: 'Failed to fetch documentation' },
      { status: 500 }
    );
  }
}
```

## Troubleshooting

### Redirect Loop

**Solution:** Ensure redirect destination doesn't include `/docs` path.

### DNS Not Propagating

**Check:** Use `dig docs.twinodds.io` or `nslookup docs.twinodds.io`  
**Wait:** DNS propagation can take 5-30 minutes (up to 24 hours in rare cases)

### SSL Certificate Issues

**Solution:** Mintlify automatically provisions SSL via Let's Encrypt. Wait 5-10 minutes after domain verification.

### CSS/JS Not Loading

**Solution:** Ensure all asset URLs are rewritten correctly in reverse proxy setup.

## Testing Checklist

After setup, verify:

- ✅ `https://twinodds.io/docs` redirects correctly
- ✅ `https://twinodds.io/docs/getting-started` works
- ✅ Internal links within docs work
- ✅ Search functionality works
- ✅ Mobile responsiveness maintained
- ✅ Analytics tracking works (if configured)

## Performance Considerations

- **Redirects:** Add minimal latency (~50-100ms)
- **Caching:** Consider caching redirects at CDN level
- **SEO:** Redirects are SEO-friendly and establish canonical URLs

## Next Steps

1. Customize Mintlify branding in dashboard
2. Configure analytics and tracking
3. Set up custom error pages
4. Consider adding sitemap for SEO


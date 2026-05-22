# fareez.dev — Static site deployment

Self-contained static site. No build step, no dependencies. Drop the files into S3 and serve.

## Files

```
fareez-dev-site/
├── index.html      Main page
├── 404.html        Error page (matches design)
├── favicon.svg     Site icon
├── robots.txt      Crawler rules
├── sitemap.xml     Search engine sitemap
└── README.md       This file (don't upload to S3)
```

Total upload: ~5 files. Everything else (fonts, etc.) loads from CDN.

---

## Quick deploy (S3 static hosting only — HTTP)

Good for testing. For production with HTTPS and your custom domain, see the CloudFront section below.

### 1. Create the bucket

In the AWS Console → S3 → Create bucket:

- **Bucket name**: `fareez.dev` (must match your domain exactly if using website hosting with a custom domain)
- **Region**: pick one close to you — `ap-southeast-2` (Sydney) makes sense from Melbourne
- **Block all public access**: **uncheck this**. Confirm the acknowledgement. The site needs to be publicly readable.
- Leave the rest as default. Create.

### 2. Enable static website hosting

In the bucket → **Properties** → scroll to **Static website hosting** → Edit:

- **Enable**
- **Hosting type**: Host a static website
- **Index document**: `index.html`
- **Error document**: `404.html`
- Save changes

Note the **Bucket website endpoint** URL it gives you — something like `http://fareez.dev.s3-website-ap-southeast-2.amazonaws.com`. You'll use this later.

### 3. Set bucket policy for public read

In the bucket → **Permissions** → **Bucket policy** → paste this (replace `fareez.dev` with your bucket name if different):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::fareez.dev/*"
    }
  ]
}
```

Save.

### 4. Upload the files

In the bucket → **Objects** → **Upload** → drag in:

- `index.html`
- `404.html`
- `favicon.svg`
- `robots.txt`
- `sitemap.xml`

**Do not upload `README.md`.**

Click Upload.

### 5. Test

Visit the website endpoint URL from step 2. The site should load.

If a file shows up as a download instead of rendering, check the file's metadata → set `Content-Type` to `text/html` for `.html` files, `image/svg+xml` for `favicon.svg`, etc. S3 usually auto-detects from extension, but if you uploaded via CLI without `--content-type` it might not.

---

## Production deploy (CloudFront + HTTPS + custom domain)

This is what you actually want. The S3-only setup above doesn't support HTTPS on a custom domain.

### Architecture

```
fareez.dev (Route 53 / your DNS)
    ↓
CloudFront (HTTPS, edge cache)
    ↓
S3 bucket (private, accessed only via CloudFront)
```

### 1. Create the S3 bucket (private this time)

Same as before but:
- Bucket name: anything (e.g. `fareez-dev-site-prod`) — doesn't need to match domain when used via CloudFront
- **Keep "Block all public access" ON**
- Don't enable static website hosting on the bucket itself — CloudFront will read it directly
- Upload the files

### 2. Request an SSL certificate

AWS Console → **Certificate Manager** → **must switch region to `us-east-1`** (CloudFront only accepts certs from there).

- Request a public certificate
- Domain names: `fareez.dev` and `*.fareez.dev` (the wildcard covers `www`, future subdomains, and the tool sub-pages if any go on subdomains)
- Validation: DNS validation
- Add the CNAME records ACM gives you to your DNS provider. Cert issues in a few minutes once validated.

### 3. Create the CloudFront distribution

CloudFront → Create distribution:

- **Origin domain**: pick your S3 bucket from the dropdown (it'll show as `bucketname.s3.region.amazonaws.com`)
- **Origin access**: choose "Origin access control settings (recommended)" → create a new OAC → save
- **Viewer protocol policy**: Redirect HTTP to HTTPS
- **Default root object**: `index.html`
- **Alternate domain names (CNAMEs)**: `fareez.dev`, `www.fareez.dev`
- **SSL certificate**: select the ACM cert from step 2
- Create distribution

After creation, CloudFront will show a banner asking you to update the S3 bucket policy. Copy the policy it gives you (it grants CloudFront read access to your private bucket) and paste it into the bucket's permissions → bucket policy.

### 4. Custom error responses

CloudFront → your distribution → **Error pages** → Create custom error response:

- HTTP error code: 403 (S3 returns 403 for not-found in private buckets)
- Customize error response: Yes
- Response page path: `/404.html`
- HTTP response code: 404

Repeat for 404. This makes S3-not-found turn into your styled 404 page.

### 5. Point DNS at CloudFront

In your DNS provider for `fareez.dev`:

- Create an **ALIAS / ANAME** record (Route 53) or **CNAME** (Cloudflare, Namecheap, etc.) from `fareez.dev` → your CloudFront distribution domain (`d1234abc.cloudfront.net`)
- Same for `www.fareez.dev`

Propagation: 5 min to a few hours.

### 6. Test

- `https://fareez.dev` should load with a valid SSL certificate
- `https://fareez.dev/nonsense` should show your styled 404 page
- `https://fareez.dev/favicon.svg` should load

---

## Cache & content-type tips

When you upload via the AWS CLI, set proper headers. Here's a one-liner that handles it correctly:

```bash
aws s3 sync . s3://your-bucket-name/ \
  --exclude "README.md" \
  --exclude ".*" \
  --cache-control "public, max-age=3600" \
  --delete
```

After every deploy, invalidate the CloudFront cache so users see your changes immediately:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DIST_ID \
  --paths "/*"
```

---

## Updating the site

For a small site like this, the easiest workflow:

```bash
# from the site directory
aws s3 sync . s3://your-bucket-name/ --exclude "README.md" --delete
aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

Two commands. Done.

---

## Notes

- **Fonts** load from Google Fonts CDN. No action needed.
- **No build step** — these files are ready to serve as-is.
- **No JavaScript framework** — just a tiny vanilla `<script>` for the live clock. Loads instantly.
- **Mobile**: tested responsive down to ~360px.
- **Adding tool sub-pages later** (Zakat, Inheritance, etc.): create them as folders with their own `index.html`, e.g. `/zakat/index.html`, and CloudFront will serve them at `/zakat/` automatically.

---

## Troubleshooting

**"Access Denied" when visiting the site (CloudFront setup)**
The bucket policy from CloudFront wasn't applied. Re-copy it from the distribution and paste into the bucket's Permissions → Bucket policy.

**Site shows old content after update**
Invalidate the CloudFront cache (see commands above). Until invalidation completes (~1 min), edge nodes still serve old files.

**Fonts not loading**
Check browser console. Likely a CSP issue if you've added one — make sure `https://fonts.googleapis.com` and `https://fonts.gstatic.com` are allowed.

**Favicon not showing**
Hard refresh (Cmd/Ctrl+Shift+R). Browsers aggressively cache favicons.

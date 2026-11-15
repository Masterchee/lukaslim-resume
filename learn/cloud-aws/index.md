# Cloud (AWS): How a Website Reaches Your Browser

AWS has hundreds of services, but a website only needs a handful. This guide follows one request for a static site and names each service it touches: **Route 53** (DNS), **CloudFront** (CDN), **ACM** (certificates), **S3** (storage) and **IAM** (permissions).

> **Note:** this is the standard reference architecture for a static site on AWS. `www.lukaslim.com` keeps its source in GitHub and can be deployed in different ways, so treat the diagram as the pattern to learn, not as an exact map of this site.

---

## 1. The 30-Second Mental Model

Think of a shop chain. The **phone book** (Route 53) tells you the address of the nearest branch. The **local branch** (CloudFront edge location) keeps popular items on the shelf, so you don't wait for the **warehouse** (S3). The **security guard** (ACM + IAM) makes sure the connection is encrypted and that only the branch, not the public, can enter the warehouse.

```text
Browser
   |  1. "What is the IP of www.example.com?"
   v
Route 53 (DNS)  -- alias record --> CloudFront distribution
   |
   |  2. HTTPS to the nearest edge location (certificate from ACM)
   v
CloudFront edge (cache)
   |  3. cache hit  -> return file immediately
   |  3. cache miss -> fetch from origin, store, return
   v
S3 bucket (private)  <-- only CloudFront may read (OAC + bucket policy)
```

---

## 2. How It Works in Practice

### 2.1 Route 53: DNS and routing
A **hosted zone** holds the DNS records for your domain. For a CloudFront site, `www` points to the distribution with an **alias record** (an AWS-specific record type that works like an A record but targets an AWS resource).

| Record | Type | Points to |
|--------|------|-----------|
| `www.example.com` | A (Alias) | CloudFront distribution (`dxxxx.cloudfront.net`) |
| `example.com` | A (Alias) | Same distribution, or a redirect to `www` |

Check it from any terminal:
```bash
dig +short www.example.com       # returns CloudFront edge IP addresses
dig NS example.com               # the name servers Route 53 assigned
```
If you registered the domain elsewhere, the registrar's name server settings must match the hosted zone's NS records.

### 2.2 S3: storage
An S3 **bucket** stores your files as **objects**. For a site served through CloudFront, keep the bucket **private** (Block Public Access on) and let CloudFront read it through its REST endpoint. The bucket does not need "static website hosting" turned on in that setup.

### 2.3 CloudFront: delivery and caching
A **distribution** has an **origin** (your S3 bucket) and one or more **behaviors** (rules for which paths use which cache settings). The first viewer in a region causes a cache miss, and CloudFront fetches from S3. Later viewers get the cached copy from a nearby **edge location**.

Key settings:
- **Default root object:** `index.html`, so `/` returns the home page.
- **Viewer protocol policy:** redirect HTTP to HTTPS.
- **TTL:** how long a file stays cached. After uploading a change you may need to run an **invalidation** (for example for `/*`), or visitors keep seeing the old version until the TTL expires.

Note that CloudFront only serves a default root object for the **top-level** `/`. A request for `/learn/` needs either a CloudFront Function that rewrites it to `/learn/index.html`, or a direct link to the full file path.

### 2.4 ACM: HTTPS certificates
**AWS Certificate Manager** issues free public TLS certificates. A certificate used by CloudFront must be requested in the **US East (N. Virginia) `us-east-1`** region, whatever region your other resources use. Validation is done with a CNAME record in Route 53 (the console has a one-click "Create records in Route 53" button).

### 2.5 IAM: least privilege
**IAM** controls who or what may do what.
- **Principle of least privilege:** grant only the actions and resources needed.
- **Origin Access Control (OAC)** lets CloudFront sign its requests to S3. The bucket policy then allows only that distribution:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontRead",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR-BUCKET/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT-ID:distribution/DISTRIBUTION-ID"
      }
    }
  }]
}
```
Replace `YOUR-BUCKET`, `ACCOUNT-ID` and `DISTRIBUTION-ID` with your own values. The console can generate this policy for you when you attach OAC to the origin.

### Alternative: GitHub plus AWS Amplify
If your code lives in GitHub, **AWS Amplify Hosting** can connect to the repository and redeploy on every push. It runs CloudFront and S3 for you behind the scenes, so you manage fewer pieces, at the cost of seeing fewer of them.

---

## 3. What Happens When It Breaks

| Symptom | Likely cause | Where to look |
|---------|--------------|---------------|
| Domain doesn't resolve (`NXDOMAIN`) | Missing record, or registrar name servers don't match the hosted zone | Route 53 hosted zone, `dig NS` |
| Domain resolves to the wrong place | Old record, or a stale DNS cache (TTL) | `dig +short`, record TTL |
| Browser certificate warning | Certificate missing the domain name, expired, or not attached to the distribution | ACM certificate, distribution alternate domain names |
| Can't select the certificate in CloudFront | Requested outside `us-east-1` | ACM region |
| `403 AccessDenied` (XML error) | Bucket policy or OAC missing, or the file doesn't exist | S3 bucket policy, object key spelling |
| `403`/`404` for `/learn/` but `/learn/index.html` works | No default-document handling for subfolders | CloudFront Function, or link to the full path |
| Old content after an update | CloudFront still serving the cached copy | Create an invalidation |
| Changes work in S3 but not on the site | Looking at S3 directly instead of the CDN URL | Test using the distribution domain |
| `502` / `504` from CloudFront | Origin unreachable or too slow | Origin settings |

Debug from the outside in:
```bash
curl -I https://www.example.com/
```
Useful response headers: `x-cache: Hit from cloudfront` or `Miss from cloudfront`, `x-amz-cf-pop` (which edge location answered), and `age` (seconds the object has been cached).

---

## 4. Hands-On Lab

Use the **AWS Management Console** with a free-tier account or sandbox. Allow about 30 minutes. Delete everything afterwards.

**Goal:** serve a one-page site privately from S3 through CloudFront over HTTPS.

1. **S3:** Create a bucket (any unique name). Leave **Block all public access** on. Upload a simple `index.html`.
2. **CloudFront:** Create a distribution.
   - Origin domain: choose your S3 bucket.
   - Origin access: **Origin access control settings**, and create a new OAC.
   - Viewer protocol policy: **Redirect HTTP to HTTPS**.
   - Default root object: `index.html`.
3. **Bucket policy:** After creating the distribution, copy the policy CloudFront offers and paste it into the bucket's **Permissions → Bucket policy**.
4. **Test:** Wait until the distribution status is **Deployed**, then open `https://<distribution-id>.cloudfront.net`. Your page should load.
5. **Prove it's private:** Try the S3 object URL directly. You should get **Access Denied**.
6. **Watch caching:**
   ```bash
   curl -I https://<distribution-id>.cloudfront.net/
   curl -I https://<distribution-id>.cloudfront.net/
   ```
   The second request should show `x-cache: Hit from cloudfront`.
7. **Stale content:** Edit and re-upload `index.html`. Reload and see the old copy, then create an **invalidation** for `/*` and confirm the new version appears.
8. **Optional, with your own domain:** Request an ACM certificate in `us-east-1`, add your domain as an alternate domain name on the distribution, and create a Route 53 **Alias** record to it.
9. **Clean up:** Disable and delete the distribution, empty and delete the bucket, and delete any certificate or Route 53 records you created.

**Check yourself:** at step 5, why did the direct S3 URL fail while CloudFront succeeded?

---

## Further Reading
- [AWS docs: Get started with a secure static website](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/getting-started-secure-static-website-cloudformation-template.html)
- [Set up S3 with CloudFront for a static site](https://oneuptime.com/blog/post/2026-02-12-s3-cloudfront-cdn-backed-static-site/view)
- [Static site hosting on AWS: S3, CloudFront, ACM and Route 53](https://dev.to/josh_blair/static-site-hosting-on-aws-s3-cloudfront-acm-and-route-53-20b2)
- [Host a static site with HTTPS and your own domain](https://www.golinuxcloud.com/create-s3-static-website-custom-domain/)

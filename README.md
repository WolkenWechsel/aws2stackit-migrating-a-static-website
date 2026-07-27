# AWS to STACKIT: A Static Website Migration 

If you have built or managed a web project over the last decade, you are likely familiar with the standard AWS pattern: host static assets (HTML, CSS, JS) in an Amazon S3 bucket and place Amazon CloudFront in front as a Content Delivery Network (CDN). It is efficient, cost-effective, and widely adopted.

We ran this exact architecture for our marketing website, **`wolkenwechsel.at`**.

```
[ Current AWS Architecture ]

       User / Browser
             │
             ▼
    ┌─────────────────┐
    │ Amazon CloudFront│ (CDN & SSL Termination)
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │   Amazon S3     │ (Static Website Hosting)
    └────────┴────────┘
```
*(Diagram 1: Classic AWS S3 + CloudFront deployment flow)*

Our goal now was to transition our own marketing website: execute a clean migration off US hyperscalers and move to **STACKIT**, the European cloud platform powered by Schwarz Group. 

Here is our technical review—why transferring the data was straightforward, why the native CDN configuration encountered limitations in beta, and what a fully European setup requires today.

---

## Step 1: Transferring Assets to STACKIT Object Storage via IaC & AWS CLI

Using **Infrastructure as Code (IaC)** principles simplifies cloud migrations significantly. Because STACKIT provides an S3-compatible Object Storage API, no proprietary tools or complex conversion scripts were necessary.

Using the standard AWS CLI pointed to STACKIT endpoints, we mirrored our site assets in two commands:

```bash
# 1. Download site files locally from AWS S3
aws s3 sync s3://wolkenwechsel-aws-bucket ./site-files

# 2. Upload directly to STACKIT Object Storage
aws s3 sync ./site-files s3://wolkenwechsel-a-sample-bucket-123 \
  --endpoint-url https://object.storage.eu01.onstackit.cloud
```

The storage transfer was completed in seconds, placing all static files securely inside a GDPR-compliant European data center.

---

## Step 2: Configuring STACKIT CDN via Terraform / OpenTofu

To turn an object storage bucket into a production website with HTTPS termination and edge caching, a CDN is required. STACKIT offers a **CDN service** (currently in **Beta**), which can be managed declaratively using Terraform or OpenTofu:

```hcl
resource "stackit_objectstorage_bucket" "my_bucket" {
  project_id = local.project_id
  name       = "wolkenwechsel-a-sample-bucket-123"
}

resource "stackit_cdn_distribution" "bucket_cdn375" {
  project_id = local.project_id

  config = {
    backend = {
      type       = "bucket"
      bucket_url = "https://wolkenwechsel-a-sample-bucket-123.object.storage.eu01.onstackit.cloud"
      region     = local.region

      credentials = {
        access_key_id     = var.bucket_access_key
        secret_access_key = var.bucket_secret_key
      }
    }
    regions = ["EU"]
  }

  lifecycle {
    ignore_changes = [
      config
    ]
  }
}
```

The deployment executed cleanly, creating the distribution linked directly to our STACKIT bucket as the backend source.

---

## The Roadblock: Root URL Routing & Directory Exposure

When testing the site through the newly generated CDN domain, we encountered unexpected behavior: 

Requesting the root domain (`/`) did not render `index.html`. Instead, it executed an S3 `ListObjects` request directly against the STACKIT bucket.

> ⚠️ **The Issue:** Rather than serving the default index document, the public root URL displayed an XML index of all raw files stored in the bucket. Exposing raw bucket structures presents both a usability challenge and an unnecessary exposure of metadata.

In Amazon CloudFront, you specify a **Default Root Object** (e.g., `index.html`), which transparently maps `/` to `/index.html`. 

We attempted to resolve this in STACKIT by defining custom redirect rules to route root requests to `index.html`. However, given the current feature scope of the CDN beta, the redirect rules did not resolve the behavior as intended.

---

## Conclusion: Is European Static Hosting Feasible Today?

We have temporarily paused this direct, native migration for `wolkenwechsel.at`.

**Does this mean hosting outside US hyperscalers is unavailable?**  
No. STACKIT's underlying Object Storage is performant, stable, and fully functional. 

However, hosting a static website on STACKIT today without exposing bucket listings typically requires combining STACKIT Object Storage with dedicated European third-party services—such as a European CDN provider (e.g **Bunny.net** configured with EU-only routing), or deploying a lightweight reverse proxy (such as Nginx or Envoy) on a STACKIT Compute instance.

We will update this article as soon as STACKIT's CDN service exits beta and introduces native default root document support.

## Ready for the "Wolkenwechsel"?

Schedule an initial consultation with our certified cloud architects to discuss your STACKIT migration requirements and evaluation strategy.

* **Website:** [wolkenwechsel.at](wolkenwechsel.at)
* **Email:** [beratung@wolkenwechsel.at](mailto:beratung@wolkenwechsel.at)
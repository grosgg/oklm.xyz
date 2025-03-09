---
layout: post
title: "Deploying a Jekyll site with GitHub Actions, AWS S3 and CloudFront"
categories: coding
image: https://cdn.oklm.xyz/blog/coding/2025-03-09/deploy-workflow-slim.png

---
Take a look around. The site you're browsing right now is powered by Jekyll, a static site generator that transforms plain text into fast websites. What keeps me coming back to Jekyll is its perfect blend of simplicity and customization potential, especially as a Ruby developer. In this guide, I'll walk through automating Jekyll deployments using GitHub Actions, then serving your site through AWS S3 and CloudFront for a cost-effective, scalable hosting solution.

![Deploy Workflow](https://cdn.oklm.xyz/blog/coding/2025-03-09/deploy-workflow-slim.png)

# The Plan
So, you built a Jekyll site. Now we need to deploy and serve this website. Here is what we need:
- a AWS S3 bucket hosting our site files
- a AWS CloudFront distribution to serve these files fast worldwide
- a Github Actions workflow to orchestrate and automate deployments

# Hosting on S3
Let’s start by creating a home for your website files. Head over to AWS S3 and:

## Create your bucket
Name it something obvious like `your-website-name.com`. AWS requires unique names globally.

## Turn on static hosting
In the bucket’s **Properties** tab, scroll to “Static website hosting” and enable it. Set `index.html` as your default page.

## Adjust permissions carefully
In the **Permissions** tab:
- Uncheck “Block __all__ public access” (don’t worry – we’ll add safety measures next)
- Add this bucket policy to allow visitors to view your site without letting them accidentally delete anything:

```json
{  
  "Version": "2012-10-17",  
  "Statement": [{  
    "Effect": "Allow",  
    "Principal": "*",  
    "Action": "s3:GetObject",  
    "Resource": "arn:aws:s3:::your-website-name.com/*"  
  }]  
}
```
You’ve now got a secure space to host your site files. Double-check everything by uploading a test `index.html` and visiting your S3 website URL (you’ll find it in the Static Website Hosting section).

# Serving with CloudFront
Now that your site’s files are in S3, you might notice it loads a bit slowly for visitors far from your bucket’s region. This is where Amazon’s CDN comes in. Here’s how to set it up:
- In the AWS Console, navigate to CloudFront and click “Create distribution.”
- Choose your S3 bucket from the dropdown (the one you named earlier)
- Under “Viewer protocol policy”, select “Redirect HTTP to HTTPS.” This ensures visitors always use a secure connection
- In the “Allowed HTTP methods” section, choose GET and HEAD.

This keeps things simple and secure for static sites. Once created, it might take 5-10 minutes for CloudFront to deploy your distribution. You’ll know it’s ready when the status changes from “In Progress” to “Deployed” – then test your new CDN URL!

# Creating Least-Privilege AWS Credentials
To restrict permissions to only your specific S3 bucket and CloudFront distribution:

## Create an IAM Policy
In AWS IAM, go to Policies > Create policy and paste this JSON, replacing YOUR_BUCKET_NAME and YOUR_CLOUDFRONT_DISTRIBUTION_ID with your actual values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_CLOUDFRONT_DISTRIBUTION_ID"
    }
  ]
}
```

## Attach to a New IAM User
- Create a user named `github-actions-deploy`
- Attach the custom policy you just created

## Add to GitHub Secrets
Use the generated access key ID and secret key in your repository’s secrets as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## Why This Matters
- The S3 permissions allow syncing files (PutObject, DeleteObject) and listing bucket contents (ListBucket).
- CloudFront access is limited to cache invalidation only for your specific distribution
- No access to other buckets, services, or administrative actions

# Automating the whole thing
Let’s set up automatic deployments so your site updates whenever you push changes to GitHub. Create a new file in your repository at .github/workflows/main.yml with this configuration:

```yaml{% raw %}
name: Deploy

on:
  push:
    branches:
    - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v1

    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        bundler-cache: true
    
    - name: Build Jekyll site
      run: JEKYLL_ENV=production bundle exec jekyll build

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: your-aws-region

    - name: Deploy static site to S3 bucket
      run: aws s3 sync ./_site/ ${{ secrets.AWS_S3_BUCKET_NAME }} --delete

    - name: Invalidate CloudFront cache
      run: aws cloudfront create-invalidation --distribution-id ${{ secrets.AWS_CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
{% endraw %}```

Let’s break this down:

1. Checkout
Grabs your latest code from the GitHub repository so the workflow can access it.

2. Set up Ruby
Prepares the Ruby environment and installs dependencies (including Jekyll) using Bundler for efficient package management.

3. Build Jekyll Site
Compiles your static site files using Jekyll’s build command, ready for deployment.

4. Configure AWS Credentials
Securely connects to your AWS account using credentials stored in GitHub’s Secrets vault (never exposed in plain text).

5. Deploy to S3
Synchronizes the generated `_site` folder with your S3 bucket, removing any outdated files automatically.

6. Clear CDN Cache
Tells CloudFront to refresh its cached content worldwide, ensuring visitors see your updates immediately.

This setup creates a reliable way to deploy your Jekyll site. GitHub Actions automates the build and deployment process, S3 securely hosts your files, and CloudFront ensures faster loading times globally. While the initial configuration requires careful steps, the system runs quietly in the background once everything’s connected.

You’ll only need to focus on updating your site’s content while the workflow handles the rest whenever you push changes. I hope this guide helps you. Let me know in the comments if you have any questions. Happy blogging!

---
layout: default
title: Authentication
nav_order: 3
description: "Configuring OIDC authentication for PaperTool"
permalink: /authentication/
---

# Authentication
{: .no_toc }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Overview

PaperTool uses **OpenID Connect (OIDC)** authentication, mediated by an AWS Cognito User Pool, to ensure that only authorised users can submit or update working papers. Unauthenticated visitors can still browse and download papers, but the upload form requires a sign-in.

### How it works

1. A visitor opens the PaperTool site. The page detects they are not signed in and shows a **Sign in** button.
2. Clicking it redirects the browser to the **Cognito hosted UI** login page, which in turn forwards the user to your organisation's OIDC provider (Google, Okta, Microsoft Entra, etc.).
3. After the user authenticates with the provider, Cognito exchanges the OIDC token for a short-lived **Cognito User Pool token** and redirects back to the PaperTool site.
4. The site completes a PKCE code exchange with the Cognito token endpoint and stores the resulting tokens in `sessionStorage`.
5. Subsequent S3 uploads and API Gateway calls use these tokens — the browser is granted temporary IAM credentials via Cognito Identity (scoped to `temp/*` only), and the API Gateway verifies the Cognito ID token before invoking Lambda.

No cookies or server-side sessions are involved. Tokens are discarded when the browser tab is closed.

---

## Prerequisites

Before deploying the `CreateLambdaAPI.yaml` stack you will need:

| Item | Description |
|------|-------------|
| OIDC provider | A provider your users already have accounts with (Google Workspace, Okta, Microsoft Entra, etc.) |
| Client ID | Issued by the provider when you register the application |
| Client secret | Issued by the provider alongside the client ID |
| Cognito domain prefix | A **globally unique** lowercase string (letters, digits, hyphens) that becomes `<prefix>.auth.<region>.amazoncognito.com` |
| Site website URL | The `WebsiteURL` output from the `CreateS3Buckets` CloudFormation stack |

---

## Registering PaperTool with your OIDC provider

### Step 1 — Decide your Cognito domain prefix

Pick a short, unique prefix. For example: `myuniversity-papertool`. The full Cognito domain will be:

```
https://myuniversity-papertool.auth.<aws-region>.amazoncognito.com
```

You will register this prefix as a CloudFormation parameter (`CognitoDomainPrefix`), so choose it before deploying the Lambda stack.

### Step 2 — Register the redirect URI with your OIDC provider

Every OIDC provider needs to know which redirect URI Cognito will use when it relays the authorisation response. Register the following URL with your provider:

```
https://<CognitoDomainPrefix>.auth.<aws-region>.amazoncognito.com/oauth2/idpresponse
```

For example, if your prefix is `myuniversity-papertool` and your region is `ap-southeast-2`:

```
https://myuniversity-papertool.auth.ap-southeast-2.amazoncognito.com/oauth2/idpresponse
```

Provider-specific instructions follow below.

---

## Provider-specific setup

### Google Workspace / Google accounts

1. Open the [Google Cloud Console](https://console.cloud.google.com/) and select or create a project.
2. Go to **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client IDs**.
3. Set **Application type** to **Web application**.
4. Under **Authorised redirect URIs**, add:
   ```
   https://<CognitoDomainPrefix>.auth.<aws-region>.amazoncognito.com/oauth2/idpresponse
   ```
5. Click **Create**. Copy the **Client ID** and **Client secret**.
6. Use these values in the CloudFormation parameters:
   - `OIDCProviderUrl` → `https://accounts.google.com`
   - `OIDCClientId` → your Client ID
   - `OIDCClientSecret` → your Client secret
   - `OIDCScopes` → `email profile openid` (default)

> **Tip:** If you want to restrict sign-in to users from a specific Google Workspace domain, you can add a Cognito pre-token-generation Lambda trigger that checks the `hd` claim in the ID token. See [AWS documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html).

---

### Microsoft Entra ID (Azure AD)

1. In the [Azure portal](https://portal.azure.com/), go to **Entra ID → App registrations → New registration**.
2. Set **Supported account types** to the appropriate scope for your organisation.
3. Under **Redirect URI**, add:
   ```
   https://<CognitoDomainPrefix>.auth.<aws-region>.amazoncognito.com/oauth2/idpresponse
   ```
4. After registration, go to **Certificates & secrets → New client secret** and copy the value immediately.
5. Go to **Overview** and copy the **Application (client) ID** and **Directory (tenant) ID**.
6. Use these values in the CloudFormation parameters:
   - `OIDCProviderUrl` → `https://login.microsoftonline.com/<tenant-id>/v2.0`
   - `OIDCClientId` → Application (client) ID
   - `OIDCClientSecret` → client secret value
   - `OIDCScopes` → `openid email profile` (default)

---

### Okta

1. In your Okta Admin Console, go to **Applications → Create App Integration**.
2. Select **OIDC – OpenID Connect** and **Web Application**.
3. Under **Sign-in redirect URIs**, add:
   ```
   https://<CognitoDomainPrefix>.auth.<aws-region>.amazoncognito.com/oauth2/idpresponse
   ```
4. Copy the **Client ID** and **Client secret** from the app settings page.
5. Use these values in the CloudFormation parameters:
   - `OIDCProviderUrl` → `https://<your-okta-domain>/oauth2/default`
   - `OIDCClientId` → your Client ID
   - `OIDCClientSecret` → your Client secret
   - `OIDCScopes` → `openid email profile` (default)

---

### Generic OIDC provider

Any standards-compliant OIDC provider will work. You need:

| Parameter | Value |
|-----------|-------|
| `OIDCProviderUrl` | The issuer URL — usually the base URL of the discovery document (without `/.well-known/openid-configuration`) |
| `OIDCClientId` | Client / application ID issued by the provider |
| `OIDCClientSecret` | Client secret issued by the provider |
| `OIDCScopes` | Scopes that include `openid` and `email`. Default: `email profile openid` |

The redirect URI to register with the provider is always:
```
https://<CognitoDomainPrefix>.auth.<aws-region>.amazoncognito.com/oauth2/idpresponse
```

---

## Deploying the Lambda stack

When deploying `CreateLambdaAPI.yaml` via CloudFormation you will be prompted for the following parameters:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `OIDCProviderUrl` | OIDC issuer URL | `https://accounts.google.com` |
| `OIDCClientId` | Client ID from your provider | `123456789-abc.apps.googleusercontent.com` |
| `OIDCClientSecret` | Client secret *(marked NoEcho — not shown in console)* | `GOCSPX-…` |
| `OIDCScopes` | OAuth scopes (space-separated) | `email profile openid` |
| `CognitoDomainPrefix` | Unique prefix for your Cognito domain | `myuniversity-papertool` |
| `SiteWebsiteUrl` | `WebsiteURL` output from the S3 stack | `http://econdept-aaa-site.s3-website-ap-southeast-2.amazonaws.com` |

> **Important:** `CognitoDomainPrefix` must be unique across **all** AWS Cognito users in the region, not just your account. If deployment fails with a domain conflict error, choose a more specific prefix.

After the stack reaches `CREATE_COMPLETE`, go to the **Outputs** tab and note the following values:

| Output key | Used in |
|------------|---------|
| `ApiEndpoint` | `settings.js → apiEndpoint` |
| `IdentityPoolId` | `settings.js → identityPoolId` |
| `UserPoolId` | `settings.js → userPoolId` |
| `UserPoolClientId` | `settings.js → userPoolClientId` |
| `CognitoDomain` | `settings.js → cognitoDomain` |

---

## Configuring settings.js

Open `aws_resources/s3_buckets/site_bucket/assets/js/settings.js` and fill in **all** fields, including the three new authentication fields:

```javascript
const config = {
    // existing fields (unchanged)
    archiveCode:          "aaa",
    seriesCode:           "ssssss",
    siteBucket:           "myuniversity-aaa-site",
    workingPapersBucket:  "myuniversity-aaa-archive",
    awsRegion:            "ap-southeast-2",
    repecHandle:          "RePEc:aaa:ssssss",
    templateUrl:          "https://myuniversity-aaa-archive.s3.ap-southeast-2.amazonaws.com/template/cover.png",
    apiEndpoint:          "https://xxxxxxxxxx.execute-api.ap-southeast-2.amazonaws.com/v1/upload",
    identityPoolId:       "ap-southeast-2:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",

    // new authentication fields — values from CreateLambdaAPI stack Outputs tab
    userPoolId:           "ap-southeast-2_XXXXXXXXX",
    userPoolClientId:     "xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    cognitoDomain:        "myuniversity-papertool.auth.ap-southeast-2.amazoncognito.com"
}
```

After saving, upload the file to the `SiteBucket` in S3, replacing the existing copy.

---

## Security model

| Action | Who can do it |
|--------|---------------|
| Browse / download papers | Anyone (public `s3:GetObject`) |
| Upload to `temp/` (browser → S3) | Authenticated Cognito users only |
| Read `metadata.json` or list the archive | Authenticated Cognito users only |
| Write final papers / RDF / index files | Lambda execution role only |
| Invoke the upload API | Cognito ID token required (API Gateway authoriser) |

The S3 bucket policies no longer grant `s3:PutObject` to `*`. Write access to the final paper paths is restricted to the Lambda IAM role, which is never exposed to the browser.

---

## Troubleshooting

**"Sign in" button does nothing / page freezes**  
Check the browser console for errors. The most common cause is a mismatch between the `cognitoDomain` in `settings.js` and the actual Cognito domain. Verify the `CognitoDomain` output in CloudFormation.

**Redirect back to the site shows a Cognito error about an invalid callback URL**  
The `SiteWebsiteUrl` parameter you entered in CloudFormation does not match the URL the browser is currently on. Make sure you used the exact `WebsiteURL` value from the S3 stack output, with no trailing slash.

**Error: "redirect_uri_mismatch" from your OIDC provider**  
The redirect URI registered with your OIDC provider does not match what Cognito is sending. Ensure you registered exactly:
```
https://<CognitoDomainPrefix>.auth.<region>.amazoncognito.com/oauth2/idpresponse
```

**403 Forbidden from the API Gateway after sign-in**  
The Cognito ID token may have expired (default: 1 hour). Sign out and sign back in to get a fresh token.

**S3 PutObject denied when uploading a paper**  
Confirm that the `identityPoolId` in `settings.js` matches the `IdentityPoolId` output from CloudFormation, and that the `userPoolId` and `userPoolClientId` are also correct. The Cognito Identity Pool links the User Pool token to temporary IAM credentials.

**Stack creation fails with "Domain already exists"**  
The `CognitoDomainPrefix` is already taken in this region. Choose a more unique prefix (e.g. include your institution abbreviation and a short random suffix) and redeploy.

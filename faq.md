---
layout: default
title: FAQ
nav_order: 4
description: "..."
permalink: /faq/
---

## FAQ

- **Where can I find the values for `settings.js`?**  
   In CloudFormation, select the Lambda stack you created and navigate to the **Outputs** tab. You may need to widen your browser window to see this tab.
  <!-- <img src="https://raw.githubusercontent.com/sodalabsio/papertool/main/assets/images/cloudformation2.png"> -->
    <img src="/assets/images/cloudformation_outputs.png">
  - The table lists all output keys and values. Copy `ApiEndpoint`, `IdentityPoolId`, `UserPoolId`, `UserPoolClientId`, and `CognitoDomain` into `settings.js`.

- **Which OIDC providers are supported?**  
  Any standards-compliant OIDC provider works — Google Workspace, Microsoft Entra ID (Azure AD), Okta, and others. See the [Authentication](../authentication/) page for provider-specific setup instructions.

- **Can I restrict sign-in to users from my organisation only?**  
  Yes. If you are using Google Workspace, your OIDC provider can be configured to issue tokens only for your domain. For other providers, the Cognito User Pool can be configured with attribute-based access control or a pre-token-generation Lambda trigger. Alternatively, you can configure the Cognito User Pool to require admin approval before new users can sign in.

- **What happens when a Cognito token expires?**  
  Cognito ID tokens expire after 1 hour by default. The user will need to sign out and sign back in to get a fresh token. The sign-out button is available in the top-right corner of the PaperTool site once signed in.

- **I get a 403 error when trying to upload a paper.**  
  Ensure you are signed in (the sign-out button should be visible). If you are signed in, your token may have expired — sign out and sign in again. Also confirm that `identityPoolId`, `userPoolId`, and `userPoolClientId` in `settings.js` match the CloudFormation outputs exactly.

- **Stack deployment fails with "Domain already exists".**  
  The `CognitoDomainPrefix` you chose is already taken in this AWS region. Choose a more specific prefix (e.g. include your institution abbreviation and a short random string) and redeploy.
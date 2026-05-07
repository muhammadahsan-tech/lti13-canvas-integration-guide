# LTI 1.3 Canvas Integration Guide

> A practical, no-fluff guide to integrating LTI 1.3 tools with Canvas LMS — written by [Muhammad Ahsan](https://github.com/muhammadahsan-tech), an LTI 1.3 architect who has implemented the spec from both sides of the handshake.

This guide covers everything from Canvas Developer Key setup to full LTI Advantage Services (AGS, NRPS, Deep Linking). It is written for **developers building LTI tools** and **IT administrators** configuring Canvas.

---

## Table of Contents

- [How Canvas LTI 1.3 Works](#how-canvas-lti-13-works)
- [What You Need Before You Start](#what-you-need-before-you-start)
- [Canvas-Specific URLs](#canvas-specific-urls)
- [Part 1: Create a Developer Key in Canvas](#part-1-create-a-developer-key-in-canvas)
- [Part 2: Deploy the Tool to an Account or Course](#part-2-deploy-the-tool-to-an-account-or-course)
- [Part 3: Configure Your Tool to Work with Canvas](#part-3-configure-your-tool-to-work-with-canvas)
- [Part 4: The OIDC Launch Flow Step by Step](#part-4-the-oidc-launch-flow-step-by-step)
- [Part 5: LTI Advantage Services](#part-5-lti-advantage-services)
  - [Assignment and Grade Services (AGS)](#assignment-and-grade-services-ags)
  - [Names and Role Provisioning Services (NRPS)](#names-and-role-provisioning-services-nrps)
  - [Deep Linking 2.0](#deep-linking-20)
- [Part 6: Canvas Placements](#part-6-canvas-placements)
- [Part 7: JSON Configuration (Paste JSON Method)](#part-7-json-configuration-paste-json-method)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Checklist Before Going to Production](#checklist-before-going-to-production)

---

## How Canvas LTI 1.3 Works

Canvas uses **Developer Keys** to store LTI tool configuration. One Developer Key = one tool registration. A single key can be deployed to multiple accounts, sub-accounts, or courses — each deployment gets its own **Deployment ID**.

The security model is built on:
- **OpenID Connect (OIDC)** — handles the 3-step login initiation flow
- **JSON Web Tokens (JWT)** — the LTI launch payload, signed by Canvas's private key
- **OAuth 2.0 Client Credentials** — used by your tool to call LTI Advantage Services

Canvas acts as the **Platform** (formerly Tool Consumer). Your tool acts as the **Tool** (formerly Tool Provider).

---

## What You Need Before You Start

From your tool, gather these values before touching Canvas:

| Value | Description | Example |
|---|---|---|
| **OIDC Login URL** | Where Canvas sends the initial login request | `https://yourtool.com/lti/oidc/login` |
| **Target Link URI** | Where the tool launches after login | `https://yourtool.com/lti/launch` |
| **Redirect URI(s)** | Must exactly match what your tool sends back | `https://yourtool.com/lti/launch` |
| **Public JWK URL** | Your tool's public key endpoint | `https://yourtool.com/lti/.well-known/jwks.json` |
| **Deep Link Return URL** | Where Canvas sends deep link responses (if using DL) | `https://yourtool.com/lti/deeplink/response` |

> **Important:** Redirect URIs in Canvas must be an **exact string match** — no trailing slashes, no case differences. This is the #1 cause of OIDC launch failures.

---

## Canvas-Specific URLs

Your tool needs these Canvas URLs hard-coded by environment:

| Environment | OIDC Auth URL | JWKS URL | Access Token URL |
|---|---|---|---|
| **Production** | `https://sso.canvaslms.com/api/lti/authorize_redirect` | `https://sso.canvaslms.com/api/lti/security/jwks` | `https://sso.canvaslms.com/login/oauth2/token` |
| **Beta** | `https://sso.beta.canvaslms.com/api/lti/authorize_redirect` | `https://sso.beta.canvaslms.com/api/lti/security/jwks` | `https://sso.beta.canvaslms.com/login/oauth2/token` |
| **Test** | `https://sso.test.canvaslms.com/api/lti/authorize_redirect` | `https://sso.test.canvaslms.com/api/lti/security/jwks` | `https://sso.test.canvaslms.com/login/oauth2/token` |

> **Note:** The issuer (`iss`) for all Canvas environments is `https://canvas.instructure.com`. For self-hosted Canvas, the issuer is your Canvas root URL.

---

## Part 1: Create a Developer Key in Canvas

> **Required role:** Canvas Site Admin or Account Admin with Developer Keys permission.

### Step 1 — Navigate to Developer Keys

1. Log in to Canvas as an administrator
2. Click **Admin** in the left sidebar
3. Select your institution/account name
4. Click **Developer Keys** in the left menu

### Step 2 — Create a new LTI Key

1. Click **+ Developer Key** (top right)
2. Select **+ LTI Key** from the dropdown

### Step 3 — Choose your configuration method

Canvas offers three methods. Use **Manual Entry** when building a new integration:

| Method | When to use |
|---|---|
| **Manual Entry** | Building a new tool, full control over every field |
| **Paste JSON** | You have a pre-built JSON config from your tool vendor |
| **Enter URL** | Your tool hosts its own config JSON at a URL |

### Step 4 — Fill in Key Settings (Manual Entry)

**Key Name:** Something descriptive, e.g. `My LTI Tool - Production`

**Redirect URIs:** One per line, must exactly match what your tool sends:
```
https://yourtool.com/lti/launch
https://yourtool.com/lti/deeplink/response
```

**Method:** Manual Entry

| Field | Value |
|---|---|
| Title | Display name for the tool |
| Description | Brief description |
| Target Link URI | `https://yourtool.com/lti/launch` |
| OpenID Connect Initiation URL | `https://yourtool.com/lti/oidc/login` |
| JWK Method | Select **Public JWK URL** |
| Public JWK URL | `https://yourtool.com/lti/.well-known/jwks.json` |

### Step 5 — Configure LTI Advantage Services

Scroll to **LTI Advantage Services** and enable what your tool needs:

| Service | Toggle | Required for |
|---|---|---|
| Can create and view assignment data in the gradebook | ✅ | AGS — creating line items |
| Can view assignment data in the gradebook | ✅ | AGS — reading grades |
| Can edit grades in the gradebook | ✅ | AGS — writing scores |
| Can retrieve user data associated with context | ✅ | NRPS — course roster |
| Can update public JWK for LTI services | Optional | Key rotation |

### Step 6 — Additional Settings

| Setting | Recommended value |
|---|---|
| Privacy Level | **Public** (required for user data in JWT) |
| Domain | Your tool's domain, e.g. `yourtool.com` |
| Custom Fields | Add any custom parameters your tool needs |

### Step 7 — Configure Placements

Select which Canvas UI locations your tool appears in. See [Part 6](#part-6-canvas-placements) for full details.

### Step 8 — Save and Enable

1. Click **Save**
2. Find your new key in the list
3. Toggle the state from **Off** to **On**
4. Copy the **Client ID** from the Details column (a 17-digit number) — you will need this

> **Critical:** Copy the number under **Details**, not the value shown when you click **Show Key**. These are different values and using the wrong one is a common mistake.

---

## Part 2: Deploy the Tool to an Account or Course

The Developer Key creates the registration. Deployment makes it available to users.

### Deploy to an Account (recommended for institution-wide tools)

1. Go to **Admin → Settings → Apps**
2. Click **View App Configurations**
3. Click **+ App**
4. Set **Configuration Type** to **By Client ID**
5. Paste the Client ID from Step 8 above
6. Click **Submit** → **Install**

### Get the Deployment ID

After installing:
1. Find the tool in the **External Apps** list
2. Click the gear icon ⚙️
3. Select **Deployment ID**
4. Copy this value — your tool needs it

### Deploy to a specific Course

1. Go to the course → **Settings → Apps**
2. Follow the same steps as above
3. The tool will only be available in that course

---

## Part 3: Configure Your Tool to Work with Canvas

After Canvas is configured, give your tool these values:

```javascript
const canvasConfig = {
  issuer: "https://canvas.instructure.com",          // Same for all Canvas instances
  clientId: "YOUR_17_DIGIT_CLIENT_ID",
  deploymentId: "YOUR_DEPLOYMENT_ID",
  
  // Canvas endpoints
  oidcAuthUrl: "https://sso.canvaslms.com/api/lti/authorize_redirect",
  jwksUrl: "https://sso.canvaslms.com/api/lti/security/jwks",
  accessTokenUrl: "https://sso.canvaslms.com/login/oauth2/token",
  
  // Your tool endpoints
  oidcLoginUrl: "https://yourtool.com/lti/oidc/login",
  launchUrl: "https://yourtool.com/lti/launch",
  publicJwksUrl: "https://yourtool.com/lti/.well-known/jwks.json",
};
```

---

## Part 4: The OIDC Launch Flow Step by Step

Understanding this flow is essential for debugging launch failures.

```
Canvas (Platform)                          Your Tool
      |                                        |
      |  1. OIDC Login Request                 |
      |  POST /lti/oidc/login                  |
      |  {iss, login_hint, target_link_uri,    |
      |   client_id, lti_deployment_id}        |
      |--------------------------------------> |
      |                                        |
      |  2. Authentication Request             |
      |  GET /api/lti/authorize_redirect       |
      |  {scope, response_type, client_id,     |
      |   redirect_uri, state, nonce}          |
      | <--------------------------------------|
      |                                        |
      |  3. Authentication Response            |
      |  POST /lti/launch (redirect_uri)       |
      |  {id_token: <signed JWT>, state}       |
      |--------------------------------------> |
      |                                        |
      |  Tool validates JWT signature          |
      |  using Canvas JWKS URL                 |
      |  Tool validates state matches nonce    |
      |  Tool renders content                  |
```

### What your tool must do at each step:

**Step 1 — Receive the login request**
- Extract `iss`, `login_hint`, `client_id`, `lti_deployment_id`, `target_link_uri`
- Generate a unique `state` and `nonce`
- Store them in a session or cookie (needed in Step 3)
- Redirect to Canvas's OIDC auth URL

**Step 2 — Send the authentication request**
```
GET https://sso.canvaslms.com/api/lti/authorize_redirect
  ?scope=openid
  &response_type=id_token
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=https://yourtool.com/lti/launch
  &login_hint=ECHO_BACK_FROM_STEP_1
  &lti_message_hint=ECHO_BACK_FROM_STEP_1
  &state=YOUR_RANDOM_STATE
  &nonce=YOUR_RANDOM_NONCE
  &response_mode=form_post
  &prompt=none
```

**Step 3 — Receive and validate the JWT**
1. Receive `id_token` (signed JWT) via POST
2. Fetch Canvas's JWKS from `https://sso.canvaslms.com/api/lti/security/jwks`
3. Validate JWT signature using the matching public key
4. Validate `nonce` matches what you sent in Step 2
5. Validate `state` matches what you stored in Step 1
6. Validate `iss` = `https://canvas.instructure.com`
7. Validate `aud` contains your `client_id`
8. Check token is not expired (`exp` claim)

> **Safari note:** Safari blocks third-party cookies inside iframes, which breaks the state/nonce verification in Step 3. Canvas implements the LTI Platform Storage spec to work around this. If you're seeing launch failures specifically in Safari, this is why.

---

## Part 5: LTI Advantage Services

### Assignment and Grade Services (AGS)

AGS allows your tool to create gradebook line items and post scores back to Canvas.

#### Getting the AGS endpoint from the JWT

After a successful launch, the JWT contains:
```json
{
  "https://purl.imsglobal.org/spec/lti-ags/claim/endpoint": {
    "scope": [
      "https://purl.imsglobal.org/spec/lti-ags/scope/lineitem",
      "https://purl.imsglobal.org/spec/lti-ags/scope/result.readonly",
      "https://purl.imsglobal.org/spec/lti-ags/scope/score"
    ],
    "lineitems": "https://canvas.instructure.com/api/lti/courses/123/line_items",
    "lineitem": "https://canvas.instructure.com/api/lti/courses/123/line_items/456"
  }
}
```

#### Getting an access token

```javascript
// POST to Canvas access token URL
const tokenResponse = await fetch("https://sso.canvaslms.com/login/oauth2/token", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({
    grant_type: "client_credentials",
    client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    client_assertion: YOUR_SIGNED_JWT,  // signed with your private key
    scope: "https://purl.imsglobal.org/spec/lti-ags/scope/score"
  })
});

const { access_token } = await tokenResponse.json();
```

#### Posting a score

```javascript
// POST to the lineitem score URL
await fetch(`${lineitemUrl}/scores`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${access_token}`,
    "Content-Type": "application/vnd.ims.lis.v1.score+json"
  },
  body: JSON.stringify({
    userId: "canvas_user_id_from_jwt",
    scoreGiven: 85,
    scoreMaximum: 100,
    activityProgress: "Completed",
    gradingProgress: "FullyGraded",
    timestamp: new Date().toISOString()
  })
});
```

---

### Names and Role Provisioning Services (NRPS)

NRPS allows your tool to retrieve the course roster from Canvas.

#### Getting the NRPS endpoint from the JWT

```json
{
  "https://purl.imsglobal.org/spec/lti-nrps/claim/namesroleservice": {
    "context_memberships_url": "https://canvas.instructure.com/api/lti/courses/123/names_and_roles",
    "service_versions": ["2.0"]
  }
}
```

#### Fetching the roster

```javascript
const roster = await fetch(context_memberships_url, {
  headers: {
    "Authorization": `Bearer ${access_token}`,
    "Accept": "application/vnd.ims.lti-nrps.v2.membershipcontainer+json"
  }
});

const { members } = await roster.json();
// members[n].roles contains the user's LTI roles
// members[n].user_id is the Canvas LTI user ID
```

---

### Deep Linking 2.0

Deep Linking allows your tool to return content items back to Canvas — useful for selecting assignments, resources, or embedding content.

#### Detect a Deep Linking launch

Check the `message_type` in the JWT:
```json
{
  "https://purl.imsglobal.org/spec/lti/claim/message_type": "LtiDeepLinkingRequest"
}
```

The JWT also contains:
```json
{
  "https://purl.imsglobal.org/spec/lti-dl/claim/deep_linking_settings": {
    "deep_link_return_url": "https://canvas.instructure.com/courses/123/deep_linking_response",
    "accept_types": ["ltiResourceLink", "link", "file", "html"],
    "accept_presentation_document_targets": ["iframe", "window", "embed"],
    "accept_multiple": true
  }
}
```

#### Return content items to Canvas

```javascript
// Build the JWT response
const responseJwt = signJwt({
  iss: YOUR_CLIENT_ID,
  aud: "https://canvas.instructure.com",
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + 600,
  nonce: generateNonce(),
  "https://purl.imsglobal.org/spec/lti/claim/message_type": "LtiDeepLinkingResponse",
  "https://purl.imsglobal.org/spec/lti/claim/version": "1.3.0",
  "https://purl.imsglobal.org/spec/lti/claim/deployment_id": YOUR_DEPLOYMENT_ID,
  "https://purl.imsglobal.org/spec/lti-dl/claim/content_items": [
    {
      type: "ltiResourceLink",
      title: "My Assignment",
      url: "https://yourtool.com/lti/launch?resource=assignment_123",
      lineItem: {
        scoreMaximum: 100,
        label: "My Assignment",
        resourceId: "assignment_123",
        tag: "grade"
      }
    }
  ]
});

// POST back to the deep_link_return_url
// This must be a form POST from the browser — not a server-to-server call
```

> **Canvas-specific note:** Canvas creates a gradebook column automatically when a Deep Linking response includes a `lineItem` object. This is how most LTI assignment tools add to the Canvas gradebook without an instructor manually creating an assignment first.

---

## Part 6: Canvas Placements

Placements control where in the Canvas UI your tool appears. Configure these in the Developer Key under **Placements**.

| Placement | Where it appears | Message type | Common use |
|---|---|---|---|
| `course_navigation` | Left sidebar in a course | `LtiResourceLinkRequest` | Main tool entry point |
| `assignment_selection` | When creating an assignment | `LtiDeepLinkingRequest` | Select tool-hosted assignments |
| `link_selection` | Rich Content Editor | `LtiDeepLinkingRequest` | Embed content in pages/discussions |
| `module_index_menu_modal` | Module page + button | `LtiDeepLinkingRequest` | Add tool content to modules |
| `editor_button` | RCE toolbar button | `LtiDeepLinkingRequest` | Insert content while editing |
| `account_navigation` | Admin account sidebar | `LtiResourceLinkRequest` | Admin-only tools |
| `user_navigation` | User profile menu | `LtiResourceLinkRequest` | User-level tools |
| `submission_type_selection` | Assignment submission types | `LtiDeepLinkingRequest` | Custom submission types |

Each placement can have its own `target_link_uri` — useful if your tool has a separate Deep Linking endpoint.

---

## Part 7: JSON Configuration (Paste JSON Method)

If you prefer to configure via JSON rather than the Canvas UI, here is the full configuration template:

```json
{
  "title": "Your Tool Name",
  "description": "Brief description of your tool",
  "oidc_initiation_url": "https://yourtool.com/lti/oidc/login",
  "target_link_uri": "https://yourtool.com/lti/launch",
  "scopes": [
    "https://purl.imsglobal.org/spec/lti-ags/scope/lineitem",
    "https://purl.imsglobal.org/spec/lti-ags/scope/lineitem.readonly",
    "https://purl.imsglobal.org/spec/lti-ags/scope/result.readonly",
    "https://purl.imsglobal.org/spec/lti-ags/scope/score",
    "https://purl.imsglobal.org/spec/lti-nrps/scope/contextmembership.readonly"
  ],
  "extensions": [
    {
      "domain": "yourtool.com",
      "tool_id": "your-tool-id",
      "platform": "canvas.instructure.com",
      "privacy_level": "public",
      "settings": {
        "text": "Launch Your Tool",
        "icon_url": "https://yourtool.com/icon.png",
        "selection_height": 800,
        "selection_width": 1000,
        "placements": [
          {
            "placement": "course_navigation",
            "message_type": "LtiResourceLinkRequest",
            "target_link_uri": "https://yourtool.com/lti/launch",
            "text": "Your Tool",
            "icon_url": "https://yourtool.com/icon.png"
          },
          {
            "placement": "assignment_selection",
            "message_type": "LtiDeepLinkingRequest",
            "target_link_uri": "https://yourtool.com/lti/deeplink",
            "text": "Select from Your Tool"
          }
        ]
      }
    }
  ],
  "public_jwk_url": "https://yourtool.com/lti/.well-known/jwks.json",
  "custom_fields": {
    "canvas_course_id": "$Canvas.course.id",
    "canvas_user_id": "$Canvas.user.id",
    "canvas_user_login_id": "$Canvas.user.loginId"
  }
}
```

### Useful Canvas custom variable substitutions

| Variable | Value returned |
|---|---|
| `$Canvas.course.id` | Numeric Canvas course ID |
| `$Canvas.user.id` | Numeric Canvas user ID |
| `$Canvas.user.loginId` | User's Canvas login (usually email) |
| `$Canvas.assignment.id` | Assignment ID (if launched from assignment) |
| `$Canvas.api.domain` | The Canvas instance domain |
| `$Canvas.environment.name` | `production`, `beta`, or `test` |
| `$Person.email.primary` | User's primary email |
| `$CourseSection.sourcedId` | SIS section ID |

---

## Troubleshooting Common Issues

### Launch fails with "redirect_uri mismatch"
The `redirect_uri` your tool sends in Step 2 does not exactly match any URI in the Developer Key. Check for trailing slashes, `http` vs `https`, and extra query parameters.

### JWT signature validation fails
Your tool is fetching the wrong JWKS URL. Make sure you're using the environment-specific Canvas JWKS URL and not your own tool's JWKS URL to validate incoming JWTs.

### "Invalid nonce" error
Your tool is not correctly storing and comparing the nonce between Step 1 and Step 3. The nonce must be stored server-side or in a secure cookie before redirecting to Canvas.

### Safari blank screen / launch fails only in Safari
Third-party cookie blocking. Implement the LTI Platform Storage spec, or configure your tool to use the `lti_storage_target` parameter from Canvas.

### AGS score POST returns 401
Your access token scope does not include the AGS score scope. Ensure `https://purl.imsglobal.org/spec/lti-ags/scope/score` is both enabled in the Developer Key and requested in your `client_credentials` token request.

### Deep Linking response not creating an assignment
Your Deep Linking JWT response is missing the `lineItem` object, or the `accept_types` in the deep linking settings did not include `ltiResourceLink`. Check the original JWT from Canvas for what types are accepted.

### Tool not appearing in course navigation
The Developer Key state is **Off**, or the tool has not been deployed to the account/course. Check both: the key state in Developer Keys and the App Configuration in Settings.

### Deployment ID mismatch
A single Developer Key can have multiple deployments. If your tool validates the `deployment_id` claim in the JWT, make sure you've stored the correct Deployment ID for each Canvas instance/course.

---

## Checklist Before Going to Production

### Canvas side
- [ ] Developer Key state is **On**
- [ ] Tool is deployed to the correct account or course
- [ ] All redirect URIs are HTTPS
- [ ] Required LTI Advantage Services are enabled
- [ ] Privacy Level is set to **Public**
- [ ] Placements are configured correctly

### Tool side
- [ ] JWKS endpoint is publicly accessible (Canvas must be able to fetch your public keys)
- [ ] Tool validates JWT signature using Canvas's JWKS URL
- [ ] Tool validates `iss`, `aud`, `exp`, `nonce`, and `state`
- [ ] Nonce is stored and compared correctly
- [ ] Access token requests use correct scope(s)
- [ ] AGS scores include all required fields (`activityProgress`, `gradingProgress`, `timestamp`)
- [ ] Deep Linking responses are signed and POSTed back to `deep_link_return_url`
- [ ] Tool tested in Canvas Beta before Production

---

## Related Repositories

- [lti13-multilms-tool](https://github.com/muhammadahsan-tech/lti13-multilms-tool) — Working LTI 1.3 tool provider with Canvas, Moodle, and Blackboard support
- [awesome-lti13](https://github.com/muhammadahsan-tech/awesome-lti13) — Curated list of LTI 1.3 libraries, tools, and resources

---

## About

Maintained by **Muhammad Ahsan**, Lead Software Engineer and LTI 1.3 architect at Perdoceo Education Corporation. Muhammad has implemented LTI 1.3 from both sides — as a platform architect enabling a proprietary LMS to accept third-party LTI tools, and as a tool provider developer building integrations for Canvas, Moodle, and Blackboard.

Available for LTI 1.3 consulting and architecture. [Connect on GitHub →](https://github.com/muhammadahsan-tech)

---

*Found an error or missing something? Pull requests welcome.*

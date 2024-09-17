# Active Mode API

A proposal to extend FedCM to handle logged-out users more gracefully. 

The first part of the proposal handles a `active mode` (as opposed to / in addition to the current `passive mode`), where the browser needs to handle more gracefully when users are logged out of IdPs (take the user to login to the IdP, as opposed to failing silently), as Mozilla pointed out [here](https://github.com/w3c-fedid/active-mode/issues/2). 

The second part allows users to `use other accounts` in the account chooser, for example, when IdPs support multiple accounts or replacing the existing account.

<img width="1031" src="https://github.com/user-attachments/assets/7a0c6b1f-b793-4712-8e5e-98a07a1ff591">

## Stage

This is a [Stage 1](https://github.com/w3c-fedid/Administration/blob/main/proposals-CG-WG.md) proposal.

## Champions

- @yi-gu 

## Participate
- https://github.com/w3c-fedid/active-mode

# Background

The active mode flow differs from the passive flow in several ways. The most significant difference is that the active flow is on the critical path of a user's sign-in journey. This means that a user must be able to successfully sign in with a federated account using this flow. In contrast, the passive flow is an optimized flow that can reduce sign-in friction. This means that if the passive flow is unavailable, a user can still click a "Sign in with IdP" button to continue.

In view of the complexity of the critical sign-in flow, as well as the diversity of deployment environments (e.g., an RP may or may not use an IdP SDK), it is necessary to explicitly define the responsibilities of all parties involved.

- What IdP controls
  - Whether a user can sign in to the IdP if they don’t have an active IdP session
  - Whether a user can sign in with new IdP accounts even if they have active IdP sessions 
- What RP controls
  - Note: we consider something RP controlled if it’s defined on the RP site even if an IdP can do it directly with SDK.
  - Whether to use the FedCM active flow and/or passive flow when user tries to sign in
- What browser controls
  - Whether to gate the active flow behind a user gesture
  - Whether to make the active flow more prominent such as controlling the UI’s modality and/or position
  - How to handle the coexistence of both flows
  - Whether to let IdP know which flow was triggered in the ID assertion endpoint

# Proposal

## IdP control

```javascript
// IdP's config file
{
  "accounts_endpoint" : ...,
  "modes: {
    "active": {
      "supports_use_other_account": true|false,
    },
    "passive": {
      "supports_use_other_account": true|false,
    }
  }
}
```

## RP control

```javascript
await navigator.credentials.get({
  identity: {
    providers: [
      {
        configURL: "https://idp.example/fedcm.json",
        clientId: "client1234",      
      },
    ],
    mode: "active"|"passive",
  },
});
```

## Browser control

Include `mode` in the POST request such that IdP can know which flow the user has gone through.

```http
POST /fedcm_assertion_endpoint HTTP/1.1
Host: idp.example
Origin: https://rp.example/
Content-Type: application/x-www-form-urlencoded
Cookie: 0x23223
Sec-Fetch-Dest: webidentity

account_id=123&client_id=client1234&nonce=Ct0D&disclosure_text_shown=true&is_auto_selected=false&**mode=active**
```

## Overview

# Open questions

## Mode options

The current option "button" and "widget" are not ideal because they are focused on the UI affordance instead of the essential distinction of the two types of flows. What "button flow" implies is that it's user initiated and the user must be served by the UA to complete the sign-in flow. This is a critical user journey so the browser should mediate it in a prominent way. What "widget flow" implies is that it may not be gated by user activation therefore it should be treated as an optimized sign-in option and it's OK to fail silently. In other words, it's up to the UA how to implement these two flows with different UI affordances. e.g. one may implement the optional flow with quieter UI like a chip on the URL bar.

We will try to gather more feedback on the naming before shipping.

> [done](https://github.com/w3c-fedid/active-mode/issues/5).

## Feature detection

There's no dedicated way to feature detect dictionary members. There have been [discussions](https://github.com/whatwg/webidl/issues/107) over the years but folks haven't been sufficiently aligned on the proposals. In the interim,  developers can use the following way for feature detection for better interoperability.

```javascript
var supportsFedCmMode = false;
try {
   navigator.credentials.get({
       identity:
     Object.defineProperty(
     {}, 'mode', {
       get: function () { supportsFedCmMode = true; }
     }
   )
   });
} catch(e) {}
```

We can revisit this decision if the community has other thoughts or better options.

# Security Considerations

The active mode shares most of the security properties from the [passive mode](https://fedidcg.github.io/FedCM/#security). e.g. honoring CSP, CORS, using security headers, not asking users to type in the browser UI etc.. It’s worth noting that the pop-up window has the same web platform properties as what one would get with window.open(url,””,”popup,noopener,noreferrer”)) that loads the login_url. There's no communication between the website and this pop-up is allowed (e.g. no postMessage, no window.opener).

# Privacy Considerations

The active mode shares all the privacy properties from the [passive mode](https://fedidcg.github.io/FedCM/#privacy). There are some notable differences:
- The active mode should be gated by user activation.
- For the passive flow which is not gated by user gesture, we do not send requests to IdP if the IdP’s login status is “logged-out”. We propose to do the same for the active flow. However, we should trigger the sign in to IdP flow instead of failing silently because this is on the critical sign-in path.
- The mismatch UI from LoginStatus API is replaced by a louder pop-up window in cases where a silent timing attack could happen.
- The new sign in to IdP affordance has a higher privacy bar because it does not append any parameters to the `login_url`

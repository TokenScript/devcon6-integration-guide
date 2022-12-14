
# Technology Overview

## Ticket & Identifier attestations

Devcon6 permissionless perks builds on top of several key technologies developed by Smart Token Labs. 
At it’s core, it utilizes ticket attestations. These are cryptographically signed tickets issued by Devcon. 
A ticket attestation is issued towards a specific personal identifier like an email address or twitter handle, 
and is usually delivered to the user through a magic link. In the case of Devcon, clicking the magic link will take 
you to the DevCon6 magic link site, where your ticket is saved in local storage and can be used to access 
NFT minting & a list of permissionless perks.

You can think of a ticket attestation as an off-chain token that is bound to your identity and contains metadata 
about the token.

Each ticket is uniquely identifiable via a ticket & conference ID, but third parties cannot learn the personal 
identifier used in the ticket. Instead, our identifier attestation service [Attestation.id](https://attestation.id) 
and some cryptographic magic is used by the user to prove:

1. They have a valid ticket
2. They own the email address (or other identifier) that was issued the ticket


## Verifying Ownership

During the verification process, if an identifier attestation isn’t generated yet, the user will be redirected to 
Attestation.id to confirm their email address & create one. Once the attestation is issued, it is used together with 
the ticket to produce a ZK proof that the ticket & identifier attestations are issued towards the same identifier. 
In simpler terms, it verifies that the user is the valid ticket holder. The result can be verified by a third party (you) 
through the use of a smart contract or server. Like the ticket, the identifier attestation is stored in local storage 
and expires after a week. Users will only need to perform this step once a week unless they move to a new browser or 
device. 


# Integration Guide

Integration with permissionless perks is a two step process. On the front-end, tickets are discovered from Devcon and the attestation is generated through the process explained above. On the back-end or in a smart contract, the following are checked to validate the ticket:

1. Ticket issuer public key is used to validate the ticket.
2. [Attestation.id](http://Attestation.id) public key is used to validate the identifier attestation.
3. The ZK proof is verified to ensure the user is the valid ticket holder.

Okay so now that your familiar with the process let’s get started with the integration.


## Front-end Integration

Token negotiator is part of STL’s Brand Connector product suite and can be used to discover & verify both on and off 
chain tokens. On the front end we will be using token-negotiator to discover tickets and orchestrate the verification 
process.

To get you up and running fast, we have provided 2 examples of Token Negotiator integration in this repo:

1. npm integration into a TypeScript project.
2. Javascript UMD bundle integration in pure HTML & Javascript.

### Running the examples

To run the TypeScript example:
```shell
$ cd frontend/npm-typescript
npm-typescript$ npm i
npm-typescript$ npm run start
```

To run the UMD example, serve the files inside the frontend/pure-javascript folder from your 
preferred webserver and open it in your browser. If you don't have a preferred webserver you 
can run it using the http-server NPM package

```shell
$ npx http-server frontend/pure-javascript -p 8085
```

### Integrating using NPM

If you are already using NPM, you can install the token-negotiator module directly with NPM:
```shell
$ npm i @tokenscript/token-negotiator
```

Token negotiator has some dependencies that require node.js polyfills for webpack.
Install the polyfill plugin:
```shell
$ npm i node-polyfill-webpack-plugin
```
And add the plugin to your webpack config:

```js
const NodePolyfillPlugin = require("node-polyfill-webpack-plugin");
...
module.exports = {
  plugins: 
    ...
    new NodePolyfillPlugin()  
  ]
  ...
}
```

Once you have installed TN, you can use the code at src/index.ts for invoking authentication. 
Copy the code into your own application & modify it as required.

Note: If you are using rollup or create-react-app, we have some examples of how to include polyfills 
over at the [Token Negotiator Examples](https://github.com/TokenScript/token-negotiator-examples) repo.

### Integrating using Javascript bundle

To integrate using the Javascript bundle, simply copy  
frontend/pure-javascript/assets/negotiator into your own project and include the JS & CSS in 
the HTML page or template of your choosing.

```html
<script src="assets/negotiator/negotiator.js"></script>
<link rel="stylesheet" href="assets/negotiator/theme/style.css">
```

TN will now be accessible via window.negotiator. Copy the code at assets/main.js and modify it 
as necessary for your application.

## Server verification

The attestation can be verified in two ways:

1. Use our attestation verification API service by sending a HTTP post request with the attestation:

```http request
POST https://attestation-verify.tokenscript.org/verify-useticket-attestation
Content-Type: application/json

{
    "useTicket": "MIIDVTCBkz...",
    "un": {
        "address": "0x0...",
        "domain": "some-domain.com",
        "expiry": 1664340935557,
        "UN": "123...",
        "randomness": "123...",
        "signature": "0x0..."
    }
}
```

Note: UN parameter is optional

2. Implement your own verification using our NPM or Java attestation libraries

### Integrate via your own node.js service

You can install our JS attestation library to an existing project via NPM:

```shell
$ npm i @tokenscript/attestation
```

Once you have installed the library, see server/src/index.ts for an example of verification

#### Running the server example

To run the server example, use the following commands:

```shell
$ cd server
npm-typescript$ npm i
npm-typescript$ npm run start
```

The server will run on port 8089, but this can be overridden using the PORT environment variable. 

### Integrate via your own Java service

Please contact us if you are interested in using our attestation.jar library for server-side verification.

## Solidity Verification

The ethereum folder in this Repo has an example of validation in Solidity. The example allows minting an NFT using a ticket attestation.
Token ID can be based on the ID of the ticket attestation (to limit minting to 1 per ticket) or a tokenId that you generate.

```shell
$ npm i
$ npm run test
```

To use ticket validation in your own contract, copy the VerifyAttestation.sol library to your project.

## UN (unpredictable number) verification

The UN challenge/signature provides an extra layer of verification for third party sites that want it. 
Attestations are valid for a week and can be used across different third-party site. 
UN verification allows the third party site to confirm that the attestation hasn't been compromised or shared by the user. 
This comes at the expense of UX, as the user is required to sign a challenge on the third-party site. 

### To turn off UN signing

Update the tokenConfig before instantiating the token-negotiator client:

`tokenConfig.unEndPoint = null`

# Simplified Verification

Signed ticket verification is offered as a way to do lightweight verification that does not require an 
email attestation to be generated. To use this method you need to be whitelisted by SmartTokenLabs. 
Once you have been whitelisted, you will receive a signedToken value in the token data:

```javascript
negotiator.on("tokens-selected", (tokens) => {
	console.log(tokens.selectedTokens['devcon'][0]);
});
```

```javascript
{
	ticketId: "123",
	signedToken: "MIGTME..."
    ...
}
```

The signedToken can easily be validated by using our API or implementing your own verification.

To verify using our API, send a HTTP POST request like so:
```http request
POST https://attestation-verify.tokenscript.org/verify-ticket
Content-Type: application/json

{
    "ticket": "MIIDVTCBkz...",
}
```

To implement your own verification, follow the example in server/src/index.ts (/verify-ticket endpoint).

To request this verification method, please contact SmartTokenLabs and provide a demo of your integration by emailing [sayhi@smarttokenlabs.com](mailto:sayhi@smarttokenlabs.com). 

# Configure for production

You may have noticed that the tokenConfig.json file is used to configure an off-chain token that you can test these examples with.
You will need to use a different config in you production environment to detect actual devcon 6 tickets.

The most up-to-date config is available at the following URL:

`https://devcon-vi.attest.tickets/devcon6.js?v=0.25`

We recommend that you load this config in a script tag, so you can stay up to date with any last minute changes:

```html
<script type="text/javascript" src="https://devcon-vi.attest.tickets/devcon6.js?v=0.25"></script>
```

```javascript
const devconConfig = window.devconTokenConfig
```

You can find the full code commented out in the frontend examples.
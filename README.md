# NKN SDK Documentation
This documentation includes the JavaScript, Java, and Go SDKs implimentations for NKN.

So let's start off with the JavaScript.

# NKN SDK JS
JavaScript implementation of NKN SDK. The SDK consists of three components:

1. [NKN Client](): Send and receive data for free between any NKN clients regardless their network condition without setting up a server or relying on any third party services. Data are end to end encrypted by default. Typically you might want to use [multiclient]() instead of using client directly.
2. [NKN MultiClient](): Send and receive data using multiple NKN clients concurrently to improve reliability and latency. In addition, it supports session mode, a reliable streaming protocol similar to TCP based on ncp.
3. [NKN Wallet](): Wallet SDK for [NKN blockchain](https://github.com/nknorg/nkn). It can be used to create wallet, transfer token to NKN wallet address, register name, subscribe to topic, and many other.
Check out more details in the [Documentation](https://docs.nkn.org/nkn-sdk-js)

## Installation
For npm:
```
npm install nkn-sdk
```

And then in your code:
```javascript
const nkn = require('nkn-sdk');
```

or using ES6 import:
```javascript
import nkn from 'nkn-sdk';
```
For browser, use ```dist/nkn.js``` or ```dist/nkn.min.js```.

For environment where cryptographically secure random number generator is not natively implemented (e.g. React Native), see [Random bytes generation]().

## Client

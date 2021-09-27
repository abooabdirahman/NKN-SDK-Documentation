# NKN SDK Documentation
This documentation includes the [JavaScript](), [Go](), and [Java]() SDKs implimentations for NKN.

So let's start off with the JavaScript.

# NKN SDK JS
JavaScript implementation of NKN SDK. The SDK consists of three components:

1.  [NKN MultiClient](): Send and receive data using multiple NKN clients concurrently to improve reliability and latency. In addition, it supports session mode, a reliable streaming protocol similar to TCP based on [ncp](https://github.com/nknorg/ncp-js).
2.  [NKN Client](): Send and receive data for free between any NKN clients regardless their network condition without setting up a server or relying on any third party services. Data are end to end encrypted by default. Typically you might want to use [multiclient]() instead of using client directly.
Check out more details in the [Documentation](https://docs.nkn.org/nkn-sdk-js)
3. [NKN Wallet](): Wallet SDK for [NKN blockchain](https://github.com/nknorg/nkn). It can be used to create wallet, transfer token to NKN wallet address, register name, subscribe to topic, and many other.

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

## MultiClient
MultiClient creates multiple NKN client instances by adding identifier prefix (```__0__.```, ```__1__.```, ```__2__.```, ...) to a NKN address and sends or receives packets concurrently. This will greatly increase reliability and reduce latency at the cost of more bandwidth usage (proportional to the number of clients).

MultiClient basically has the same API as [client](), with a few additional initial configurations and session mode:

```javascript
let multiclient = new nkn.MultiClient({
  numSubClients: 4,
  originalClient: false,
});
```

where ```originalClient``` controls whether a client with original identifier (without adding any additional identifier prefix) will be created, and ```numSubClients``` controls how many sub-clients to create by adding prefix ```__0__.```, ```__1__.```, ```__2__.```, etc. Using ```originalClient: true``` and ```numSubClients: 0``` is equivalent to using a standard NKN Client without any modification to the identifier. Note that if you use ```originalClient: true``` and ```numSubClients``` is greater than 0, your identifier should not starts with ```__X__``` where X is any number, otherwise you may end up with identifier collision.

Any additional options will be passed to NKN client.

MultiClient instance shares most of the public API as regular NKN [client](), see client for usage and examples. If you need low-level property or API, you can use ```multiclient.defaultClient``` to get the default client and ```multiclient.clients``` to get all clients.

And do Check out [complete examples](https://github.com/nknorg/nkn-sdk-js/blob/master/examples/client.js).

Also, check out the [full documentation](https://docs.nkn.org/nkn-sdk-js).


### Session

In addition to the default packet mode, multiclient also supports session mode, a reliable streaming protocol similar to TCP based on [ncp](https://github.com/nknorg/ncp-js).

Listens for incoming sessions (without ```listen()``` no sessions will be accepted):

```multiclient.listen();```

Dial a session:

```javascript
multiclient.dial('another-client-address').then((session) => {
  console.log(session.localAddr, 'dialed a session to', session.remoteAddr);
});
```

Accepts for incoming sessions:
```javascript
multiclient.onSession((session) => {
  console.log(session.localAddr, 'accepted a session from', session.remoteAddr);
});
```

Write to session:

```javascript
session.write(Uint8Array.from([1,2,3,4,5])).then(() => {
  console.log('write success');
});
```

Read from session:

```javascript
session.read().then((data) => {
  console.log('read', data);
});
```

```session.read``` also accepts a ```maxSize``` parameter, e.g. ```session.read(maxSize)```. If ```maxSize > 0```, at most maxSize bytes will be returned. If ```maxSize == 0``` or not set, the first batch of received data will be returned. If ```maxSize < 0```, all received data will be concatenated and returned together.

Session can be converted to WebStream using ```session.getReadableStream()``` and ```session.getWritableStream(closeSessionOnEnd = false)```. Note that WebStream is not fully supported by all browser, so you might need to polyfill it globally or ```setting session.ReadableStream``` and ```session.WritableStream``` constructors.

Check out for [complete example](https://github.com/nknorg/nkn-sdk-js/blob/master/examples/session.js) on this. 

Or try out the demo file transfer [web app](https://nftp.nkn.org).  

And you can peek at its [source code](https://github.com/nknorg/nftp-js) on github.


## Client
NKN client provides the basic functions of sending and receiving data between NKN clients or topics regardless their network condition without setting up a server or relying on any third party services. Typically you might want to use [multiclient]() instead of using client directly.

Create a client with a generated key pair:

```javascript
let client = new nkn.Client();
```

Or with an identifier (used to distinguish different clients sharing the same key pair):

```javascript
let client = new nkn.Client({
  identifier: 'any-string',
});
```

Get client secret seed and public key:

```javascript
console.log(client.getSeed(), client.getPublicKey());
```

Create a client using an existing secret seed:

```javascript
let client = new nkn.Client({
  seed: '2bc5501d131696429264eb7286c44a29dd44dd66834d9471bd8b0eb875a1edb0',
});
```

Secret key should be kept **SECRET!** Never put it in version control system like here.

By default the client will use bootstrap RPC server (for getting node address) provided by nkn.org. Any NKN full node can serve as a bootstrap RPC server. You can create a client using customized bootstrap RPC server:

```javascript
let client = new nkn.Client({
  rpcServerAddr: 'https://ip:port',
});
```

Get client NKN address, which is used to receive data from other clients:

```javascript
console.log(client.addr);
```

Listen for connection established:

```javascript
client.onConnect(() => {
  console.log('Client ready.');
});
```

Send text message to other clients:

```javascript
client.send(
  'another-client-address',
  'hello world!',
);
```

You can also send byte array directly:

```javascript
client.send(
  'another-client-address',
  Uint8Array.from([1,2,3,4,5]),
);
```

The destination address can also be a name registered using [wallet]().

Publish text message to all subscribers of a topic (subscribe is done through [wallet]()):

```javascript
client.publish(
  'topic',
  'hello world!',
);
```

Receive data from other clients:

```javascript
client.onMessage(({ src, payload }) => {
  console.log('Receive message', payload, 'from', src);
});
```

If a valid data (string or Uint8Array) is returned at the end of the handler, the data will be sent back to sender as reply:

```javascript
client.onMessage(({ src, payload }) => {
  return 'Well received!';
});
```

Handler can also be an async function, and reply can be byte array as well:

``` javascript
client.onMessage(async ({ src, payload }) => {
  return Uint8Array.from([1,2,3,4,5]);
});
```

Note that if multiple message handlers are added, the result returned by the first handler (in the order of being added) will be sent as reply.

The ```send``` method will return a Promise that will be resolved when sender receives a reply, or rejected if not receiving reply or acknowledgement within timeout period. Similar to message, reply can be either string or byte array:

``` javascript
client.send(
  'another-client-address',
  'hello world!',
).then((reply) => {
  // The reply here can be either string or Uint8Array
  console.log('Receive reply:', reply);
}).catch((e) => {
  // This will most likely to be timeout
  console.log('Catch:', e);
});
```

Client receiving data will automatically send an acknowledgement back to sender if message handler returns ```null``` or ```undefined``` so that sender will be able to know if the packet has been delivered. On the sender's side, it's almost the same as receiving a reply, except that the Promise is resolved with ```null```:

``` javascript
client.send(
  'another-client-address',
  'hello world!',
).then(() => {
  console.log('Receive ACK');
}).catch((e) => {
  // This will most likely to be timeout
  console.log('Catch:', e);
});
```

Check out for [complete examples](https://github.com/nknorg/nkn-sdk-js/blob/master/examples/client.js) and  [full documentation](https://docs.nkn.org/nkn-sdk-js).

## Wallet

NKN Wallet SDK.

Create a new wallet with a generated key pair:

```javascript
let wallet = new nkn.Wallet({ password: 'password' });
```

Create wallet from a secret seed:

```javascript
let wallet = new nkn.Wallet({
  seed: wallet.getSeed(),
  password: 'new-wallet-password',
});
```

Export wallet to JSON string:

```javascript
let walletJson = wallet.toJSON();
```

Load wallet from JSON and password:

```javascript
let wallet = nkn.Wallet.fromJSON(walletJson, { password: 'password' });
```

By default the wallet will use RPC server provided by nkn.org. Any NKN full node can serve as a RPC server. You can create a wallet using customized RPC server:

```javascript
let wallet = new nkn.Wallet({
  password: 'password',
  rpcServerAddr: 'https://ip:port',
});
```

Verify whether an address is a valid NKN wallet address:

```javascript
console.log(nkn.Wallet.verifyAddress(wallet.address));
```

Verify password of the wallet:

```javascript
console.log(wallet.verifyPassword('password'));
```

Get balance of this wallet:

```javascript
wallet.getBalance().then((value) => {
  console.log('Balance for this wallet is:', value.toString());
});
```

Transfer token to another wallet address:

```javascript
wallet.transferTo(wallet.address, 1, { fee: 0.1, attrs: 'hello world' }).then((txnHash) => {
  console.log('Transfer transaction hash:', txnHash);
});
```

Subscribe to a topic for this wallet for next 100 blocks (around 20 seconds per block), client using the same key pair (seed) as this wallet and same identifier as passed to subscribe will be able to receive messages from this topic:

```javascript
wallet.subscribe('topic', 100, 'identifier', 'metadata', { fee: '0.1' }).then((txnHash) => {
  console.log('Subscribe transaction hash:', txnHash);
});
```

You can also check for [complete examples](https://github.com/nknorg/nkn-sdk-js/blob/master/examples/wallet.js) and  for [full documentation](https://docs.nkn.org/nkn-sdk-js).


## Random bytes generation

By default, this library uses the same random bytes generator as [tweetnacl-js](https://github.com/dchest/tweetnacl-js).

If a platform you are targeting doesn't implement secure random number generator, but you somehow have a cryptographically-strong source of entropy (not ```Math.random```!), and you know what you are doing, you can plug it like this:

```javascript
nkn.setPRNG(function(x, n) {
  // ... copy n random bytes into x ...
});
```

An example using node.js native crypto library:

```javascript
crypto = require('crypto');
nkn.setPRNG(function(x, n) {
  var i, v = crypto.randomBytes(n);
  for (i = 0; i < n; i++) x[i] = v[i];
  // clean up v
});
```

Note that ```setPRNG``` completely _replaces_ internal random byte generator with the one provided.


# NKN SDK for Java/Kotlin/JVM

Go implementation of NKN client and wallet SDK consists three components same as JS above.

## Usage

### Client 

NKN Client provides low level p2p messaging through NKN network. For most applications, it's more suitable to use multiclient (see [multiclient]() section below) for better reliability, lower latency, and session mode support.

Create a client with a generated key pair:

```go
account, err := NewAccount(nil)
client, err := NewClient(account, "", nil)
```

Or with an identifier (used to distinguish different clients sharing the same key pair):

```go
client, err := NewClient(account, "any string", nil)
```

Get client key pair:

```go
fmt.Println(account.Seed(), account.PubKey())
```

Create a client using an existing secret seed:

```go
seed, err := hex.DecodeStrings("039e481266e5a05168c1d834a94db512dbc235877f150c5a3cc1e3903672c673")
account, err := NewAccount(seed)
client, err := NewClient(account, "any string", nil)
```

Secret seed should be kept SECRET! Never put it in version control system like here.

By default the client will use bootstrap RPC server (for getting node address) provided by NKN. Any NKN full node can serve as a bootstrap RPC server. To create a client using customized bootstrap RPC server:

```go
conf := &ClientConfig{SeedRPCServerAddr: NewStringArray("https://ip:port", "https://ip:port", ...)}
client, err := NewClient(account, "any string", conf)
```

Get client NKN address, which is used to receive data from other clients:

```go
fmt.Println(client.Address())
```

Listen for connection established:Listen for connection established:

```go
<- client.OnConnect.C
fmt.Println("Connection opened.")
```

Send text message to other clients:

```go
response, err := client.Send(NewStringArray("another client address"), []byte("hello world!"), nil)
```

You can also send byte array directly:

```go
response, err := client.Send(NewStringArray("another client address"), []byte{1, 2, 3, 4, 5}, nil)
```

Or publish a message to a specified topic (see wallet section for subscribing to topics):

```go
client.Publish("topic", []byte("hello world!"), nil)
```

Receive data from other clients:

```go
msg := <- client.OnMessage.C
fmt.Println("Receive message from", msg.Src + ":", string(msg.Payload))
msg.Reply([]byte("response"))
```

Get 100 subscribers of specified topic starting from 0 offset, including those in tx pool (fetch meta):

```go
subscribers, err := client.GetSubscribers("topic", 0, 100, true, true)
fmt.Println(subscribers.Subscribers, subscribers.SubscribersInTxPool)
```

Get subscription:

```go
subscription, err := client.GetSubscription("topic", "identifier.publickey")
fmt.Printf("%+v\n", subscription) // &{Meta:meta ExpiresAt:100000}
```

## Multiclient

Multiclient creates multiple client instances by adding identifier prefix (```__0__.```, ```__1__.```, ```__2__.```, ...) to a nkn address and send/receive packets concurrently. This will greatly increase reliability and reduce latency at the cost of more bandwidth usage (proportional to the number of clients).

Multiclient basically has the same API as client, except for a few more initial configurations:

```go
numSubClients := 3
originalClient := false
multiclient, err := NewMultiClient(account, identifier, numSubClient, originalClient)
```

where ```originalClient``` controls whether a client with original identifier (without adding any additional identifier prefix) will be created, and ```numSubClients``` controls how many sub-clients to create by adding prefix ```__0__.```, ```__1__.```, ```__2__.```, etc. Using ```originalClient == true``` and ```numSubClients == 0``` is equivalent to using a standard client without any modification to the identifier. Note that if you use ```originalClient == true``` and ```numSubClients``` is greater than 0, your identifier should not starts with ```__X__``` where ```X``` is any number, otherwise you may end up with identifier collision.

Any additional options will be passed to NKN client.

multiclient instance shares the same API as regular NKN client, see above for usage and examples. If you need low-level property or API, you can use ```multiclient.DefaultClient``` to get the default client and ```multiclient.Clients``` to get all clients.

### Session

Multiclient supports a reliable transmit protocol called session. It will be responsible for retransmission and ordering just like TCP. It uses multiple clients to send and receive data in multiple path to achieve better throughput. Unlike regular multiclient message, no redundant data is sent unless packet loss.

Any multiclient can start listening for incoming session where the remote address match any of the given regexp:

```go
multiclient, err := NewMultiClient(...)
// Accepting any address, equivalent to multiclient.Listen(NewStringArray(".*"))
err = multiclient.Listen(nil)
// Only accepting pubkey 25d660916021ab1d182fb6b52d666b47a0f181ed68cf52a056041bdcf4faaf99 but with any identifiers
err = multiclient.Listen(NewStringArray("25d660916021ab1d182fb6b52d666b47a0f181ed68cf52a056041bdcf4faaf99$"))
// Only accepting address alice.25d660916021ab1d182fb6b52d666b47a0f181ed68cf52a056041bdcf4faaf99
err = multiclient.Listen(NewStringArray("^alice\\.25d660916021ab1d182fb6b52d666b47a0f181ed68cf52a056041bdcf4faaf99$"))
```

Then it can start accepting sessions:

```go
session, err := multiclient.Accept()
```

Multiclient implements ```net.Listener``` interface, so one can use it as a drop-in replacement when ```net.Listener``` is needed, e.g. ```http.Serve```.

On the other hand, any multiclient can dial a session to a remote NKN address:


```go
session, err := multiclient.Dial("another nkn address")
```

Session implements ```net.Conn``` interface, so it can be used as a drop-in replacement when ```net.Conn``` is needed:

```go
buf := make([]byte, 1024)
n, err := session.Read(buf)
n, err := session.Write(buf)
```

### Wallet

Create wallet SDK:

```go
account, err := NewAccount(nil)
wallet, err := NewWallet(account, &nkn.WalletConfig{Password: "password"})
```

By default the wallet will use RPC server provided by ```nkn.org```. Any NKN full node can serve as a RPC server. To create a wallet using customized RPC server:

```go
conf := &WalletConfig{
  Password: "password",
  SeedRPCServerAddr: NewStringArray("https://ip:port", "https://ip:port", ...),
}
wallet, err := NewWallet(account, conf)
```

Export wallet to JSON string, where sensitive contents are encrypted by password provided in config:

```go
walletJSON, err := wallet.ToJSON()
```

Load wallet from JSON string, note that the password needs to be the same as the one provided when creating wallet:

```go
walletFromJSON, err := nkn.WalletFromJSON(walletJSON, &nkn.WalletConfig{Password: "password"})
```

Verify whether an address is a valid NKN wallet address:

```go
err := nkn.VerifyWalletAddress(wallet.Address())
```

Verify password of the wallet:

```go
err := wallet.VerifyPassword("password")
```

Query asset balance for this wallet:


```go
balance, err := wallet.Balance()
if err == nil {
    log.Println("asset balance:", balance.String())
} else {
    log.Println("query balance fail:", err)
}
```

Query asset balance for address:

```go
balance, err := wallet.BalanceByAddress("NKNxxxxx")
```

Transfer asset to some address:

```go
txnHash, err := wallet.Transfer(account.WalletAddress(), "100", nil)
```

Open nano pay channel to specified address:

```go
// you can pass channel duration (in unit of blocks) after address and txn fee
// after expired new channel (with new id) will be created under-the-hood
// this means that receiver need to claim old channel and reset amount calculation
np, err := wallet.NewNanoPay(address, "0", 4320)
```

Increment channel balance by 100 NKN:

```go
txn, err := np.IncrementAmount("100")
```

Then you can pass the transaction to receiver, who can send transaction to on-chain later:

```go
txnHash, err := wallet.SendRawTransaction(txn)
```

Register name for this wallet:

```go
txnHash, err = wallet.RegisterName("somename", nil)
```

Delete name for this wallet:

```go
txnHash, err = wallet.DeleteName("somename", nil)
```

Subscribe to specified topic for this wallet for next 100 blocks:

```go
txnHash, err = wallet.Subscribe("identifier", "topic", 100, "meta", nil)
```

Unsubscribe from specified topic:

```go
txnHash, err = wallet.Unsubscribe("identifier", "topic", nil)
```

## Compiling to iOS/Android native library

This library is designed to work with [gomobile](https://godoc.org/golang.org/x/mobile/cmd/gomobile) and run natively on iOS/Android without any modification. You can use ```gomobile bind``` to compile it to Objective-C framework for iOS:

```go
gomobile bind -target=ios -ldflags "-s -w" github.com/nknorg/nkn-sdk-go github.com/nknorg/ncp-go github.com/nknorg/nkn/v2/transaction
```

and Java AAR for Android:

```go
gomobile bind -target=android -ldflags "-s -w" github.com/nknorg/nkn-sdk-go github.com/nknorg/ncp-go github.com/nknorg/nkn/v2/transaction
```

It's recommended to use the latest version of gomobile that supports go modules.

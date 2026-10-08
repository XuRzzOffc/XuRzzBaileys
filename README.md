# <div align='center'>Baileys - Typescript/Javascript WhatsApp Or Telegram Web API XuRzzOffc</div>

<div align='center'>

![GitHub Downloads (all assets, all releases)](https://img.shields.io/github/downloads/whiskeysockets/baileys/total)
![NPM Downloads](https://img.shields.io/npm/dw/%40whiskeysockets%2Fbaileys?label=npm&color=%23CB3837)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/whiskeysockets/baileys)
![GitHub License](https://img.shields.io/github/license/whiskeysockets/baileys)
![Discord](https://img.shields.io/discord/725839806084546610?label=discord&color=%235865F2)
![GitHub Repo stars](https://img.shields.io/github/stars/whiskeysockets/baileys)
![GitHub forks](https://img.shields.io/github/forks/whiskeysockets/baileys)

</div>

<div align="center">

# @XuRzzOffc/XuRzzBails

A WebSocket-based library for interacting with WhatsApp Web — a fork of
[Baileys](https://github.com/WhiskeySockets/Baileys) with additional socket layers
(Communities, Interop, Privacy, GraphQL) and helpers for special message types
such as payments, products, albums, events, poll results, and order messages.

</div>

---

> **Not affiliated with WhatsApp or Meta.** Using this library may violate
> WhatsApp's Terms of Service. Use it at your own risk, and do not use it for spam.

### Important Note

This library was originally a project for **CS-2362 at Ashoka University** and is in no way affiliated with or endorsed by WhatsApp. Use at your own discretion. Do not spam people with this. We discourage any stalkerware, bulk or automated messaging usage.

#### Liability and License Notice
Baileys and its maintainers cannot be held liable for misuse of this application, as stated in the [MIT license](https://github.com/WhiskeySockets/Baileys/blob/master/LICENSE).
The maintainers of Baileys do not in any way condone the use of this application in practices that violate the Terms of Service of WhatsApp. The maintainers of this application call upon the personal responsibility of its users to use this application in a fair way, as it is intended to be used.
##

- Baileys does not require Selenium or any other browser to be interface with WhatsApp Web, it does so directly using a **WebSocket**.
- Not running Selenium or Chromimum saves you like **half a gig** of ram :/
- Baileys supports interacting with the multi-device & web versions of WhatsApp.
- Thank you to [@pokearaujo](https://github.com/pokearaujo/multidevice) for writing his observations on the workings of WhatsApp Multi-Device. Also, thank you to [@Sigalor](https://github.com/sigalor/whatsapp-web-reveng) for writing his observations on the workings of WhatsApp Web and thanks to [@Rhymen](https://github.com/Rhymen/go-whatsapp/) for the __go__ implementation.

> [!IMPORTANT]
> The original repository had to be removed by the original author - we now continue development in this repository here.
This is the only official repository and is maintained by the community.
> **Join the Discord [here](https://discord.gg/WeJM5FP9GG)**

## Table of Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
  - [Login with QR Code](#login-with-qr-code)
  - [Login with Pairing Code](#login-with-pairing-code)
- [Store](#store)
- [Sending Messages](#sending-messages)
  - [Generic Send / Relay](#generic-send--relay)
  - [Simple Senders](#simple-senders)
  - [Special Message Types](#special-message-types)
- [Events](#events)
- [Examples](#examples)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

## Requirements

- Node.js **>= 20**
- Optional peer dependencies depending on the features you use:
  - `sharp` or `jimp` for image processing
  - `link-preview-js` for link previews
  - `audio-decode` for audio waveform handling

## Installation

```bash
npm install @XuRzzOffc/XuRzzBails
```

You can also add it to your package manifest as `baileys` or `@whiskeysockets/baileys`:

```json
{
  "dependencies": {
    "@whiskeysocket/baileys": "github:XuRzzOffc/XuRzzBails"
  }
}
```

```js
import makeWASocket from '@whiskeysocket/baileys'
// or with CommonJS:
const { default: makeWASocket } = require('@whiskeysocket/baileys')
```

## Quick Start

### Login with QR Code

```js
import makeWASocket, { Browsers, useMultiFileAuthState } from '@whiskeysocket/baileys'

const { state, saveCreds } = await useMultiFileAuthState('auth_info')

const client = makeWASocket({
  browser: Browsers.ubuntu('Chrome'),
  printQRInTerminal: true,
  auth: state
})

client.ev.on('creds.update', saveCreds)
```

### Login with Pairing Code

```js
import makeWASocket, { Browsers, fetchLatestWAWebVersion, useMultiFileAuthState } from '@whiskeysocket/baileys'

const { state, saveCreds } = await useMultiFileAuthState('auth_info')
const { version } = await fetchLatestWAWebVersion()

const client = makeWASocket({
  browser: Browsers.ubuntu('Chrome'),
  printQRInTerminal: false,
  version,
  auth: state
})

client.ev.on('creds.update', saveCreds)

if (!client.authState?.creds?.registered) {
  const phoneNumber = '628XXXXXXXXXX' // international format, without '+'
  const code = await client.requestPairingCode(phoneNumber)
  // custom pairing code (8 characters):
  // const code = await client.requestPairingCode(phoneNumber, 'YYYYYYYY')
  console.log('Pairing code:', code)
}
```

## Store

`makeInMemoryStore` creates a local cache for chats, contacts, and messages,
which Baileys does not persist automatically by default.

```js
import makeWASocket, { makeInMemoryStore } from '@whiskeysocket/baileys'
import pino from 'pino'

const store = makeInMemoryStore({
  logger: pino().child({ level: 'silent', stream: 'store' })
})

const client = makeWASocket({ /* ...other options */ })
store.bind(client.ev)

client.ev.on('contacts.upsert', () => {
  console.log('New contact:', Object.values(store.contacts))
})
```

Need a persistent store such as Redis? Use `makeCacheManagerStore` — see the [API Reference](docs/API.md#store-libstore).

### Saving the Store to a File

`makeInMemoryStore` can read from and write to a local JSON file, so it can
survive restarts without requiring an external database.

```js
const store = makeInMemoryStore({ /* ... */ })

// load once on startup, then auto-save every 10 seconds
const stopAutoSave = store.writeToFileInterval('./store.json')

process.on('SIGINT', () => {
  stopAutoSave()
  process.exit(0)
})
```

You can also manage it manually with `store.readFromFile(path)` and `store.writeToFile(path)`.
See the [API Reference](docs/API.md#file-persistence-makeinmemorystore) for more details.

## Sending Messages

### Generic Send / Relay

```js
// relayMessage — sends a raw message object, bypassing the sendMessage pipeline
await client.relayMessage(jid, { conversation: 'Hello from Baileys' }, {})

// sendMessage — the standard way to send a message
await client.sendMessage(jid, { text: 'Hello from Baileys' })
```

### Simple Senders

All `sendX` helpers below are shortcuts built on top of `sendMessage`.

```js
await client.sendText(jid, 'Hi!', { contextInfo: { mentionedJid: [jid] } })
await client.sendImage(jid, { url: './photo.jpg' }, 'image caption')
await client.sendVideo(jid, { url: './clip.mp4' }, 'video caption')
await client.sendAudio(jid, { url: './clip.mp3' })
await client.sendLocation(jid, 'Location name', -6.2, 106.8, 'https://maps.example', '1234567890')
await client.sendPoll(jid, 'Pick one', ['Option 1', 'Option 2', 'Option 3'], /* multiSelect */ true)
await client.sendQuiz(jid, 'Correct answer?', ['1', '2', '3'], /* correctIndex */ '2')
```

### Special Message Types

These are handled internally by the `Socket/luxu.js` helper and are invoked automatically
by `sendMessage` or `relayMessage` whenever the payload contains one of the fields below.

### Rich Menu

`richMenu` sends a rich response with an optional header image, action buttons, carousel cards, and an open-URL footer:

```js
await client.richMenu(jid, {
  header: {
    title: 'Menu',
    image: { url: 'https://example.com/banner.png' }
  },
  body: {
    title: 'Choose an option',
    buttons: ['Profile', 'Help']
  },
  footer: {
    text: 'Open website',
    url: 'https://example.com'
  }
})
```

```js
// Product message (catalog)
await client.relayMessage(jid, {
  productMessage: {
    title: 'Product Name',
    description: 'Product description',
    thumbnail: { url: './product.jpg' },
    productId: 'PRODUCT_ID',
    retailerId: 'RETAILER_ID',
    url: 'https://store.example/product',
    body: 'Body text',
    footer: 'Footer text',
    priceAmount1000: 72502, // price x 1000
    currencyCode: 'IDR'
  }
}, {})

// Order message
await client.sendMessage(jid, {
  thumbnail: fs.readFileSync('./thumb.jpg'),
  message: 'Order details',
  orderTitle: 'Store Name',
  totalAmount1000: 72502,
  totalCurrencyCode: 'IDR'
}, { quoted: m })

// Poll result snapshot (usually from a newsletter)
await client.sendMessage(jid, {
  pollResultMessage: {
    name: 'Poll Title',
    options: [{ optionName: 'Option 1' }, { optionName: 'Option 2' }],
    newsletter: { newsletterName: 'Newsletter Name', newsletterJid: '1234567890@newsletter' }
  }
})

// Interactive message (button)
await client.sendMessage(jid, {
  image: { url: './banner.jpg' },
  text: 'Message body',
  title: 'Title',
  footer: 'Footer',
  interactiveButtons: [{
    name: 'cta_url',
    buttonParamsJson: JSON.stringify({ display_text: 'Visit', url: 'https://example.com' })
  }]
})

// Group member label
await client.sendMessage(jid, {
  groupLabel: { labelText: 'Admin' }
})

// Broadcast to specific group members
await client.sendMessageMembers(jid, { extendedTextMessage: { text: 'Announcement' } }, {})
```

> Fields such as `sender` and `participant: true` in the second argument of
> `sendMessage` or `relayMessage` are used to provide group-participant context.
> See the [API Reference](docs/API.md) for details on each socket layer.

## Events

All interactions are exposed through events on `client.ev`:

```js
client.ev.on('connection.update', ({ connection, lastDisconnect }) => {
  console.log('Connection status:', connection)
})

client.ev.on('messages.upsert', ({ messages, type }) => {
  for (const m of messages) {
    console.log('Incoming message:', m.message)
  }
})

client.ev.on('creds.update', saveCreds)
```

See the full list of available events in `lib/Types/Events.js`.

## Examples

Ready-to-run examples are available in [examples/](examples):

- [examples/qr-login.js](examples/qr-login.js) — QR login with automatic reconnection
- [examples/pairing-code.js](examples/pairing-code.js) — pairing code login
- [examples/store-usage.js](examples/store-usage.js) — using the in-memory store

## Known Limitations

- **No `.d.ts` files for `lib/`** — only `WAProto` ships TypeScript definitions. Full autocomplete requires the TypeScript source, which is not included in this build.
- **Console banner on import** — every time the package is imported, an ASCII art banner and promotional link are printed to stdout. In production or multi-instance setups, this can become noisy.
- **`optionHash` for per-option image polls** is not implemented — it appears to require further reverse-engineering at the WhatsApp APK level.
- See the [API Reference](docs/API.md#️-important-notes) for additional technical notes.

## Contributing

Issues and pull requests are welcome. Before submitting a PR, please run:

```bash
npm run lint
npm test
```

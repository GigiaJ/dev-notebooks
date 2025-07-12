---
id: nmwab52k73ryuyn8ltggauf
title: 'cinny-draft-sync'
desc: ''
updated: 1752095650112
created: 1752094620353
jupyter:
  jupytext:
    cell_metadata_filter: -all
    formats: ipynb,md
    main_language: typescript
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.15.0
  kernelspec:
    display_name: TypeScript
    language: typescript
    name: tslab
---

Alright to start we'll bind this together with

```
jupytext --set-formats ipynb,md --output /home/jaggar/Dendron/dependencies/localhost/dev/notes/scratch.2025.07.09.155651.md  --sync /home/jaggar/Code/cinny/notebooks/draft-sync.ipynb
```


Next I want to run a simple test to see how Jupyter notebooks work.

<!-- #raw -->
console.log('This is a test');
<!-- #endraw -->

# Cross Device Draft Sync


Outlining the issue to solve is as followed:

- Device A should be able to type a message and have it first stored in the local storage. The draft should then be synced to the server.
- Device B should always reflect the content of Device A assuming it has no draft in local storage draft

For the sake of brevity in that explanation I am choosing to omit the details such as encryption and decryption. However, for encryption 
we check if the room supports encryption and if it does not support it then we simply don't use it. If it does then we use the room key
to encrypt our messages and similarly we decrypt them the same way. For Javascript we have the matrix-js-sdk which doesn't have a direct
generic way to decrypt, so we do have to utilize casting our CryptoAPI to the CryptoBackend to fulfil that goal.

##### Edge cases to consider
An important facet to consider is the app is usable while offline. This means one could be offline and type a draft on a mobile device and turn the screen off. After which one may then type another message on another device.

A draft should never be overwritten, so the in previous attempts a timestamp check isn't enough. Nor is a event-id. In the above scenario it is feasible to have the same event-id since they'll share one. If neither device sends the message then for most cases they'll still have the same device id. If the online device types a completely different message then the mobile device reconnects to the internet... The draft might be entirely shifted.

Some ideas to consider might be always changing the event-id when the draft is empty. When a user does a select all and changes the draft an onChange listener detects for the brief moment when the text field is 'empty' before the new characters typed appear.

This is a decent solution for that case, but let us say that the user DID mean to revise the mobile message draft and do in fact no longer want it. This is generally seen as a *send*. 

Mapping draft IDs won't scale but showing the user diffs between their server draft and client draft may come across as clunky. 

That said, offering an opt-in or toggle like "Always show me the diff between drafts and allow me to choose" might not be an awful option.
So considering the above it seems like opt-in to cloud message draft sync is the best possible idea where the server ALWAYS has the absolute true state and the default behavior is local draft sync in which the local storage is used.


##### Mocking
Now we're going to begin with mocking out the Matrix client to test the rest of the behavior in the file.

```typescript
// A fake MatrixClient for testing purposes
const mockMx = {
  getUserId: () => '@user:example.com',
  getCrypto: () => ({
    // A fake crypto backend
    encryptEvent: async (event: any) => {
      // Simulate encryption by adding a property
      event.event.type = 'm.room.encrypted';
      event.event.content = { algorithm: 'm.megolm.v1.aes-sha2' };
    },
    attemptDecryption: async (event: any) => {
      // Simulate decryption by providing clear content
      event.clearEvent = {
        type: 'm.room.message',
        content: {
          msgtype: 'm.text',
          body: 'decrypted draft',
          content: [{ type: 'paragraph', children: [{ text: 'Hello World' }] }],
        },
      };
    },
  }),
};
<!-- #endraw -->

Now we'll start with pulling out toPlainText for testing since it has no inherent dependencies:

```typescript
// --- The function to test ---
function toPlainText(nodes: any[] | null | undefined): string {
  if (!Array.isArray(nodes)) {
    return '';
  }
  return nodes.map((n) => (n as any).children?.map((c: any) => c.text).join('') ?? '').join('\n');
}


// --- The test ---
const testNodes = [
  { type: 'paragraph', children: [{ text: 'First line.' }] },
  { type: 'paragraph', children: [{ text: 'Second line.' }] },
];

const plainTextResult = toPlainText(testNodes);
console.log('Plain Text Result:', plainTextResult);
// Expected output: "First line.\nSecond line."
```

Now we'll test a piece that *does* have dependencies in getContentFromEvent

```typescript
// --- The functions to test ---
const getContentFromEvent = (event: any) => {
  const decryptedContent = event.getClearContent();
  if (!decryptedContent || decryptedContent.msgtype === 'm.bad.encrypted') {
    return event?.event?.content?.content;
  }
  delete decryptedContent.body;
  delete decryptedContent.msgtype;
  return decryptedContent.content;
};

async function decryptDraft(mx: any, savedEventData: any): Promise<any | null> {
  const cryptoApi = mx.getCrypto();
  if (!cryptoApi) return null;

  // We need a mock MatrixEvent class for the test
  class MockMatrixEvent {
    clearEvent: any = null;
    constructor(public event: any) {}
    getClearContent() { return this.clearEvent; }
    async attemptDecryption(cryptoBackend: any) { await cryptoBackend.attemptDecryption(this); }
  }

  const eventToDecrypt = new MockMatrixEvent(savedEventData);
  try {
    await eventToDecrypt.attemptDecryption(cryptoApi);
    return getContentFromEvent(eventToDecrypt);
  } catch (e) {
    return null;
  }
}

// --- The test ---
const mockEncryptedEvent = {
  type: 'm.room.encrypted',
  content: { algorithm: 'm.megolm.v1.aes-sha2' },
};

// We pass our mockMx object here
const decryptedContent = await decryptDraft(mockMx, mockEncryptedEvent);
console.log('Decrypted Content:', JSON.stringify(decryptedContent));
// Expected output: The 'Hello World' content from our mock
<!-- #endraw -->

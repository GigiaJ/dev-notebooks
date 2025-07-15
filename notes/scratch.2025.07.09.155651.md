---
id: nmwab52k73ryuyn8ltggauf
title: 'Cinny Cross Device Message Draft Sync'
desc: ''
updated: 1752362610124
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

##### Problem to solve
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

```

Now we'll start with pulling out toPlainText for testing since it has no inherent dependencies:

```typescript
// --- The function to test ---
const toPlainText = (nodes: any[] | null | undefined): string => {
  if (!Array.isArray(nodes)) return '';
  return nodes.map((n) => n.children?.map((c: any) => c.text).join('') ?? '').join('\n');
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

const decryptDraft = async (mx: any, savedEventData: any): Promise<any | null> => {
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
```

Now we have a functional sample for "decrypting content". With that we can move towards actually writing the sync behavior from the server and local storage. It should be clarified that much of this is written as a result of the original behavior of useMessageDraft.tsx not being perfectly desirable. Sync bugs cause UI issues for the user and impact their ability to type messages. So instead of just trying to make minute changes we'll articulate each point until we reach a functional result.

So next we should mock the local storage behavior. We can make an atom to store the plain Javascript object rather than actually using the IndexedDB. This functionally will behave the same for our intents of making sure the sync behavior between the local and server is consistent and well-behaved.

```typescript
// --- Mocks ---

// Mock consistent Room Id
const roomId = 'draft-event-key';

// Mock Jotai's state management
const fakeDB = {}; // Our fake in-memory database
const mockSetDraftEvent = async (newValue: any) => {
  console.debug('Mock setDraftEvent called with:', JSON.stringify(newValue));
  fakeDB[roomId] = newValue;
};

// Mock debounce to run immediately
const mockDebounce = async (fn: Function, _wait: number) => {
  return (...args: any[]) => fn(...args);
};

// Mock the server sync function
let syncSpy = {};
const mockSyncDraftToServer = async (eventToSave: any) => {
  console.debug('Mock syncDraftToServer called with:', JSON.stringify(eventToSave));
  syncSpy[roomId] = eventToSave;
};
```

```typescript
// --- Functions to Test ---

const debouncedUpdate = await mockDebounce(async (newContent: any[]) => {
  const isEmpty = newContent.length <= 1 && toPlainText(newContent) === '';

  const partial = {
    sender: '@user:example.com',
    type: 'm.room.message',
    room_id: roomId,
    content: {
      msgtype: 'm.text',
      body: 'draft',
      content: isEmpty ? [] : newContent,
    },
    origin_server_ts: Date.now(),
    event_id: `$fake-event-id`,
  };

  await mockSetDraftEvent(partial);
  await mockSyncDraftToServer(partial);
}, 25);

const updateDraft = async (newContent: any[]) => {
  return await debouncedUpdate(newContent);
};
```

```typescript
// --- Test ---

const myNewDraftContent = [
  { type: 'paragraph', children: [{ text: 'This is my new draft.' }] }
];

await updateDraft(myNewDraftContent);

console.log('Final state of fakeDB:', fakeDB);
console.log('Final state of syncSpy:', syncSpy);
```

The above provide a simulation of storing the text into the local storage and syncing to the server as well as debouncing adequetely. From that it is possible to simulate user input and syncing from the server to then test for any concurrency issues or sync problems. After which the edge case solutions can be implemented in the form of a settings check which will need to be mocked too.

```typescript
// --- Mocks ---

// A fake MatrixEvent class to simulate server events
class MockMatrixEvent {
  constructor(private event: any) {}
  getType = () => this.event.type;
  getContent = () => this.event.content;
}

let mockServerData = {
  sender: '@user:example.com',
  type: 'm.room.message',
  room_id: roomId,
  content: {
    msgtype: 'm.text',
    body: 'draft',
    content: [{ type: 'paragraph', children: [{ text: 'server version' }] }]
  },
  origin_server_ts: 1000,
  event_id: `$fake-event-id`,
};

```

```typescript
// --- Functions to Test ---

// This is a simplified version of handleDraftContent for our test
const handleDraftContent = async (event: any): Promise<any[] | null> => {
  if (!event) return null;
  return event.content?.content ?? null;
}

// The main function we are testing
const handleAccountData = async (event: any, roomId: string, localDraft: any) => {
  //if (event.getType() !== 'org.cinny.draft.v1') return; Not applicable here
  //const allSyncedDrafts = event.content;
  //const serverEvent = allSyncedDrafts[roomId] as any | undefined;
  const serverEvent = event;
  if (!serverEvent) return;

  const isServerNewer = serverEvent.origin_server_ts > (localDraft?.origin_server_ts ?? 0);

  if (isServerNewer) {
    const serverContent = await handleDraftContent(serverEvent);
    const localContent = await handleDraftContent(localDraft);

    if (toPlainText(serverContent) !== toPlainText(localContent)) {
      await mockSetDraftEvent(serverEvent);
    }
  }
}

```

```typescript
// --- The Test Runner ---

const runServerSyncTest = async () => {

  // == SCENARIO 1: Server draft is NEWER and should update local state ==
  console.debug('--- Running Scenario 1: Server is Newer ---');
  const localDraft =  {
  sender: '@user:example.com',
  type: 'm.room.message',
  room_id: roomId,
  content: {
    msgtype: 'm.text',
    body: 'draft',
    content: [{ type: 'paragraph', children: [{ text: 'server version' }] }]
  },
  origin_server_ts: 2000,
  event_id: `$fake-event-id`,
};
  
  await handleAccountData(mockServerData, roomId, localDraft);
  console.debug('Result 1:', JSON.stringify(fakeDB[roomId]));
  console.log('\n');

  // Reset DB with the initial local draft
  fakeDB[roomId] = localDraft;

  // == SCENARIO 2: Server draft is OLDER and should be ignored ==
  console.log('--- Running Scenario 2: Server is Older ---');
  mockServerData.origin_server_ts = 500;
  mockServerData.content = {
    msgtype: 'm.text',
    body: 'draft',
    content: [
      { type: 'paragraph', children: [{ text: 'old server version' }] }
    ]
  };


  await handleAccountData(mockServerData, roomId, localDraft);
  console.log('Result 2:', JSON.stringify(fakeDB[roomId]));

  mockServerData.origin_server_ts = 2500;
  mockServerData.content = {
    msgtype: 'm.text',
    body: 'draft',
    content: [{ type: 'paragraph', children: [{ text: 'new new server version' }] }]
  }

}

await runServerSyncTest();
```

A key situation to avoid is when syncing from the server and updating the local storage and editor... is that it is never allowed to resync back that update.

The above code is not complete enough yet to check for such a circumstance, so below we should draft out a mock of the editor logic.

```typescript
// --- Mocks and Spies ---

// Spies to track function calls
let editorSpy = { reset: 0, insert: 0, select: 0 };
let newDraftState: any = null;

// A fake editor object with the properties our logic needs
const mockEditor = {
  children: [{ type: 'paragraph', children: [{ text: 'Some random initial text.' }] }],
  updateContent: (text: string) => { mockEditor.children[0].children[0].text = text },
};

// Mock functions that the logic will call
const resetEditor = (editor: any) => {
  editorSpy.reset++;
  editor.updateContent('');
  console.debug('spy: resetEditor was called');
};
const Transforms = {
  insertFragment: (editor: any, fragment: any) => {
    editor.children = fragment;
    editorSpy.insert++; },
  select: (editor: any, location: any) => { editorSpy.select++; },
};
const Editor = {
  end: (editor: any, location: any) => 'end_location', // Just needs to return something
};
```

```typescript
// --- Logic to Test ---

// Logic from `handleOnChange` useCallback
const runOnChangeLogic = async (editor: any) => {
  console.debug(JSON.stringify(editor.children))
  return await updateDraft([...editor.children]);
}

// Logic from `useEffect`
const runEffectLogic = async (msgDraft: any, editor: any) => {
  if (!msgDraft || msgDraft === 0) {
    console.debug("No message draft passed.")
    resetEditor(editor);
    return;
  }
  if (JSON.stringify(msgDraft) === JSON.stringify(editor.children)) {
    console.debug('Effect logic: No changes detected.');
    return;
  }
  
  console.debug('Effect logic: Changes detected, updating editor.');
  resetEditor(editor);
  Transforms.insertFragment(editor, msgDraft);
  Transforms.select(editor, Editor.end(editor, []));
}
```

Now while those are normally React components, the only concern as far as syncing cares is the changes incurred by said React components. So assuming their behavior is cleanly doable and enables testing out even the editor behavior which is important for actually providing a level of coverage to the end-user experience.

```typescript
// --- Test Runner ---

// Scenario 1: Test the onChange handler
console.log('--- SCENARIO 1: Testing onChange ---');
await runOnChangeLogic(mockEditor);
console.log('Result: setMsgDraft was called with:', JSON.stringify(fakeDB[roomId]));
console.log('\n');


// Scenario 2: Test the effect when content is the SAME
console.log('--- SCENARIO 2: Testing effect with no changes ---');
await runEffectLogic(mockEditor.children, mockEditor);
console.log(JSON.stringify(mockEditor));
console.log('Result: Editor functions called:', editorSpy); // All should be 0
console.log('\n');


// Scenario 3: Test the effect when content is DIFFERENT
console.log('--- SCENARIO 3: Testing effect with new content ---');
editorSpy = { reset: 0, insert: 0, select: 0 }; // Reset spy
const incomingDraft = [{ type: 'paragraph', children: [{ text: 'new server text' }] }];
runEffectLogic(incomingDraft, mockEditor);
console.log('Result: Editor functions called:', editorSpy); // All should be 1
```

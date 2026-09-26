# Discord Knowledge Assistant — Reference Architecture

Notes from building and maintaining a production Discord AI assistant for a documentation-heavy community. This is the architecture and the design decisions behind it, not a drop-in bot. There is no installer here and no configuration file to fill in — the useful part is the reasoning, and most of it is not obvious until something has been running in front of real members for a few months.

## The problem

A community forms around a product with substantial documentation — a game, a course, a piece of software. The documentation exists. Almost nobody reads it. The same questions repeat every week, and one or two people answer them by hand, indefinitely.

An AI assistant looks like the obvious fix. Point a retrieval model at the documents, put it in the channel, done.

Most implementations built that way fail. They do not fail loudly, which is the difficulty. They fail by giving answers that sound correct and are not, until members stop trusting the assistant and go back to asking a human.

## Design decisions

### 1. Refusing is a feature

The default behaviour of a retrieval assistant is to answer. Given partial information it will assemble something plausible from unrelated passages and present it with the same confidence it uses for a direct quote.

For a community built on rules or procedures, that is the worst available failure mode. A member acts on the answer. Another member contradicts it. The dispute is resolved against the assistant, and from that point the assistant is a thing people check rather than a thing people use.

So the system prompt is built around declining rather than answering:

- Answer only from what is explicitly found in the documents.
- Never treat the absence of information as proof of the negative. "It does not say you can" is not the same as "you cannot".
- When something is not found, return one fixed phrase, verbatim.
- Add nothing after that phrase — no speculation, no "but generally", no helpful guess.

A fixed refusal string has a second benefit that is easy to miss: it makes gaps countable. Grep the logs for that exact sentence and you have a ranked list of what the documentation does not cover, which is the most useful thing you can hand back to the person who owns the documents.

```
Answer only from the attached documents.

If the answer is not explicitly present, reply with exactly:
"I could not find this in the documents I have."
Add nothing after that sentence.

Do not infer that something is disallowed because it is absent.

Name the source document in every answer.

If two documents disagree, present both, name each, and say
that they conflict. Do not choose between them.
```

### 2. Versions must be isolated, not tagged

A product with several live editions cannot have all of them in one knowledge base.

The tempting approach is to tag each document with its edition and let the model filter. It does not hold up. Retrieval has no reliable way to tell which edition a passage belongs to once it is a chunk of text among other chunks, and metadata filtering is only as good as the labelling — which, in a real document set assembled over years by different people, is inconsistent.

The result is blended answers: a paragraph from edition two and a paragraph from edition three, joined into one confident reply that matches neither printed copy.

One edition, one knowledge store, one channel. The routing layer decides which store to read before the model sees anything at all, so the model is never in a position to mix them.

```js
// One channel maps to exactly one knowledge store.
// Anything unmapped is not answered at all.
const ROUTES = {
  'channel-product-a-ed1': 'store_product_a_ed1',
  'channel-product-a-ed2': 'store_product_a_ed2',
  'channel-product-b':     'store_product_b',
};

function storeForChannel(channelId) {
  return ROUTES[channelId] ?? null;
}
```

Returning `null` rather than a default store matters. A default store is how a question about one product quietly gets answered from another.

### 3. Every answer names its source

Without attribution, a disputed answer cannot be resolved. You cannot tell whether the model invented it, retrieved the wrong document, or read a document that is itself out of date. All three look identical from the outside, and all three have different fixes.

With the source document named in the answer, the same dispute takes seconds. A member opens their copy and checks.

One practical warning. If you suppress the retrieval system's internal citation markers — and you probably want to, because they are ugly in a chat message — write that instruction narrowly. A broad "do not cite sources" will also remove the plain-text source naming you asked for in the line above, and you will not notice until someone asks where an answer came from.

### 4. Conflicting sources must surface, not resolve

Real document sets contradict themselves. A base document and a variant. An original and an amendment nobody merged. Two files with nearly the same name and different contents.

Left to itself the model picks one, silently, and you never learn the conflict exists.

The instruction is to present both, name each source, and state plainly that they disagree. The assistant is not the right place to adjudicate — the person who owns the documents is, and surfacing the conflict is how they find out it is there.

## Operational lessons

### Conversion failures are silent

A PDF with sideways column headers extracts as scrambled text: numbers with no labels attached to them. The file size looks healthy. The indexer reports success. The content is worthless.

Worse, a file can extract to almost nothing and still index as completed. A near-empty file can sit in a production store for months before anyone notices.

Check the output, not the status. Compare byte size against comparable files. Read the first page of what was actually extracted.

### Usage-based billing fails quietly

An exhausted credit balance returns a 429 and the assistant stops answering. Nothing alerts anyone. Members assume the bot is broken, stop asking, and do not report it — because a bot that is down is not a bug anyone feels responsible for.

Monitor the balance, or enable auto-recharge, or both.

### Build alongside, then repoint

Replacing a knowledge store by deleting it and re-uploading destroys the working state the moment the upload fails partway.

Build the new store alongside the old one. Verify it. Then change one routing value. Keep the old store for a week. Rollback is one line, not a re-upload under pressure.

### Filenames lie

Across a single project: a store whose name did not match the edition of the files inside it, a folder of "converted" documents that had never been converted, and a store whose contents belonged to a different product than its name suggested.

Verify by reading content. Never by reading names.

## Architecture

```
         Approved documents
    one set per edition, verified on load
                    |
                    v
   +---------+  +---------+  +---------+
   |  store  |  |  store  |  |  store  |
   |  ed. 1  |  |  ed. 2  |  | prod. B |
   +---------+  +---------+  +---------+
        ^            ^            ^
        |            |            |
        +------------+------------+
                     |
              reads exactly one
                     |
           +---------------------+
           |    Routing layer    |
           |  channel --> store  |
           +---------------------+
              ^               |
     question |               | answer, with
     + channel|               | source named
              |               v
           +---------------------+
           |    Chat platform    |
           +---------------------+


   Scoring and roles — a separate path

           +---------------------+
           |    Chat platform    |
           +---------------------+
                     | activity events
                     v
           +---------------------+
           |  Activity scoring   |
           |  daily cap per type |
           +---------------------+
                     v
           +---------------------+
           |   Member records    |
           |  points, rank, log  |
           +---------------------+
                     v
           +---------------------+
           |      Role sync      |
           |  applies rank change|
           +---------------------+
```

The two paths share the chat platform and nothing else. Keeping them separate means the assistant can be taken offline for a knowledge base rebuild without stopping scoring, and a scoring bug cannot affect what the assistant answers.

Daily caps per activity type are what stop the scoring path being farmed. Without them, the cheapest activity to repeat becomes the only activity anyone does.

## Not included

The client implementation is not public. No code, configuration, document set or identifier from that system appears in this repository. What is here is the pattern and the reasoning, written from scratch; the snippets above are illustrative sketches, not extracts.

## License

MIT.

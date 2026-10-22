# Markdown Injection in Telegram Bots: When User Data Breaks Out of Your Message Template

*A practical look at a small bug class that hands an attacker the voice of your official bot.*

## TL;DR

Many backends send notifications through the Telegram Bot API with `parse_mode: "Markdown"` and build the message text by string interpolation. If any interpolated value can be controlled by a user (a username, a product title, an item name), that user can close your formatting early and inject their own Markdown. The most useful injection is a clickable link with attacker-chosen text, delivered by **your** bot to **other** users. The fix is to stop treating message text as code: escape for the exact parse mode you use, or better, send formatting as explicit `entities` and never parse user data at all.

## The pattern

A typical notification template looks like this:

```js
function alertTemplate({ itemName, daysLeft }) {
  return {
    text: `Your item \`${itemName}\` expires in *${daysLeft} days*.`,
    parse_mode: "Markdown",
  };
}
```

It looks harmless. The value is even "safely" wrapped in a code span. But Markdown is a small markup language, and `itemName` is being inserted into it unescaped.

## Why the code span does not protect you

Telegram's legacy Markdown mode supports `*bold*`, `_italic_`, `` `inline code` ``, ```` ```pre``` ```` and `[text](url)`. A backtick inside the value closes the code span early, and everything after it is parsed as ordinary Markdown.

There is one constraint that makes naive payloads fail: the Bot API **rejects a message whose entities do not balance** (`Bad Request: can't parse entities`). A careless payload produces an error, not an injection, and many developers stop testing there.

A working payload therefore keeps every entity balanced. It closes the template's span, injects its own content, and reopens a span so that the template's closing backtick has something to pair with:

```
x` [Claim your refund here](https://attacker.example) `y
```

After interpolation the message becomes:

```
Your item `x` [Claim your refund here](https://attacker.example) `y` expires in *5 days*.
```

Telegram parses this as: code `x`, then a **real clickable link**, then code `y`, then bold `5 days`. The message is accepted and rendered.

## Verifying it properly

Do not rely on a local Markdown renderer: Telegram's parser has its own rules. Send the message to a test chat you control, and read the API response. `sendMessage` returns the parsed message, including its `entities` array:

```json
{"type": "text_link", "offset": 12, "length": 22, "url": "https://attacker.example"}
```

A `text_link` entity in the response is Telegram's own confirmation that the injection renders. Test only with your own bot token and your own chat.

## Why it matters

- **Trusted channel.** The link arrives from the official bot that users have already allowed to message them, often next to a legitimate call-to-action button.
- **One-to-many.** If the injected value belongs to a shared object (an item many users follow), every subscriber receives the payload, every time the notification fires.
- **No interaction with the attacker.** The victim only needs to be a normal recipient of the notification.
- **Hidden destination.** In Telegram's UI, a text link shows its label, not its URL, so the destination is not visible at a glance.

The typical real-world outcome is phishing: the link leads to a convincing clone of the service's login or payment page.

## Fixes, from best to acceptable

**1. Do not parse user data at all: send `entities`.** `sendMessage` accepts an explicit `entities` array instead of `parse_mode`. You build the text as plain text, then describe the formatting by offset and length. User data is never interpreted as markup.

```js
const name = itemName; // untrusted, stays plain text
const prefix = "Your item ";
const text = `${prefix}${name} expires in ${daysLeft} days.`;
const entities = [
  { type: "code", offset: prefix.length, length: name.length },
];
// sendMessage({ chat_id, text, entities })  — no parse_mode
```

Offsets and lengths are in **UTF-16 code units**, so an emoji outside the Basic Multilingual Plane counts as two. In JavaScript `str.length` already counts UTF-16 code units; in Python or Go, compute the length explicitly (for example `len(s.encode("utf-16-le")) // 2`).

**2. Use `MarkdownV2` or `HTML` with a correct escaper.** Legacy `Markdown` is kept only for backward compatibility. Its escaping rules are limited, and escaping inside an entity is not allowed. `MarkdownV2` defines exact rules:

- Outside entities, escape every one of ``_ * [ ] ( ) ~ ` > # + - = | { } . !`` with a preceding `\`.
- Inside `code` and `pre`, escape every `` ` `` and `\`.
- Inside the `(...)` of an inline link, escape every `)` and `\`.

```js
const escapeMdV2 = (s) => s.replace(/[_*\[\]()~`>#+\-=|{}.!\\]/g, "\\$&");
const escapeMdV2Code = (s) => s.replace(/[`\\]/g, "\\$&");
```

With `HTML`, escape `&`, `<` and `>` in every interpolated value.

**3. Defence in depth.** Bound the length of interpolated fields. Where the domain allows it, reject or normalise control and markup characters at input. Apply the same review to every other template that reuses the value: push notifications that build URLs from it, emails and webhooks.

## A quick audit checklist

1. Search for `parse_mode` in the codebase and list every template that uses it.
2. For each interpolated value, trace it back to its source. Can a user influence it, directly or through an object they create?
3. Check whether the value is escaped for the **exact** mode in use. A MarkdownV2 escaper does not make legacy Markdown safe, and vice versa.
4. Look for the same value flowing into other channels: URL builders (`https://app.example/${value}`), email HTML, deep links.
5. Prove it end to end in a test chat and capture the `entities` in the API response.

## Takeaway

A notification template is code that runs in your users' chat client. Treat every interpolated value like you would treat input to an HTML template: escape it for the exact language it lands in, or keep it out of the markup entirely.

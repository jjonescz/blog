---
title: Freezing Puppeteer DOM
slug: freeze-puppeteer
date: 2022-05-09T08:39:12+0200
summary: How to freeze a page's DOM in Puppeteer,
  so the page's JavaScript cannot manipulate it,
  while traversing the DOM tree or taking its screenshot.
tags:
  - Puppeteer
categories:
  - Programming
cover:
  image: cover.png
  alt: Puppeteer freezes DOM
  relative: true
---

**TL;DR**: See [my solution](#my-solution).

While developing an [automatic web scraper](https://github.com/jjonescz/awe)
for the [Apify](https://apify.com) company,
I needed to extract features from lots of pages.
(I've used [Puppeteer](https://github.com/puppeteer/puppeteer) v11
to load the pages.)
More specifically, I was working with the Document Object Model (DOM) tree
of each page.
However, some pages use JavaScript code modifying the DOM tree dynamically.
This was not ideal, as I wanted to save three things for each page<!--
-->&mdash;HTML, some visual characteristics, and a screenshot&mdash;<!--
-->and I needed all of these things to be consistent.
But, while extracting HTML,
the client-side JavaScript code of the target page could have modified its DOM,
so the visual characteristics would not match the DOM anymore
(and so would not the screenshot).

## Disabling JavaScript

Disabling JavaScript in Puppeteer can definitely help:

```js
page.setJavaScriptEnabled(false);
```

This is not useful in general, though,
because many pages use JavaScript to actually render their main content.
So with JavaScript disabled, the page might render only a white screen,
not very interesting to extract any data from.

## Freezing the page

Ideally, I would like to load the page, let JavaScript to do its magic,
and then *freeze* it, i.e., disable JavaScript once the page is fully loaded.
Unfortunately, this is not possible out-of-the-box in Puppeteer.

### Existing solutions

Some have
[suggested to issue a breakpoint](https://stackoverflow.com/q/51897899/9080566):

```js
await page.evaluate(() => {
  debugger;
});
```

However, this does not let you use `page.evaluate`
(which I definitely need to use
in order to extract HTML and visual characteristics).

Another option mentioned in
[a Stack Overflow answer](https://stackoverflow.com/a/51902806/9080566)
is removing event handlers from all DOM nodes:

```js
await page.evaluate(() => {
  document.querySelectorAll('*').forEach(element => {
    element.parentNode.replaceChild(element.cloneNode(true), element);
  });
});
```

Again, there is a problem&mdash;this does not stop already running scripts
or asynchronous `setTimeout` handlers.

### My solution

I simply take a snapshot of the page
(as [MHTML](https://en.wikipedia.org/wiki/MHTML)),
disable JavaScript,
and reload the page from the snapshot:

```ts
import { unlink, writeFile } from 'fs/promises';
import puppeteer from 'puppeteer-core';
import { pathToFileURL } from 'url';

function freezePage(page: puppeteer.Page) {
  const wasJavaScriptEnabled = page.isJavaScriptEnabled();

  // Capture snapshot.
  // WARNING: The `snapshotPath` must have `.mhtml` file extension!
  const snapshotPath = 'snapshot.mhtml'; // or temporary file...
  const cdp = await page.target().createCDPSession();
  const { data } = await cdp.send('Page.captureSnapshot', {
    format: 'mhtml',
  });
  await writeFile(snapshotPath, data, { encoding: 'utf-8' });

  try {
    // Reload page.
    page.setJavaScriptEnabled(false);
    page.goto(pathToFileURL(snapshotPath).toString(), {
      waitUntil: 'networkidle0',
    });
  } finally {
    page.setJavaScriptEnabled(wasJavaScriptEnabled);

    // Remove snapshot.
    await unlink(snapshotPath);
  }
}
```

Note that the snapshot must be saved into a file with extension `.mhtml`.
That's the only way it can be then loaded by Puppeteer correctly.
If loaded from an in-memory string or without the extension,
Puppeteer would treat the snapshot as plain HTML which would not work.

Why MHTML?
Because it includes much more than HTML, such as `<iframe>`s.
So it should be fairly comprehensive snapshot of the whole page.
See also
[puppeteer/puppeteer#3658](https://github.com/puppeteer/puppeteer/issues/3658).

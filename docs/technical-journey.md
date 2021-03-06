# The technical journey

In order to understand how the internal architecture of React Screenshot Test came about, let's rewind a bit.

The original idea was simple: what if we could write tests for React components that looked almost like snapshot tests, but compared actual screenshots instead of HTML?

In case you're not already familiar with Jest snapshots, here is an example pulled from [their documentation](https://jestjs.io/docs/en/snapshot-testing).

```js
import React from "react";
import Link from "../Link.react";
import renderer from "react-test-renderer";

it("renders correctly", () => {
  const tree = renderer
    .create(<Link page="http://www.facebook.com">Facebook</Link>)
    .toJSON();
  expect(tree).toMatchSnapshot();
});
```

This generates the following snapshot of the rendered HTML:

```js
exports[`renders correctly 1`] = `
<a
  className="normal"
  href="http://www.facebook.com"
  onMouseEnter={[Function]}
  onMouseLeave={[Function]}
>
  Facebook
</a>
`;
```

Instead, I wanted it to generate a screenshot:

![facebook-link-screenshot](./assets/facebook-link-screenshot.png)

It turns out, generating a screenshot from a React component isn't as straightforward as I'd hoped.

The first thing you need to get a screenshot of a piece of HTML is, obviously, a web browser. Luckily, Google Chrome can be controlled easily from Node by using the [Puppeteer library](https://github.com/puppeteer/puppeteer). Great, we have a browser.

Now, what about the HTML? If you're familiar with server-side rendering, you may already have the answer: use `ReactDomServer.renderToString()`! Indeed, that's exactly what I used.

I decided to spin up a local server (called the "component server"), which would use server-side rendering to serve the HTML. Each "node" (a React component with a specific set of props) would be allocated a random ID, and therefore a unique URL. For example, `/render/abc-123` may show our wonderful Facebook link above.

With this local server, taking a screenshot was straightforward:

- Add the node to the component server and store its generated ID.
- Open a browser with Puppeteer.
- Navigate to `http://localhost:[port]/render/[node-id]`.
- Take a screenshot with [`page.screenshot()`](https://github.com/puppeteer/puppeteer/blob/master/docs/api.md#pagescreenshotoptions).

The last piece of the puzzle was comparing PNG snapshots. Fortunately, the folks at American Express built [jest-image-snapshot](https://github.com/americanexpress/jest-image-snapshot) which does exactly that. No need to reinvent the wheel.

All done!

Well, not exactly. Much to my dismay, as soon as I set up React Screenshot Test on CircleCI, tests failed. There was a small (2%) visual diff between the screenshots I had generated on my MacBook Pro, and the ones being generated by CircleCI.

It turns out, rendering is [expected to be inconsistent](https://github.com/puppeteer/puppeteer/issues/661) between different platforms. Well, bummer. But there was an interesting idea in that thread: what if we used Docker?

One option would have been to say "always run your tests within Docker, otherwise you'll have a bad time™". However, that wouldn't have been a great developer experience. What if React Screenshot Test seamlessly ran a browser within Docker for you?

This made things a bit more complicated. If the browser is running in Docker, but the tests are running on the host machine, you can't simply use Puppeteer's API to control the browser anymore. They're effectively running on different machines.

What's a good way to communicate between different machines? HTTP, of course!

This led to a new abstraction: the "screenshot server". It's an HTTP server with a single endpoint:

```
POST /render { url: string } -> image/png
```

Implementing this was straightforward with an Express server that ran Puppeteer. I created a [Docker image](https://hub.docker.com/r/fwouts/chrome-screenshot) which wrapped it all up into a nice package.

Then, I updated the screenshot logic to talk to the screenshot server instead of using Puppeteer directly.

Did that work?

No, CircleCI still wasn't happy. That's because CircleCI jobs already run inside Docker, and while they can run other Docker containers, [they cannot communicate with them](https://circleci.com/docs/2.0/building-docker-images/#separation-of-environments).

Why run Docker inside Docker anyway? This was an unnecessary level of nesting. Summarising:

- On the developer's machine, we want to run the screenshot server in Docker.
- Within Docker, we want to run Puppeteer directly.

Another constraint came about from the way that Jest works.

In order to run tests in parallel, Jest spins up multiple Node processes. Because they are separate processes, they cannot share memory. In particular, they cannot share access to a Puppeteer instance. This isn't ideal for resource sharing: launching a new Chrome binary for each test file doesn't scale very well!

Now, what if we started a single screenshot server before our tests? Since it's an HTTP server, all Node processes could talk to it, as long as they know its port.

The solution, which may seem a bit convoluted at first, was to:

- Start a screenshot server (either locally, or in Docker) in Jest's global setup hook.
- Start a different component server in each test (Express servers are cheap).
- Ask the screenshot server to take screenshots URLs served by the component server.

This is how React Screenshot Test came about.

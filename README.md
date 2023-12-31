# Remote DOM

Remote DOM lets you recreate a tree of [DOM elements](https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Using_the_Document_Object_Model) between JavaScript environments. You can think of it as an alternative to using an [`<iframe>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe): with Remote DOM, a developer builds up a tree of DOM elements in a sandboxed environment to render their user interface, just as they do inside an `<iframe>`. However, unlike an `<iframe>`, Remote DOM renders those DOM elements as part of the top-level HTML document, allowing UI elements to be more consistent, and preventing the need to load potentially-large JavaScript and CSS assets multiple times.

To help you use sandboxed JavaScript environments that are less expensive than a full `<iframe>`, Remote DOM also offers a minimal polyfill of key DOM APIs. This lets you use frameworks that would usually need to be run on the top-level HTML page, like [Preact](https://preactjs.com) and [Svelte](https://svelte.dev), inside of a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API).

## Examples

- [Minimal iframe](./examples/minimal-iframe/): an example using the smallest number of Remote DOM APIs to mirror basic HTML between a main page and an `<iframe>`.
- [Custom elements](./examples/custom-elements/): an example that defines custom elements, and synchronizes an HTML tree between a main page and an `<iframe>`.
- [Preact in a web worker](./examples/preact-web-worker/): an example that uses Remote DOM to render a Preact application inside of a web worker.
- [Svelte in a web worker, rendered by React](./examples/svelte-web-worker/): an example that uses Remote DOM to render a Svelte application inside of a web worker, which is rendered by a React application on the host page.

## Building a project with Remote DOM

To use Remote DOM, you’ll need a web project that is able to run two JavaScript environments: the “host” environment, which runs on the main HTML page and renders actual UI elements, and the “remote” environment, which is sandboxed and renders an invisible version of the DOM that will be mirrored by the host. You can [mix-and-match any combination of “host” and “remote” technologies](#examples) — you don’t need to use a particular JavaScript framework or backend technology to use Remote DOM. If you don’t know how to get started, we recommend starting a [Vite project](https://vitejs.dev) using whatever JavaScript library you prefer, as Vite lets you create `<iframe>` and Web Worker sandboxes with no extra configuration.

Once you have a project, install [`@remote-dom/core`](./packages/core/), which you’ll need to create the connection between host and remote environments:

```bash
npm install @remote-dom/core --save # npm
pnpm install @remote-dom/core --save # pnpm
yarn add @remote-dom/core # yarn
```

Next, on the “host” HTML page, you will need to create a “receiver”. This object will be responsible for receiving the updates from the remote environment, and mapping them to actual DOM elements. `@remote-ui/core` provides a few different types of receivers, but for now we use the `DOMRemoteReceiver`, which directly mirrors the DOM elements created remotely in the host HTML page. You’ll create a `DOMRemoteReceiver` and connect it to an existing HTML element in order to teach Remote DOM where to render the remote DOM elements:

```html
<!doctype html>
<html>
  <body>
    <div id="root"></div>

    <script type="module">
      import {DOMRemoteReceiver} from '@remote-dom/core/receiver';

      const root = document.querySelector('#root');

      const receiver = new DOMRemoteReceiver();
      receiver.connect(root);
    </script>
  </body>
</html>
```

Our host is ready to receive elements to render, but we don’t have a remote environment yet. For this example, we will use a hidden iframe, but the [examples section](#examples) shows alternative sandboxes. We’ll add the iframe to the host HTML page we started above, and we’ll also listen for `postMessage` events from the iframe, in order to pass changes in the remote tree to our receiver:

```html
<!doctype html>
<html>
  <body>
    <div id="root"></div>

    <iframe id="remote-iframe" src="/remote" hidden></iframe>

    <script type="module">
      import {DOMRemoteReceiver} from '@remote-dom/core/receiver';

      const root = document.querySelector('#root');
      const iframe = document.querySelector('#remote-iframe');

      const receiver = new DOMRemoteReceiver();
      receiver.connect(root);

      // We will send this message in the next step.
      window.addEventListener('message', ({source, data}) => {
        if (source !== iframe.contentWindow) return;
        receiver.connection.mutate(data);
      });
    </script>
  </body>
</html>
```

Next, let’s create the document that will be loaded into the iframe. It will use another utility provided by `@remote-ui/core`, `RemoteMutationObserver`, which extends the browser’s [`MutationObserver` interface](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) in order to communicate changes to the host.

```html
<!doctype html>
<html>
  <body>
    <div id="root"></div>

    <script type="module">
      import {RemoteMutationObserver} from '@remote-dom/core/elements';

      // We will synchronize everything inside this element to the host.
      const root = document.querySelector('#root');

      let count = 0;

      setInterval(() => {
        count += 1;
        render();
      }, 1_000);

      function render() {
        root.textContent = `Rendered ${count} ${
          count === 1 ? 'second' : 'seconds'
        } ago`;
      }

      // Send the mutations to the host via `postMessage`, which we just finished
      // adding a listener for in the previous step.
      const observer = new RemoteMutationObserver({
        mutate(mutations) {
          window.parent.postMessage(mutations, '*');
        },
      });

      observer.observer(root);
    </script>
  </body>
</html>
```

And just like that, the text we render in the `iframe` is now rendered in the host HTML page! You can see a full version of this example in the [“minimal iframe” example](./examples/minimal-iframe/).

### Adding custom elements

Now, just mirroring raw HTML isn’t very useful. Remote DOM works best when you define custom elements for the remote environment to render, which map to more complex, application-specific components on the host page. In fact, most of Remote DOM’s receiver APIs are geared towards you providing an allowlist of custom elements that the remote environment can render, which allows you to keep tight control over the visual appearance of the resulting output.

Remote DOM adopts the browser’s [native API for defining custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements) to represent these “remote elements”. To make it easy to define custom elements that can communicate their changes to the host, `@remote-dom/core` provides the `RemoteElement` class. This class, which is a subclass of the browser’s [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement), lets you define how properties, attributes, methods, and event listeners on the element should be transferred.

To demonstrate, let’s imagine that we want to allow our remote environment to render a `ui-button` element. This element will have a `primary` property, which sets it to a more prominent visual style. It will trigger a `click` event, and will also have a `focus()` method, which will focus the button when called.

First, we’ll create the remote environment’s version of `ui-button`. The remote version doesn’t have to worry about rendering any HTML — it’s only a signal to the host environment to render the “real” version. However, we do need to teach this element to communicate its `primary` property and `focus()` method to the host version of that element. We’ll do this using the [`RemoteElement` class provided by `@remote-dom/core`](./packages/core#remoteelement):

```html
<!doctype html>
<html>
  <body>
    <div id="root">
      <ui-button primary="">Clicked 0 times</ui-button>
    </div>

    <script type="module">
      import {RemoteElement} from '@remote-dom/core/elements';

      // For full details on defining remote elements, see the documentation
      // for `@remote-dom/core/elements`:
      // https://github.com/Shopify/remote-ui/tree/main/packages/core#elements
      class UIButton extends RemoteElement {
        static get remoteProperties() {
          return {
            // A boolean property can be set either by setting the attribute to a non-empty
            // value, or by setting the property to `true`.
            primary: {type: Boolean},
            // Remote DOM will convert the `click` event into an `onClick` property that
            // is communicated to the host.
            onClick: {event: true},
          };
        }

        static get remoteMethods() {
          return ['focus'];
        }
      }

      customElements.define('ui-button', UIButton);

      let count = 0;
      const button = document.querySelector('ui-button');

      button.addEventListener('click', () => {
        count += 1;

        button.textContent = `Clicked ${count} ${
          count === 1 ? 'time' : 'times'
        }`;
      });
    </script>

    <script type="module">
      // Just like the previous example, we will use a `RemoteMutationObserver` to
      // communicate the remote elements to the parent HTML page.
      import {RemoteMutationObserver} from '@remote-dom/core/elements';

      const root = document.querySelector('#root');

      const observer = new RemoteMutationObserver({
        mutate(mutations) {
          window.parent.postMessage(mutations, '*');
        },
      });

      observer.observer(root);
    </script>
  </body>
</html>
```

Finally, we need to provide a “real” implementation of our `ui-button` element, which will be rendered on the host page. The `DOMRemoteReceiver` we’ve used to receive elements in previous examples will automatically create an element matching the name provided in the remote environment, so we need to have a `ui-button` element defined in the host page. You can implement this element however you like, but for this example we’ll use the custom element APIs directly:

```html
<!doctype html>
<html>
  <body>
    <div id="root"></div>

    <iframe id="remote-iframe" src="/remote" hidden></iframe>

    <script type="module">
      class UIButton extends HTMLElement {
        // By default, `DOMRemoteReceiver` will assign remote properties as properties,
        // but only if the element has a matching property defined. Otherwise, the remote
        // properties will be set as attributes. We’ll observe the `primary` attribute
        // in order to update our rendered content when that attribute changes. We’ll
        // define a `onClick` method, though, which will be set to the value of the `onClick`
        // remote property.
        static get observedAttributes() {
          return ['primary'];
        }

        onClick;

        connectedCallback() {
          const primary = this.hasAttribute('primary') ?? false;

          const root = this.attachShadow({mode: 'open'});

          // We render a <slot> where we want the element’s children to go.
          root.innerHTML = `<button class="Button"><slot></slot></button>`;

          if (primary) {
            root.querySelector('.Button').classList.add('Button--primary');
          }

          // We’ll listen for clicks on our button, and call the remote `onClick`
          // property when it happens.
          root.querySelector('button').addEventListener('click', () => {
            this.onClick?.();
          });
        }

        attributeChangedCallback(name, oldValue, newValue) {
          if (name === 'primary') {
            const button = this.shadowRoot?.querySelector('.Button');

            if (button == null) return;

            if (newValue == null) {
              button.classList.remove('Button--primary');
            } else {
              button.classList.add('Button--primary');
            }
          }
        }

        // Remote DOM will automatically call methods on a custom element to satisfy
        // remote method calls.
        focus() {
          this.shadowRoot.querySelector('button').focus();
        }
      }

      customElements.define('ui-button', UIButton);
    </script>

    <script type="module">
      import {DOMRemoteReceiver} from '@remote-dom/core/receiver';

      const root = document.querySelector('#root');
      const iframe = document.querySelector('#remote-iframe');

      // In earlier examples, we did not pass any arguments, which allows the DOM
      // receiver to mirror any element it receives. By passing the `elements` option,
      // we are restricting the allowed elements to only the ones we list, which in this
      // case means only our `ui-button` element can be rendered.
      const receiver = new DOMRemoteReceiver({
        elements: ['ui-button'],
      });
      receiver.connect(root);

      window.addEventListener('message', ({source, data}) => {
        if (source !== iframe.contentWindow) return;
        receiver.connection.mutate(data);
      });
    </script>
  </body>
</html>
```

With those changes, you should now see your button rendering on the page, and responding to click events by updating its contents. You can see an extended version of this example in the [custom elements example](./examples/custom-elements/).

## Learn more

You’ve now seen the key elements of parts of Remote DOM, but it can help you with a few more related tasks, like allowing event handlers on custom elements and rendering remote elements using front-end JavaScript frameworks. For full details on the core APIs Remote DOM provides for rendering remote elements, please refer to the [documentation for `@remote-dom/core`](./packages/core/). You can also see the flexibility of Remote DOM in the [examples section](#examples), where the library is combined with different tools and frameworks.

This repository also contains a few companion packages to `@remote-dom/core` that are used in some of the examples above:

- [`@remote-dom/preact`](./packages/react/), which provides [Preact](https://preactjs.com) wrapper components for the remote environment, and the ability to map remote elements directly to Preact components on the host.
- [`@remote-dom/react`](./packages/react/), which provides [React](https://react.dev) wrapper components for the remote environment, and the ability to map remote elements directly to React components on the host.
- [`@remote-dom/polyfill`](./packages/polyfill/), which provides a minimal polyfill of the DOM APIs needed to run Remote DOM inside a non-DOM environment, like a Web Worker.
- [`@remote-dom/signals`](./packages/signals/), which lets you receive remote updates into a tree of [signals](https://preactjs.com/guide/v10/signals/).

## Want to contribute?

Check out our [contributing guide](CONTRIBUTING.md).

## License

MIT &copy; [Shopify](https://shopify.com/), see [LICENSE.md](LICENSE.md) for details.

<a href="http://www.shopify.com/"><img src="https://cdn.shopify.com/assets2/brand-assets/shopify-logo-main-8ee1e0052baf87fd9698ceff7cbc01cc36a89170212ad227db3ff2706e89fd04.svg" alt="Shopify" width="200" /></a>

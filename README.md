# Jupyter Extensible Renderers

## Story

Currently there are many different rendering engines for Jupyter notebook outputs. Off the top of my head...


1. JupyterLab
2. nteract
3. voila
4. nbconvert
5. Jupyter sphinx
6. Colab
7. VS Code

What this means is that if you have a custom way of rendering some widget (plotly, vega, etc), you maybe have to build
integration for ALL of these separately! It's insane!

Jupyter Widgets gets around this by providing a common base and [uses require.js](https://ipywidgets.readthedocs.io/en/stable/examples/Widget%20Custom.html) (in Classic Notebook and HTML pages) to let you provide custom renderers and it [uses a CDN](https://github.com/jupyter-widgets/ipywidgets/issues/1627) to grab the JS files.

Another way to get around this is to use a more general output, like html, and just embed a bunch of JS in there that creates a DOM node and does what you want. However, a limitation of this is that you don't have a way to get access to the kernel, so you can't connect to comms. 

So today, to support comms on these different renderers you either have to wrap everything in Jupyter Widgets or create specialized renderers for each of these targets.

## Requirements

So here is a list of requirements I would have for any cross-implementation MIME rendering thingy:

1. Supports comms (including changing kernels, no kernel available)
2. Support multiple instances of one output (happens in JupyterLab when you open an output in a new tab. Now you have it mounted in two nodes). If your renderer desires it, it should be able to syncronize state between these outputs without needing to use comms or globals.
3. Support data update (it should be able to pick up on updates to the mime data, without a full re-render)


And in terms of hosting it should be able to be run in both:

1. prebuilt mode, like JupyterLab is today, where you already have the renderer loaded
2. Dynamic mode, like Jupyter Widgets, to pull new renderers from a CDN.


## Spec

So what I see here is at the core, for a mime render extension author, you should write a function that looks like this:


```typescript

type Data = {[key: string]: object};

type CommMessage {
  readonly data: Data;
  readonly buffers?: (ArrayBuffer | ArrayBufferView)[];
}

type Comm = {
    send(data: CommMessage): Promise<void>;
    close(data: CommMessage): Promise<void>;
    // Iterator ends on close
    readonly msgs: AsyncIterable<CommMessage>
}


type ExtendableEvent {
  // call this when you are done processing this event, if it is not processesd synchronously
  waitUntil(promise: Promise<void>): void;
}

/**
 * An update about the display data.
 * The `dataUpdated` and `metadataUpdated` are a mapping of keys to values which
 * are updates for the keys in the data and metadata.
 * The `dataDeleted` and `metadataDeleted` are keys that are deleted from them.
 * A key that is deleted should not be in the updated mappings.
 */
type OutputEvent extends ExtendableEvent {
  readonly dataUpdated?: Data;
  readonly dataDeleted?: Array<string>;
  readonly metadataUpdated?: Data;
  readonly metadataDeleted?: Array<string>;
}

/**
 * An update about the nodes you have access to.
 * The new nodes will be the set of current nodes, minus the removed nodes, plus the added nodes.
 * A node should not be in the added and removed arrays.
 */
type NodeEvent extends ExtendableEvent {
  added?: Array<Element>;
  removed?: Array<Element>;
}

type CleanupFn = () => Promise<void>;

/**
 * Renders the `data` to the `node`.
 * 
 * Optionally returns a cleanup function that is called to let it try to clean up when it's done rendering.
 */
type RenderFn extends ExtendableEvent = (options: {
    output: {
      // mime bundle
      data: Data,
      metadata: Data,
    }
    // calling this function will given you the changes in the output
    // If you don't call it, your function will be re-rendered on every change with the new output
    listenOutputEvents: () => AsyncIterable<OutputEvent>,
    // make a change to the output
    emitOutputEvent: (event: OutputEvent) => void,
    // The initial node for rendering
    node: Element,
    
    
    // Use this function if you want to synchronize state between multiple views of this renderer.
    // The iterable will be updated whenever this output is mounted on a new node, like in JupyterLab
    // when you click "Create New View for Output"
    //
    // If you don't call before a new node is mounted, then this function is called again for the new node
    // and rendered independently.
    listenNodeEvents: () => AsyncIterable<NodeEvent>
    // will resolve to error if not connected to kernel
    createComm(targetName: string, data: object): Promise<Comm>;
}) => (undefined | CleanupFn)>
```

## MIME Type

Now that we have this function signature, as someone who is authoring a rendering library (like ipywidgets, plotly, etc)
you would implement this function. Now if we were just targeting JupyterLab, we could make a helper library that injests
your function adds adds a MIME Renderer for your plugin. But that doesn't help us with our other platforms. So we need
a platform agnostic way of signalling, in an output, which renderer you would like to use.

So we could use a new MIME type called (tentatively) `application/vnd.jupyter.renderer-alpha` that should have as it's data
a URL that points to a JS module with a default export of this function type. So your mime bundle might look like:
So I propose a new `mimeType` called `application/vnd.jupyter.renderer-alpha` that should :

```json
{
   "application/vnd.jupyter.renderer-alpha": "http://cdn.com/my-library.js",
   "some-other-mimetype": "whatever-data"
}
```

## JupyterLab

In JupyterLab, we can build the render of this mime type to also allow extension, so that if you want to build JupyterLab with this renderer "pre-built" you can,
in which case it won't fetch the package. We can do this by allowing you to register some function that corresponds to a URL to say use this function instead of fetching that es6 module URL.


## Sample Implementations

* [JupyterLab](https://github.com/blois/js-module-renderer)

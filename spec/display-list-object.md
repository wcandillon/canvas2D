Display List Object (DLO)
=========================

> This explainer is an incomplete working draft.

**Status**: explainer.

_A format and API for efficiently creating, transmitting, modifying and rendering UI and graphics on 2D Canvas._


Rationale
---------

HTML and the DOM provide a format and API respectively to manipulate content using high level abstractions. In the graphics sense, these abstractions operate in "retained mode."

In contrast, HTML Canvas provides APIs to draw directly to logical and physical surfaces. These abstractions operate in "immediate mode."

Retained mode has several benefits over immediate mode:

* **Accessibility**: retained mode graphics can be inspected by the platform and exposed to accessibility tools
* **Faster loads of initial UI state**: initial state of an application for certain common display sizes can be serialized, cached, streamed and displayed quickly
* **Faster updates of displayed state**: implementation can calculate minimal deltas and apply them more efficiently (e.g. using GPU-accelerated code)
* **Indexability**: retained mode text can be exposed to Web crawlers and search engines

Display List Object (DLO) is a proposal to add a retained mode API to the low level drawing abstraction provided by HTML Canvas.

Use cases
---------

### Accessibility, Testability, Indexability

Currently, applications drawing text to Canvas are inaccessible to browsers, extensions and tools.

A DLO allows applications to present graphics and text to the implementation in a retained mode object that can be inspected by the application, by related code (e.g. testing frameworks), by extensions, and by web services (e.g. search engine crawlers).

### Incremental updates and animation

Currently, applications animating graphics need to maintain display lists in user space, apply updates per frame, and redraw each frame to Canvas. Doing this work in JavaScript can be CPU intensive. Since new frames are drawn from user space, implementations must assume that changes could be made anywhere in the scene, making potential pipelines optimizations brittle and the resulting performance unpredictable.

A DLO allows the user space application to describe a scene, draw it, update only the parts of the DLO that need to be changed, and redraw it for the next frame. The implementation is able to optimize the update in various ways suitable to its pipeline.

This approach unburdens JavaScript execution, reduces call pressure along the API boundary and provides the opportunity for engines to support more complex graphics and animations at higher frame rates.

Requirements
------------

A retained mode Canvas should provide the following features:

* **Legible text**: text should be programmatically inspectable in human-understandable spans like formatted multi-line paragraphs (not glyphs, words or other fragments) without the need for OCR
* **Styled text**: applied text styles should be programmatically inspectable (e.g. size, bold, etc.)
* **Fast**: updating a retained mode Canvas should scale proportional to the size of the update, not the size of the display list
* **Inexpensive**: display lists should not consume backing memory for raster storage unless and until needed (e.g. when a raster image is requested or when a drawing is displayed on screen)
* **Scalable**: scaling a retained mode Canvas does not produce pixelation artifacts like with the current immediate mode Canvas
* **Incrementally adoptable**: applications using the current Canvas APIs should be able to gradually migrate to using a retained mode Canvas

Strawman Proposal
-----------------

_(this strawman is only to aid in understanding the above requirements; significant changes to the below are expected)_

We propose a format and data structure pair (similar to HTML and the DOM) for low level drawing primitives.

### Container

Programmatically, a DLO can be used with a Canvas context of type `2dRetained`:

```js
const canvas = document.getElementById("my-canvas-element");
const ctx = canvas.getContext("2dRetained");
```

A `2dRetained` context type is a drop-in replacement for the current `2d` context type and supports the same drawing methods.

> _**Why**: A drop-in replacement context type (actually a superset) allows applications to incrementally adopt retained mode Canvas._

### Reading a DLO

A DLO can be obtained from a Canvas `2dRetained` context:

```js
dlo = ctx.getDisplayList();
```

The DLO contains a capture of the drawing commands issued to the context since its creation (or since the last call to `reset()`) as a JSON payload:

```js
{
    "metadata": {
        "version": "0.0.1"
    },
    "commands": [
        // ...
    ]
}
```

### Drawing and updating a Canvas with a DLO

A Canvas `2dRetained` context can draw a DLO directly:

```js
ctx.drawDisplayList(dlo);
```

Drawing a DLO applies the commands in the DLO on top of existing elements already drawn to the context. In other words, it appends the commands in the DLO into the internal command list of the context.

> _**Why**: The append behavior of `drawDisplayList` aids in incremental adoption: applications can draw some parts of their scene with unmodified code that calls `ctx.draw*()` methods directly, while updated code draws other parts of the scene into a DLO which is then appended to the same context. The application can be updated over time to draw more of the scene into the DLO and issue fewer draw commands to the context._

A Canvas context of type `2dRetained` can be entirely _updated_ so that it matches a given DLO:

```js
ctx.updateDisplayList(dlo);
```

> _**Why**: The replacement behavior of `updateDisplayList` allows applications that do all drawing for a given context into a DLO to get maximum performance by presenting the desired DLO in its entirety to the implementation. The implementation can then efficiently determine and apply the needed updates to the context._

### DLO handles

Drawing against a handle allows the application to later modify certain commands in the DLO:

```js
rectHandle = ctx.withHandle("rectHandle").strokeRect(50, 50, 50, 50);
```

Handle IDs are propagated in the serialized form of the DLO as an index into the commands array:

```js
{
    "metadata": {
        "version": "0.0.1"
    },
    "commands": [
        ["strokeRect", 50, 50, 50, 50],
    ],
    "handles": [
        {"id": "rectHandle", "index": 0}
    ]
}
```

Handles can be retained from the original draw call as above, or obtained from a DLO by ID:

```js
rectHandle = ctx.getHandle("rectHandle");
```

Handles can be used to query the DLO command and update it:

```js
rectHandle.getCommand(); // ["strokeRect", 50, 50, 50, 50]
rectHandle.update(100, 100, 100, 100);
rectHandle.getCommand(); // ["strokeRect", 100, 100, 100, 100]
```

The `update` method removes the command replaces it with the equivalent command with the new arguments in the DLO. Updates do not propagate to a Canvas context until the DLO is drawn on the context or updated into the context as above.

> _**Why**: "Out of band" handles like this allow applications to modify DLOs with memory and performance that scales with the size of the anticipated updates rather than with the size of the DLO. This enables applications to update very large and complex scenes without needing to traverse the DLO for lookups or store indexes into the full DLO. Handles also allow the implementation to more quickly parse the information it needs to draw a display list, without interposing IDs or other metadata into the command list._

### Text

Drawing text is one of the main reasons to use a DLO as it allows the implementation to retain text in a Canvas for accessibility and indexability purposes.

```js
ctx.strokeText("Hello World", 10, 50);
```

The resulting DLO retains the text passed to the context:

```js
{
    "metadata": {
        "version": "0.0.1"
    },
    "commands": [
        ["strokeText", "Hello World", 10, 50],
    ]
}
```

> _**Why**: Drawing text into a DLO allows that text to be accessed later by the application, extensions and the implementation, improving the accessibility of Canvas-based applications._

### Formatted Text

Applications drawing text to a Canvas often apply their own layout rules (e.g. a document editor wrapping text at some document-defined page margin). To do this, applications need to know the dimensions of formatted text under some constraints, as well as apply line- and word-breaking according to language-specific rules.

This proposal is meant to interoperate with the [WICG Canvas Formatted Text proposal](https://github.com/WICG/canvas-formatted-text) for handling formatted text. An application would usually create a formatted text metrics object, inspect the resulting dimensions to make application-specific layout decisions, and then draw the (possubly adjusted) text to a Canvas.

```js
ftx = FormattedText.format( [   "The quick ", 
                                {
                                    text: "brown",
                                    style: "color: brown; font-weight: bold"
                                },
                                " fox jumps over the lazy dog." 
                            ], "font-style: italic", 350 );

// inspect ftxt to make layout decisions, adjust text as needed
ctx.drawFormattedText(ftxt, 50, 50 );
```

The corresponding DLO in JSON format is:

```js
{
     "metadata": {
        "version": "0.0.1"
    },
    "commands": [
        [
            "drawFormattedText", [
                "format",
                [
                    "The quick ",
                    {
                        "text": "brown",
                        "style": "color: brown; font-weight: bold"
                    },
                    " fox jumps over the lazy dog."
                ],
                "font-style: italic",
                350,
            ],
            50,
            50
        ],
    ]
}
```

Applications will typically make their layout decisions before serializing a DLO to JSON. E.g. an application may produce a DLO for common Canvas sizes, serialize and store them, and draw them directly for faster startup.

> _**Why**: As above, drawing formatted text makes the text and its associated style information available to the application, extensions and the implementation, improving the accessibility of Canvas-based applications._

Resources
---------

* [WICG Canvas Formatted Text proposal](https://github.com/WICG/canvas-formatted-text)
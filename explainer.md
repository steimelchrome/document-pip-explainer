# Document Picture-in-Picture Explained

2022-06-09

## What's all this then?

There currently exists a Web API for putting an `HTMLVideoElement` into a
Picture-in-Picture window (`HTMLVideoElement.requestPictureInPicture()`). This
limits a website's ability to provide a custom picture-in-picture experience
(PiP). We want to expand upon that functionality with a new method on the
`Window` object (`window.requestPictureInPictureWindow()`) which opens a
picture-in-picture (i.e., always-on-top) window with a blank document that can
be populated with arbitrary `HTMLElement`s instead of only a single
`HTMLVideoElement`.

This new window will be much like a blank same-origin window opened via the
existing `window.open()` API, with some limitations for security reasons:

- The PiP window will never outlive the opening window.
- The PiP window can only be opened as a response to a user gesture.
- The website cannot set the position of the PiP window.
- The PiP window cannot be navigated (any `window.history` or `window.location`
  calls that change to a new document will close the PiP window).
- The PiP window cannot open more windows.
- The PiP window must be populated via JS (i.e., cannot be loaded via URL).
- The UA can restrict the size of the PiP window.
- The UA can restrict input on the window.

### Proposed spec

```webidl
// We will add a new |requestPictureInPictureWindow()| method that opens an
// always-on-top window with the given aspect ratio (subject to restrictions
// by the UA) and an option to force the window to keep its original aspect
// ratio when resizing.
// We will also add events for detecting when picture-in-picture is entered
// or exited.
partial interface Window {
  Promise<PictureInPictureWindow> requestPictureInPictureWindow(
      PictureInPictureWindowOptions options);
  attribute EventHandler onenterpictureinpicture;
  attribute EventHandler onleavepictureinpicture;
};

dictionary PictureInPictureWindowOptions {
  // An initial aspect ratio of 0.0 implies that the website does not care to
  // set an initial aspect ratio and the UA can determine a size.
  float initialAspectRatio = 0.0;
  boolean lockAspectRatio = false;
};

// The current PictureInPictureWindow object has no document accessor (since
// it wasn't a window with a Document before), and has accessors for current
// size and events for resizing (which are not needed in document PiP since the
// website has access to the window object itself). Therefore we will add a new
// DocumentPictureInPictureWindow object instead of updating the existing one.
// This has a Document accessor so the website can use it to populate the PiP
// window, along with the aspect ratio options so they can be updated by the
// website if, e.g., a new video is loaded with a different aspect ratio.
interface DocumentPictureInPictureWindow {
  readonly attribute Document? document;
  Promise<void> setAspectRatio(float aspectRatio);
  boolean lockAspectRatio;
};
```

### Goals

- Allow a website to display arbitrary `HTMLElements` in an always-on-top
  window.
- To be simple for web developers to use and understand. Note that while
  allowing websites to call `requestPictureInPicture()` on any element would be
  the simplest way, for reasons described below, this isn't feasible.

### Non-goals

- This API is not attempting to handle placeholder content for elements that are
  moved out of the page (that is the responsibility of the website to handle).
- Allowing websites to open always-on-top widgets that outlive the webpage (the
  PiP window will close when the webpage is closed).

## Example code

### HTML

```hhtml
<body>
  <div id="player-container">
    <div id="player">
      <video id="video" src="foo.webm"></video>
      <!-- More player elements here. -->
    </div>
  </div>
  <input type="button" onclick="enterPiP();" value="Enter PiP" />
</body>
```

### JavaScript

```js
// Handle to the picture-in-picture window.
let pipWindow = null;

function enterPiP() {
  const player = document.querySelector('#player');

  const pipOptions = {
    initialAspectRatio: player.clientWidth / player.clientHeight,
    lockAspectRatio: true,
  };

  window.requestPictureInPictureWindow(pipOptions).then((_pipWin) => {
    pipWindow = _pipWin;

    // Style remaining container to imply the player is in PiP.
    playerContainer.classList.add('pip-mode');

    // Add styles to the PiP window.
    const styleLink = document.createElement('link');
    styleLink.href = 'pip.css';
    styleLink.rel = 'stylesheet';
    const pipBody = pipWindow.document.body;
    pipBody.append(styleLink);

    // Add player to the PiP window.
    pipBody.append(player);

    // Listen for the PiP closing event to put the video back.
    window.addEventListener('leavepictureinpicture', onLeavePiP, { once: true });
  });
}

// Called when the PiP window has closed.
function onLeavePiP(event) {
  // Remove PiP styling from the container.
  const playerContainer = document.querySelector('#player-container');
  playerContainer.classList.remove('pip-mode');

  // Add the player back to the main window.
  const player = event.pictureInPictureWindow.document.querySelector('#player');
  playerContainer.append(player);

  pipWindow = null;
}
```

## Key scenarios

### Accessing elements on the PiP window

The document attribute provides access to the DOM of the
`PictureInPictureWindow` object:

```js
const video = pipWindow.document.querySelector('#video');
video.loop = true;
```

### Listening to events on the PiP window

As part of creating an improved picture-in-picture experience, websites will
often want customize buttons and controls that need to respond to user input
events such as clicks.

```js
const video = pipWindow.document.querySelector('#video');
const muteButton = pipWindow.document.createElement('button');
muteButton.textContent = 'Toggle mute';
muteButton.addEventListener('click', () => {
  video.muted = !video.muted;
});
pipWindow.document.body.append(muteButton);
```

### Exiting PiP

The website may decide to close the `PictureInPictureWindow` themselves. The
existing `HTMLVideoElement` PiP functionality uses the `exitPictureInPicture()`
method on the `Document` object to accomplish this. For consistency, we will
still use this method to close the PiP window:

```js
// This will close the PiP window and trigger our existing onLeavePiP()
// listener.
document.exitPictureInPicture();
```

### Changing aspect ratio

Sometimes the website will want to change the aspect ratio after the PiP window
is open (e.g., because a new video is playing with a different aspect ratio).
The website can change it via the `aspectRatio` attribute on the
`PictureInPictureWindow`:

```js
const newVideo = document.createElement('video');
newVideo.id = 'video';
newVideo.src = 'newvideo.webm';
newVideo.addEventListener('loadedmetadata', async (_) => {
  const aspectRatio = newVideo.videoWidth / newVideo.videoHeight;
  const player = pipWindow.document.querySelector('#player');
  const oldVideo = pipWindow.document.querySelector('#video');
  player.remove(oldVideo);
  player.append(newVideo);
  await pipWindow.setAspectRatio(aspectRatio);
});
newVideo.load();
```

### Getting elements out of the PiP window when it closes

When the PiP window is closed for any reason (either because the website
initiated it or the user closed it), the website will often want to get the
elements back out of the PiP window. The website can perform this in an event
handler for the `onleavepictureinpicture` event on the `Window` object. The UA
must guarantee that the PiP document will live long enough for the event
handlers to be run. This is shown in the `onLeavePiP()` handler in the
[Example code](#example-code) section above and is copied below:

```js
// Called when the PiP window has closed.
function onLeavePiP(event) {
  // Remove PiP styling from the container.
  const playerContainer = document.querySelector('#player-container');
  playerContainer.classList.remove('pip-mode');

  // Add the player back to the main window.
  const player = event.pictureInPictureWindow.document.querySelector('#player');
  playerContainer.append(player);
}
```

## Detailed design discussion

### Why not extend the `HTMLVideoElement.requestPictureInPicture()` idea to allow it to be called on any `HTMLElement`?

Any API where the UA is taking elements out of the page and then reinserting
them ends up with tricky questions on what to show in the current document when
those elements are gone (do elements shift around? Is there a placeholder? What
magic needs to happen when things resize? etc). By leaving it up to websites to
move their own elements, the API contract between the UA and website is much
clearer and simpler to understand.

### Since this is pretty close to `window.open()`, why not just add an `alwaysOnTop` flag to `window.open()`?

The main reason we decided to have a completely separate API is to make it
easier for websites to detect it (since in most cases, falling back to a
standard window would be undesirable and websites would rather use
`HTMLVideoElement` PiP instead). Additionally, it also works differently enough
from `window.open()` (e.g., never outliving the opener) that having it separate
makes sense.

### Why not give the website more control over the size/position of the window?

Giving websites less control over the size/position of the window will help
prevent, e.g., phishing attacks where a website pops a small always-on-top
window over an `input` element to steal your password.

## Considered alternatives

Surface Element was a proposal where the website would wrap PiP-able content in
advance with a new type of iframe-like element that could be pulled out into a
separate window when requested. This had some downsides including always
requiring the overhead of a separate document (even in the most common case of
never entering picture-in-picture).

We also considered a similar approach to the one in this document, but with no
input allowed in the DOM (only allowlisted controls from a predetermined list in
a similar fashion to the existing `HTMLVideoElement` PiP). One issue with this
approach is that it really didn't help websites do much more than they already
can today, since a website can draw anything in a canvas element and PiP a video
with the canvas as a source. `Having HTMLElements` that can actually be
interacted with is what makes the Document Picture-in-Picture feature worth
implementing.

## References and acknowledgements

Many thanks to Frank Liberato, Mark Foltz, Klaus Weidner, François Beaufort,
Charlie Reis, and Joe DeBlasio for their comments and contributions to this
document and to the discussions that have informed it.

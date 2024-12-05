This proposal is an early design sketch by ChromeOS team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

# Explainer: `navigator.clipboard.contentsID()`

## Authors:

- [Luke Klimek](mailto:zgroza@chromium.org)

## Participate

- [https://crbug.com/382270770](https://crbug.com/382270770)
- [https://github.com/explainers-by-googlers/clipboard-contents-id/issues](https://github.com/explainers-by-googlers/clipboard-contents-id/issues)

## Table of Contents

- [Introduction](#introduction)
  - [Why a new thing, aren’t other clipboard APIs enough?](#why-a-new-thing-arent-other-clipboard-apis-enough)
  - [What is the optimal solution then?](#what-is-the-optimal-solution-then)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Token stability across tabs or app windows](#token-stability-across-tabs-or-app-windows)
- [How to use it?](#how-to-use-it)
- [Security considerations](#security-considerations)
- [Alternatives](#alternatives)
  - [Functionality itself](#functionality-itself)
  - [Format of the token](#format-of-the-token)
- [References & acknowledgements](#references--acknowledgements)

## Introduction

### Why a new thing, aren’t other clipboard APIs enough?

In short, without this there's no efficient way to detect clipboard changes.
To elaborate, let's consider a common use case: Virtual Desktop Infrastructure (VDI). Many Clipboard API use cases within VDI environments center around synchronizing the local clipboard with a remote machine, so that:

1. When a user copies something locally outside the VDI app and then switches to it, the new clipboard contents are seamlessly available in the remote session.
2. When a user copies something on the remote machine and switches away from the VDI app, they can paste the copied content locally.

Without `contentsID()`, there are two primary ways to achieve the first scenario:

* Upon refocusing the VDI app, automatically send the content from the local clipboard to the remote machine.
* Upon refocusing the VDI app, read the clipboard contents, compare them with the last known state, and send to the remote machine only if they have changed.

Neither of these approaches is optimal (especially with large clipboard contents), and additional challenges related to sanitization and encoding make it difficult to directly compare the clipboard contents byte-by-byte with previously received data.

### What is the optimal solution then?

Several platforms (ex. [MacOS](https://developer.apple.com/documentation/uikit/uipasteboard/1622103-changecount?language=objc), [Windows](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-addclipboardformatlistener), [X11](https://source.chromium.org/chromium/chromium/src/+/main:ui/base/x/x11_clipboard_helper.cc;drc=d815f515138991af2aa5b1d07c64906fd8a7366b;bpv=1;bpt=1;l=68?gsn=SelectionChangeObserver&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Dc%252B%252B%3Fpath%3Dui%2Fbase%2Fx%2Fx11_clipboard_helper.cc%238ndnC55hoYsX0PuoXruTyg4VFTFux3LU_qg9KPKIcTE) and [Wayland](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_data_device.cc;drc=d815f515138991af2aa5b1d07c64906fd8a7366b;bpv=1;bpt=1;l=182?gsn=OnSelection&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Dc%252B%252B%3Fpath%3Dui%2Fozone%2Fplatform%2Fwayland%2Fhost%2Fwayland_data_device.cc%23KBIABXwYhD42mocIlezMjghFMtoChm0IKDja7p09J9o), [Android](https://developer.android.com/reference/android/content/ClipboardManager.OnPrimaryClipChangedListener)) offer efficient ways to track clipboard content changes without directly reading the data. This is often achieved through clipboard sequence numbers or change notifications. The `navigator.clipboard.contentsID()` API aims to leverage these capabilities. It allows websites to request a numeric token (a 128-bit integer) representing the current clipboard state. If this token differs from a previously retrieved one, it indicates that the clipboard contents have changed between the two calls. Importantly, this operation has a constant time complexity (O(1)), independent of the clipboard's size. Therefore, even frequent checks (e.g., on window refocus) remain efficient, even when dealing with large amounts of copied data.

## Goals

* Provide a way to check if the clipboard changed between two points in time that is:
  * Easy to use
  * Efficient, no matter how big the clipboard contents are
  * Usable across multiple windows/tabs under one browser process
* Improve potential current heuristics for clipboard synchronization…

## Non-goals

* …without providing a new fingerprinting surface.

## Token stability across tabs or app windows

One of the goals of this API is to enable cross-app synchronization of clipboard \- so this should be as close to the stability of the clipboard itself as possible. So, every site under the same browser process should get the same token from calling `contentsID()`.

## How to use it?

Frankly, quite straightforwardly. Signature of the method will look somewhat like this:

```javascript
Promise<BigInt> contentsID();
```

So in the mentioned VDI case, the code could look somewhat like this:

```javascript
var lastToken = null;

// Handler called on every window refocus.
// It checks if it's necessary to sync clipboard contents to remote.
window.addEventListener("focus", () => {
  navigator.clipboard.contentsID().then(token => {
    if (token !== lastToken) {
      // Clipboard contents have changed!
      // Send to remote machine
    }
    lastToken = token;
  });
});

// Function that is called by the client app when user copies something on remote.
async function onRemoteClipboardChanged(remoteClipboardItems) {
  await navigator.clipboard.write(remoteClipboardItems);
  lastToken = await navigator.clipboard.contentsID();
}
```

Then, all that remains is to call `onRemoteClipboardChanged` every time the clipboard changes remotely \- and provided that no changes occur locally while the window is in focus (which is usually the case, as clipboard changes mostly occur due to user actions \- especially in case of local clipboard and VDI), clipboard synchronization will look seamless.
In the unfortunate case of anticipated local changes to the clipboard done in the background, this can be improved in two ways:

* Regular polling of the token and invoking a similar handler to the `focus` [handler](?tab=t.0#bookmark=id.vsbj524ib3z1) in the snippet above: this is generally not the best solution, but this API should be lightweight enough that it doesn’t create much overhead.
* Integrating this with `clipboardchange` event in addition (or instead) or the `focus` event: this depends on whether `clipboardchange` event becomes a part of the web standard.

Both however would require some synchronization of the handler and `onRemoteClipboardChanged` to prevent handlers getting between `write` and `contentsID`.

**Note:** In any case, this will be in some degree prone to inherent race conditions due to lack of clipboard atomic operations \- which will show themselves mostly in case of user switching apps very rapidly. This API exists in order to enable heuristics to make this invisible in most cases, but will not fix it completely.

## Security considerations

This should be under the same restrictions as the `navigator.clipboard.read()`:

* It should require `clipboard-read` permissions and request them on call.
* It should be available only while the tab has focus.

Thus, it doesn’t expose any new not-available-before security-sensitive information.
The only potential attack vector would be correlating different sessions with the same user based on the token, which provides a more precise way of ensuring across sessions that those to clipboards are in fact the same user. In practice however, this could be done by just re-reading the clipboard contents and comparing them, especially across changes \- which is possible already. Correlating users across sites by the origins that have clipboard permissions is already trivially easy and existence of this API does not change this state significantly.

## Alternatives

### Functionality itself

There is another proposed API for tracking clipboard changes \- a `clipboardchange` event. However, even if implemented and standardized, it operates differently. Instead of determining if a change has occurred between two points in time, it provides real-time notifications for every change, without detailed information about the cause. Therefore, if your app also writes to the clipboard, it can be challenging to determine whether you or another source caused the change (especially with multiple windows/tabs of the same app open), potentially leading to unnecessary data transfers or having to implement comparison anyway.
In case of `contentsID()`, you can save the new token just after writing \- and it will be irrelevant for all active tabs/windows irrelevant what caused the change, only that this change is already in sync with the remote and no action is needed.

### Format of the token

There are several ways in which the token could look like, including:

1. Sequence number that would increase with each change (or with each call that detected a change)
2. Timestamp of the last change (or call that detected it)
3. Hash of the clipboard contents
4. Random 128-bit number without any specified scheme or significance \- other than “after something is written to the clipboard, `contentsID()` should yield a different value than it did before the write”

Preferred approach is 4, for the following reasons:

* It doesn’t provide any information about the user’s action other than already available
* Randomness of this degree is enough to ensure the lack of false positives, conforming with [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) standards
* It’s implementationally and computationally the simplest
* It’s the simplest solution that is sufficient for the provided use case
* It’s trivial to compare and store

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [Andrew Rayskiy](mailto:greengrape@google.com)
- [Ayu Ishii](mailto:ayui@chromium.org)
- [Dominik Bylica](mailto:bylica@google.com)
- [Robert Ferens](mailto:rferens@google.com)

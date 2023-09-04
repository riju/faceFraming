# Face Framing

## Authors:

- Rijubrata Bhaumik, Intel Corporation
- Eero Häkkinen, Intel Corporation
- Youenn Fablet, Apple Inc.

## Participate
- github.com/riju/faceFraming/issues/
- [Spec](https://w3c.github.io/mediacapture-extensions/#exposing-mediastreamtrack-source-automatic-face-framing-support)

## Introduction

Video conferencing has become an integral part of communication for both personal and professional purposes. With the rise of remote work and virtual interactions, the need for seamless and engaging video conferences has never been more crucial. One of the technological advancements making waves in this realm is AI auto framing of faces. We want to achieve the goal of giving all the Web apps similar levers as their native counterparts, leveraging the same platform APIs and to delight users without completely relying on ML frameworks like TensorFlow.js, [Mediapipe](https://ai.googleblog.com/2020/10/background-features-in-google-meet.html), etc, or cloud based solutions.

## Use Cases

A vast majority of communication these days happens on our client devices.  

Auto framing takes the hassle out of adjusting the camera angle or manually positioning yourself during a video call. This technology automatically detects and frames participants' faces within the video feed, ensuring that everyone is clearly visible and centered. This feature eliminates distractions caused by awkward camera angles or participants being out of frame, leading to a more immersive and enjoyable user experience.

In a professional setting, auto framing ensures that you appear composed and well-prepared at all times and allows you to focus on the content of your presentation rather than worrying about adjusting their camera angle or distance from the screen.

Auto face framing can help ensure that participants with mobility challenges or those who use assistive devices are consistently in the frame.

On the Web, due to a lack of a standardized JS API for Face Framing and widespread demand, developers have no options but to use ML frameworks like Tensorflow.js and other WASM libraries to satify their customers. This Face Framing API gives developers a **choice** to use the native platform's API. This would ensure conformance to the corresponding native apps.


## Goals

* Ideally, Face Framing API should offer a good developer experience like a simple boolean with getUserMedia. We have PTZ controls for the Web and proposals for Face Detection feature, but automatic Face Framing is like an AI powered option, where faces are detected and centered without user intervention.

## Non-goals


## Performance

On Windows and Mac platforms, Face Framing would leverage NPU (Neural Processing Unit). Windows 11 and a explicit NPU/ VPU might be a requirement for some implementation to guarantee a level of user experience expected by users of modern computing platforms.

## User research

* TEAMS : 

* MEET : 

* Zoom : 

## Face Framing API

This is about bringing platform automatic face framing to the web so constrainable media track capabilities fit naturally to the purpose. Because the face framing is (or should be) implemented by the platform media pipeline, it is enough for an (web) application to control the face framing through constrainable properties. On Apple devices, Face framing is controlled globally from Command Center and as such any app cannot independently switch on/off the feature. 

```js
  partial dictionary MediaTrackSupportedConstraints {
    boolean faceFraming = true;
  };

  partial dictionary MediaTrackCapabilities {
    sequence<boolean> faceFraming;
  };

  partial dictionary MediaTrackConstraintSet {
    ConstrainBoolean faceFraming;
  };

  partial dictionary MediaTrackSettings {
    boolean faceFraming;
  };
  ```

[Spec](https://w3c.github.io/mediacapture-extensions/#exposing-mediastreamtrack-source-automatic-face-framing-support)


## Exposing change of MediaStreamTrack configuration

The configuration (capabilities, constraints or settings) of a MediaStreamTrack may be changed dynamically outside the control of web applications.
One example is when a user decides to switch on face framing through the operating system.
Web applications might want to know that the configuration of a particular MediaStreamTrack has changed.
For that purpose, a new event is defined below.

```js
partial interface MediaStreamTrack {
  attribute EventHandler onconfigurationchange;
};
```

[PR](https://github.com/w3c/mediacapture-extensions/pull/61)

## Example

 * main.js:
   ```js
   // main.js:
   // Open camera.
   const stream = await navigator.mediaDevices.getUserMedia({video: true});
   const [videoTrack] = stream.getVideoTracks();

   // Use a video worker and show to user.
   const videoElement = document.querySelector('video');
   const videoWorker = new Worker('video-worker.js');
   videoWorker.postMessage({videoTrack}, [videoTrack]);
   const {data} = await new Promise(r => videoWorker.onmessage);
   videoElement.srcObject = new MediaStream([data.videoTrack]);
   ```
 * video-worker.js:
   ```js
   self.onmessage = async ({data: {videoTrack}}) => {
     const processor = new MediaStreamTrackProcessor({track: videoTrack});
     let readable = processor.readable;

     const videoCapabilities = videoTrack.getCapabilities();
     if ((videoCapabilities.faceFraming || []).includes(true)) {
       // The platform supports face framing and
       // allows it to be enabled.
       // Let's try use platform face framing.
       await track.applyConstraints({faceFraming: true});
     }
     const videoSettings = videoTrack.getSettings();
     if (videoSettings.faceFraming) {
       // Face framing is enabled.
       // No transformer streams are needed.
       // Pass the original track back to the main.
       parent.postMessage({videoTrack}, [videoTrack]);
     } else {
       // The platform does not support face framing or
       // does not allow it to be enabled.
       // Let's use custom face detection to aid custom face framing.
       importScripts('custom-face-detection.js', 'custom-background-face-framing.js');
       const transformer = new TransformStream({
         async transform(frame, controller) {
           // Use a custom face detection.
           const detectedFaces = await detectFaces(frame);
           // Use a custom face framing.
           const newFrame = await faceFraming(frame, detectedFaces);
           frame.close();
           controller.enqueue(newFrame);
         }
       });
       // Transformer streams are needed.
       // Use a generator to generate a new video track and pass it to the main.
       const generator = new VideoTrackGenerator();
       parent.postMessage({videoTrack: generator.track}, [generator.track]);
       // Pipe through a custom transformer.
       await readable.pipeThrough(transformer).pipeTo(generator.writable);
     }
   };
   ```

## Security considerations

The face framing capability (a boolean sequence describing whether it is possible to enable and/or to disable the face framing) is provided by the User-Agent and cannot be modified by web applications. It may however change for instance if the user uses operating system controls to enforce face framing or to remove such an enforcement.

The face framing constraints are provided by web applications by passing them to `navigator.mediaDevices.getUserMedia()` or to `MediaStreamTrack.applyConstraints()`. Constraints allow web applications to change settings within the bounds of capabilities.

## Privacy considerations

Automatic face framing combines face tracing, a composition algorithm, like something based on the principle of thirds, and pixel-wise super-resolution to create a portrait with the best composition from the image with the detected face. A wider Filed Of View (FOV) allows for better coverage, ensuring the webcam can track your face even if you move around. The maximum FOV is the camera's FOV, and `navigator.mediaDevices.getUserMedia()` is used to get permission for the camera access. When you move from the front of the camera to another corner of the meeting room, the camera will change the focal length to capture you in the corner and frame you in the center of the screen, provided you still fall within the camera's FOV.

### Fingerprinting

If a site does not have [permissions](https://w3c.github.io/permissions/), face framing provides practically no fingerprinting posibilities.
The only provided information is `navigator.mediaDevices.getSupportedConstraints().faceFraming` which either is true (if the User-Agent supports automatic face framing in general irrespective of whether the device has a camera or a platform version which support face framing) or does not exist.
That same information can most probably obtained also by other means like from the User-Agent string.

If a site utilizes `navigator.mediaDevices.getUserMedia({video: {}})` which resolves only after the user has [granted](https://w3c.github.io/permissions/#dfn-granted) the ["camera"](https://www.w3.org/TR/mediacapture-streams/#dfn-camera) permission, the returned video tracks may have `faceFraming` capabilities and settings.
The `faceFraming` capability can be either non-existing, `[false]`, `[true]` or `[false, true]` and the setting can be either non-existing, false or true.
Based on the capability, it is possible to determine if the platform is one which allows application only to observe face framing setting changes or one which allows applications also to set the face framing setting.
In essence, this splits operating systems to two groups but does not differentiate between platform versions.

All the frames for which automatic face framing happens, originate from cameras.
No methods are provided for sites to insert frames for face framing.
As such, sites cannot fingerprint the face framing implementation as the sites have no access to original frames and have access to frames with face centered only if the user has the user has [granted](https://w3c.github.io/permissions/#dfn-granted) the ["camera"](https://www.w3.org/TR/mediacapture-streams/#dfn-camera) permission.

## Stakeholder Feedback / Opposition

[ChromeOS camera team] As of this writing, ChromeOS supports one-shot face framing mode. Continuously moving face framing is not currently supported due to additional work required to meet accessibility and inclusion requirements.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [Bernard Aboba]
- [Harald Alvestrand]
- [Jan-Ivar Bruaroey]
- [Dominique Hazael-Massieux]
- [François Beaufort]


## Disclaimer

Intel is committed to respecting human rights and avoiding complicity in human rights abuses. See Intel's Global Human Rights Principles. Intel's products and software are intended only to be used in applications that do not cause or contribute to a violation of an internationally recognized human right.

Intel technologies may require enabled hardware, software or service activation.

No product or component can be absolutely secure.

Your costs and results may vary.

© Intel Corporation

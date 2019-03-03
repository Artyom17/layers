A feature repo for working on multi-layer support in WebXR

This is a feature repo (as defined in the proposals process) that has been created for exploring multi-layer graphics for XR.

The conversation that led to the creation of this repo can be found in [Proposals Issues #46](https://github.com/immersive-web/proposals/issues/46).

Artem Bolgar (@Artyom17) is coordinating the creation of the initial explainers.


# Layers in WebXR

Current WebXR spec is written with layers in mind (see [XRLayer interface](https://immersive-web.github.io/webxr/#xrlayer-interface)), however, only a single layer ([baseLayer](https://immersive-web.github.io/webxr/#dom-xrrenderstate-baselayer)) is currently supported.

Other VR APIs (such as soon-to-be-public-supposedly-open-api-for-XR, Oculus SDK, more? **tbd**) support so called "VR compositor layers". 

## VR Compositor Layers

Objects and textures rendered as compositor layers render at the frame rate of the compositor, the refresh rate of the HMD, instead of rendering at the application frame rate. Because of this, the compositor layers are less prone to judder and are raytraced through the lenses. This improves the clarity of the textures or text displayed on them.

Compositor layers are useful for displaying information, text, video, or textures that are intended to be focal objects in your scene. Compositor layers can also be useful for displaying simple environments and backgrounds to your scene.

### Proposed types of layers
Not all layers are going to be supported by all hardware/browsers. We would need to figure out the bare minimum of layer types to be supported. I have the following ones in mind: the transparent or opaque quadrilateral, cubemap, cylindrical and equirect
 layers.
 
 *to be continued...*



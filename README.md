### Layers in WebXR

Current WebXR spec is written with layers in mind (see [XRLayer interface](https://immersive-web.github.io/webxr/#xrlayer-interface)), however, only a single layer ([baseLayer](https://immersive-web.github.io/webxr/#dom-xrrenderstate-baselayer)) is currently supported.

Other XR APIs (such as soon-to-be-public-supposedly-open-api-for-XR, Oculus SDK, more? tbd) support so called 

### Summary

Objects and textures rendered as compositor layers render at the frame rate of the compositor, the refresh rate of the HMD, instead of rendering at the application frame rate. Because of this, the compositor layers are less prone to judder and are raytraced through the lenses, as well as textures / videos are not double sampled and sampled directly by the compositor at full resolution. This improves the clarity of the textures or text displayed on them.

### Example use cases

Compositor layers are useful for displaying information, text, video, or textures that are intended to be focal objects in your scene. Compositor layers can also be useful for displaying simple environments and backgrounds to your scene.

One of the very common use cases of WebXR is 180 and/or 360 photos or videos, both - equirect and cubemaps. Usual implementation involves a lot of CPU and GPU power and the result is usually pretty poor in terms of visual quality, latency and power consumption (the latter is especially critical for mobile / standalone devices, such as Oculus Go, Quest, Vive Focus, ML1, HoloLens, etc).

Another example where compositor layers are going to shine is displaying text or high resolution textures in the virtual world. Since compositor layers allow to sample source texture at full resolution without downsampling to the resolution of the eye buffers (which is usually much lower than the physical resolution of the device's screen(s)) the readability of the text is going to be significantly improved.
I would propose two types of compositor layers for the text - quad and cylinder. While cylinder layer is harder to implement than a simple Quad, it provides much better way to display readable text in VR space. But if the cylinder layer is not supported by the certain hardware or the browser then the Quad layer could be used.

### Implementation overview

A very high-level description of the implementation could be this:
* Replace the XRRenderState [baseLayer](https://immersive-web.github.io/webxr/#dom-xrrenderstate-baselayer) with the sequence of XRLayers;
* Introduce different subtypes to [XRLayer](https://immersive-web.github.io/webxr/#xrlayer-interface) which will have all the necessary attributes for each layer;
* Introduce a way to determine which layers are supported and what is the maximum amount of the layers supported;

### Proposed types of layers
Not all layers are going to be supported by all hardware/browsers. We would need to figure out the bare minimum of layer types to be supported. I have the following ones in mind: the transparent or opaque quadrilateral, cubemap, cylindrical and equirect
 layers.

### Stereo vs mono

### Layer image source

#### Anti-aliasing

Unfortunately, even WebGL 2 is pretty lame in terms of supporting multisampling rendering into a texture. There is no way (TODO) to render into a texture with implicit multisampling (no analog of EXT_render_to_texture_multisampled GL extension). Using multisampled renderbuffers is possible, but it involves extra copying (blitting from the renderbuffer to a texture).
To address this performance issue, I think to introduce an XRLayerFramebufferImage, that will create a framebuffer with multisampling support.

### Modify XRWebGLLayer
Having XRLayerImage concept introduced, shouldn't we modify XRWebGLLayer to use it instead of explicit reference to framebuffer or texture array (for the XRWebGLArrayLayer)?
```
dictionary XRWebGLLayerInit {
  boolean antialias = true;
  boolean depth = true;
  boolean stencil = false;
  boolean alpha = true;
  boolean multiview = false;
  double framebufferScaleFactor = 1.0;
};

[
    SecureContext,
    Exposed=Window,
    RuntimeEnabled=WebXR,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRWebGLLayerInit layerInit),
    RaisesException=Constructor
] interface XRWebGLLayer : XRLayer {
  [ImplementedAs=getXRWebGLRenderingContext] readonly attribute XRWebGLRenderingContext context;
  readonly XRLayerImage image;

  XRViewport? getViewport(XRView view);
  void requestViewportScaling(double viewportScaleFactor);

  static double getNativeFramebufferScaleFactor(XRSession session);
};
```
 
 *to be continued...*






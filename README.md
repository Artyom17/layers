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
```webidl
dictionary XRRenderStateInit {
  double depthNear;
  double depthFar;
  sequence<XRLayer>? layers;
};

[SecureContext, Exposed=Window] interface XRRenderState {
  readonly attribute double depthNear;
  readonly attribute double depthFar;
  readonly attribute sequence<XRLayer>? layers;
};
```
Example of use:
```javascript
function onXRSessionStarted(xrSession) {
  let glCanvas = document.createElement("canvas");
  let gl = glCanvas.getContext("webgl", { xrCompatible: true });

  loadWebGLResources();

  xrSession.updateRenderState({ [ new XRWebGLLayer(xrSession, gl) ] });
}
```


* Introduce different subtypes to [XRLayer](https://immersive-web.github.io/webxr/#xrlayer-interface) which will have all the necessary attributes for each layer;
* Introduce a way to determine which layers are supported and what is the maximum amount of the layers supported;

### Proposed types of layers
Not all layers are going to be supported by all hardware/browsers. We would need to figure out the bare minimum of layer types to be supported. I have the following ones in mind: the transparent or opaque quadrilateral, cubemap, cylindrical and equirect
 layers.

### Layer image source
Layers require image source that is used for rendering. In order to achieve maximum performance and to avoid extra texture copies, the image sources might be implemented as direct compositor swapchains under-the-hood. The proposed image sources are as follows:
* `XRLayerTextureImage` - the WebGLTexure is exposed, so the content can be copied into it.
* `XRLayerTextureArrayImage` - the WebGLTexture, that represents texture array is exposed, so the content can be copied into layers of it. Layer 0 represents the left eye image, 1 - the right eye image.
* `XRLayerFramebufferImage` - the opaque WebGLFramebuffer is exposed, see 'Anti-aliasing' below.

#### Stereo vs mono
Each layer could render the same image for both eyes ("mono") or different images for each eye ("stereo"). `XRLayerImageSource` contains a member of type `XRStereoLayout` that indicates the layout for stereo rendering. If the image source is used for mono rendering or for rendering only for one particular eye then `stereoLayout` should be set to `none`.
For image sources which are used for stereo layers there are three options: 
* `leftRight` - side-by-side stereo, left half represents the left eye image, the right half - the right eye image;
* `topBottom` - the top half of the image is the image for the left eye, the bottom half is the right eye image;
* `layers` - used with `XRLayerTextureArrayImage`, where layer 0 represents the left eye image, 1 - the right eye image.


#### Anti-aliasing

Unfortunately, even WebGL 2 is pretty lame in terms of supporting multisampling rendering into a texture. There is no way to render directly into a texture with implicit multisampling (`WEBGL_texture_multisample` extension is proposed, but not even in draft yet). Using multisampled renderbuffers is possible, but it involves extra copying (blitting from the renderbuffer to a texture).
To address this performance issue, I think to introduce an `XRLayerFramebufferSourceImage`, that will create an opaque framebuffer with multisampling support, similarly to what is used for `XRWebGLLayer`.

## Proposed IDL

```webidl
typedef (WebGLRenderingContext or WebGL2RenderingContext) XRWebGLRenderingContext;

dictionary XRLayerInit {
  boolean visible = true;
  boolean chromaticAberration = false;
  boolean blendTextureSourceAlpha = false;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerInit imageInit)
] interface XRLayer {
  readonly attribute boolean visible;
  readonly attribute chromaticAberration;
  readonly attribute boolean blendTextureSourceAlpha;
  
  setVisibility(boolean visible);
};

enum XRStereoLayout {
  // mono
  "none",
  // left to right side-by-side stereo
  "leftRight", 
  // left eye image is at the top, right eye image at the bottom
  "topBottom", 
  // texture array layers, 0 - left eye, 1 - right eye
  "layers" 
};

enum XRLayerEyeVisibility {
  "both",
  "left",
  "right" 
};

dictionary XRLayerImageSourceInit {
  XRStereoLayout  stereoLayout = "none",
}

// https://immersive-web.github.io/webxr/#xrwebgllayer-interface
[
    SecureContext,
    Exposed=Window
] interface XRLayerImageSource {
  readonly attribute XRStereoLayout stereoLayout;
};

/////////////////////////
dictionary XRLayerTextureImageInit : XRLayerImageSourceInit {
  boolean antialias = true;
  boolean alpha = true;
  unsigned long textureWidth;
  unsigned long textureHeight;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerTextureImageInit imageInit)
] interface XRLayerTextureImage : XRLayerImageSource {
  readonly attribute XRWebGLRenderingContext context;
  readonly attribute unsigned long textureWidth;
  readonly attribute unsigned long textureHeight;
  readonly attribute WebGLTexture texture;
};

/////////////////////////
dictionary XRLayerTextureArrayImageInit : XRLayerImageSourceInit {
  boolean antialias = true;
  boolean alpha = true;
  unsigned long arrayTextureWidth;
  unsigned long arrayTextureHeight;
  unsigned long arrayTextureDepth;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerTextureArrayImageInit imageInit)
] interface XRLayerTextureArrayImage : XRLayerImageSource {
  readonly attribute XRWebGLRenderingContext context;
  readonly attribute unsigned long arrayTextureWidth;
  readonly attribute unsigned long arrayTextureHeight;
  readonly attribute unsigned long arrayTextureDepth;
  readonly attribute WebGLTexture arrayTexture;
};

/////////////////////////
dictionary XRLayerFramebufferImageInit : XRLayerImageSourceInit {
  boolean antialias = true;
  boolean depth = true;
  boolean stencil = false;
  boolean alpha = true;
  boolean multiview = true;
  unsigned long framebufferWidth;
  unsigned long framebufferHeight;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerFramebufferImageInit imageInit)
] interface XRLayerFramebufferImage : XRLayerImageSource {
  readonly attribute XRWebGLRenderingContext context;
  readonly attribute boolean antialias;
  readonly attribute boolean depth;
  readonly attribute boolean stencil;
  readonly attribute boolean alpha;
  readonly attribute boolean multiview;

  readonly attribute unsigned long framebufferWidth;
  readonly attribute unsigned long framebufferHeight;
  readonly attribute WebGLFramebuffer framebuffer;
};

/////////////////////////
dictionary XRLayerDOMImageInit : XRLayerImageSourceInit {
  DOMString     url;
  unsigned long width  = 0;
  unsigned long height = 0;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerDOMImageInit imageInit)
] interface XRLayerDOMSource : XRLayerImageSource {
  readonly attribute DOMString     url;
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
};

/////////////////////////
dictionary XRLayerVideoSourceInit : XRLayerImageSourceInit {
  HTMLVideoElement video;
};


[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRLayerVideoSourceInit imageInit)
] interface XRLayerVideoSource : XRLayerImageSource {
  HTMLVideoElement video;
};

/////////////////////////
dictionary XRQuadLayerInit {
  XRLayerImageSource imageSource;
  XRLayerEyeVisibility eyeVisibility = "both";
  XRSpace   space;
  XRPose?   pose;
  long      width;
  long      height;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRQuadLayerInit layerInit)
] interface XRQuadLayer : XRLayer {

  readonly attribute XRLayerImageSource imageSource;

  readonly attribute XRLayerEyeVisibility eyeVisibility;
  readonly attribute XRSpace   space;
  readonly attribute XRPose?   pose;
  readonly attribute long      width;
  readonly attribute long      height;

  XRViewport? getViewport(XRView view);
};


/////////////////////////
dictionary XRCylinderLayerInit {
  XRLayerImageSource imageSource;
  XRLayerEyeVisibility eyeVisibility = "both";
  XRSpace  space;
  XRPose?  pose;
  float    radius;
  float    centralAngle;
  float    aspectRatio;
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRCylinderLayerInit layerInit)
] interface XRCylinderLayer : XRLayer {

  readonly attribute XRLayerImageSource imageSource;

  readonly attribute XRLayerEyeVisibility eyeVisibility;
  readonly attribute XRSpace  space;
  readonly attribute XRPose?  pose;
  readonly attribute float    radius;
  readonly attribute float    centralAngle;
  readonly attribute float    aspectRatio;

  XRViewport? getViewport(XRView view);
};

/////////////////////////
dictionary XREquirectLayerInit {
  XRLayerImageSource imageSource;
  XRLayerEyeVisibility eyeVisibility = "both";
  XRSpace  space;
  XRPose?  pose;
  DOMPoint offset; // x,y,z
  DOMPoint scale2d;// x,y
  DOMPoint bias2d; // x,y
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XREquirectLayerInit layerInit)
] interface XREquirectLayer : XRLayer {

  readonly attribute XRLayerImageSource imageSource;

  readonly attribute XRLayerEyeVisibility eyeVisibility;
  readonly attribute XRSpace          space;
  readonly attribute XRPose?          pose;
  readonly attribute DOMPointReadOnly offset; //3f
  readonly attribute DOMPointReadOnly scale2d;//2f
  readonly attribute DOMPointReadOnly bias2d; //2f

  XRViewport? getViewport(XRView view);
};

/////////////////////////
dictionary XRCubeLayerInit {
  XRLayerImageSource imageSource;
  XRLayerEyeVisibility eyeVisibility = "both";
  XRSpace  space;
  DOMPoint orientation; 
  DOMPoint offset; 
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRCubeLayerInit layerInit)
] interface XRCubeLayer : XRLayer {

  readonly attribute XRLayerImageSource imageSource;

  readonly attribute XRLayerEyeVisibility eyeVisibility;
  readonly attribute XRSpace  space;

  readonly attribute DOMPointReadOnly orientation; 
  readonly attribute DOMPointReadOnly offset; 

  XRViewport? getViewport(XRView view);
};

```
 
### An extra mile: modify XRWebGLLayer?
Having XRLayerSourceImage concept introduced, shouldn't we modify XRWebGLLayer to use it instead of explicit reference to framebuffer or texture array (for the XRWebGLArrayLayer)? We could avoid introducing an extra XRWebGLArrayLayer type in this case.
```webidl
dictionary XRWebGLLayerInit {
  boolean antialias = true;
  boolean depth = true;
  boolean stencil = false;
  boolean alpha = true;
  boolean multiview = false;
  double framebufferScaleFactor = 1.0;
};

dictionary XRWebGLArrayLayerInit {
  boolean alpha = true;
  double arrayTextureScaleFactor; // Same as the framebufferScaleFactor
};

[
    SecureContext,
    Exposed=Window,
    Constructor(XRSession session, XRWebGLRenderingContext context, optional XRWebGLLayerInit layerInit),
    Constructor(XRSession session, WebGL2RenderingContext context,  optional XRWebGLArrayLayerInit layerInit)
] interface XRWebGLLayer : XRLayer {
  readonly attribute XRWebGLRenderingContext context;
  readonly attribute XRLayerSourceImage imageSource;

  XRViewport? getViewport(XRView view);
  void requestViewportScaling(double viewportScaleFactor);

  static double getNativeFramebufferScaleFactor(XRSession session);
};
```








# Anti Aliasing
## Off-screen MSAA
### Multisampled texture attachments
To create a texture that supports storage of multiple sample points we use __glTexImage2DMultisample__ instead of __glTexImage2D__
```C++
glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, tex);
glTexImage2DMultisample(GL_TEXTURE_2D_MULTISAMPLE, samples, GL_RGB, width, height, GL_TRUE);
glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, 0);  
```
To attach a multisampled texture to a framebuffer we use glFramebufferTexture2D, but this time with GL_TEXTURE_2D_MULTISAMPLE as the texture type:
```C++
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D_MULTISAMPLE, tex, 0);
```
### Multisampled renderbuffer objects
Change from glRenderbufferStorage to glRenderbufferStorageMultisample when we configure the (currently bound) renderbuffer's memory storage:
```C++
glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_DEPTH24_STENCIL8, width, height);  
```
### Render to multisampled framebuffer
A multisampled image contains much more information than a normal image so what we need to do is downscale or resolve the image. 
__Resolving a multisampled framebuffer__ is generally done through __glBlitFramebuffer__ that copies a region from one framebuffer to the other while also resolving any multisampled buffers.
```C++
glBindFramebuffer(GL_READ_FRAMEBUFFER, multisampledFBO);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);
glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);
```
__!!!What if we wanted to use the texture result of a multisampled framebuffer to do stuff like post-processing?__

We can't directly use the multisampled texture(s) in the fragment shader. What we can do however is blit the multisampled buffer(s) to a different FBO with a non-multisampled texture attachment. 
```C++
unsigned int msFBO = CreateFBOWithMultiSampledAttachments();
// then create another FBO with a normal texture color attachment
[...]
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, screenTexture, 0);
[...]
while(!glfwWindowShouldClose(window))
{
    [...]
    
    glBindFramebuffer(msFBO);
    ClearFrameBuffer();
    DrawScene();
    // now resolve multisampled buffer(s) into intermediate FBO
    glBindFramebuffer(GL_READ_FRAMEBUFFER, msFBO);
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, intermediateFBO);
    glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);
    // now scene is stored as 2D texture image, so use that image for post-processing
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    ClearFramebuffer();
    glBindTexture(GL_TEXTURE_2D, screenTexture);
    DrawPostProcessingQuad();  
  
    [...] 
}
```
### Custom Anti-Aliasing algorithm
In shader, define the texture uniform sampler as a sampler2DMS instead of the usual sampler2D:
```GLSL
uniform sampler2DMS screenTextureMS;
```
Using the texelFetch function it is then possible to retrieve the color value per sample:
```GLSL
vec4 colorSample = texelFetch(screenTextureMS, TexCoords, 3);  // 4th subsample
```

# Deferred Shading
__In one word, get all geometry information first and postpone the process of computing fragments color.__

The way we did lighting so far was called forward rendering or forward shading. 
A straightforward approach where we render an object and light it according to all light sources in a scene.
Wasting a lot of fragment shader runs in scenes with a high depth complexity (multiple objects cover the same screen pixel) as fragment shader outputs are overwritten.

Deferred shading or deferred rendering aims to overcome these issues by drastically changing the way we render objects. 
Deferred shading consists of two passes: in the first pass, called the geometry pass, 
we render the scene once and retrieve all kinds of geometrical information from the objects that we store in a collection of textures called the __G-buffer__; 
think of __position vectors, color vectors, normal vectors, and/or specular values__. 
The geometric information of a scene stored in the G-buffer is then later used for (more complex) lighting calculations. 

![image](https://user-images.githubusercontent.com/98029669/214482040-324476ce-6b8b-4db6-9564-99e99b93523a.png)
![image](https://user-images.githubusercontent.com/98029669/214482083-7476c439-4b26-423a-b2f1-285d2b7cb70b.png)

A major advantage of this approach is that whatever fragment ends up in the G-buffer is the actual fragment information that ends up as a screen pixel. 
The depth test already concluded this fragment to be the last and top-most fragment.(No redundent computation for color)

## G-buffer
We have freedom of choosing what we want to store in G-buffer. Here, we store these things in out case(for blinn-phong):
1. A 3D world-space position vector to calculate the __(interpolated) fragment position variable__ used for lightDir and viewDir.  
2. An RGB __diffuse color vector__ also known as __albedo__.  
3. A 3D normal vector for determining a __surface's slope__.  
4. A __specular intensity__ float.  
5. All __light source position and color vectors__.  
6. The __player or viewer's position vector__.  

```C++
int main()
{
    ...
    // build and compile shaders
    // -------------------------
    Shader shaderGeometryPass("8.1.g_buffer.vs", "8.1.g_buffer.fs");
    Shader shaderLightingPass("8.1.deferred_shading.vs", "8.1.deferred_shading.fs");
    Shader shaderLightBox("8.1.deferred_light_box.vs", "8.1.deferred_light_box.fs");
    ...
    // configure g-buffer framebuffer(one geometry information one attachment, only one FBO and using MRT)
    // ------------------------------
    unsigned int gBuffer;
    glGenFramebuffers(1, &gBuffer);
    glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
    unsigned int gPosition, gNormal, gAlbedoSpec;
    // position color buffer
    glGenTextures(1, &gPosition);
    glBindTexture(GL_TEXTURE_2D, gPosition);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, gPosition, 0);
    // normal color buffer
    glGenTextures(1, &gNormal);
    glBindTexture(GL_TEXTURE_2D, gNormal);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT1, GL_TEXTURE_2D, gNormal, 0);
    // color + specular color buffer
    glGenTextures(1, &gAlbedoSpec);
    glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT2, GL_TEXTURE_2D, gAlbedoSpec, 0);
    // tell OpenGL which color attachments we'll use (of this framebuffer) for rendering 
    // otherwise all to first
    unsigned int attachments[3] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2 };
    glDrawBuffers(3, attachments);
    // create and attach depth buffer (renderbuffer)
    // only one depth buffer attachment is needed
    unsigned int rboDepth;
    glGenRenderbuffers(1, &rboDepth);
    glBindRenderbuffer(GL_RENDERBUFFER, rboDepth);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT, SCR_WIDTH, SCR_HEIGHT);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, rboDepth);
    ...

    // shader configuration
    // --------------------
    // Specify the MRT channel
    shaderLightingPass.use();
    shaderLightingPass.setInt("gPosition", 0);
    shaderLightingPass.setInt("gNormal", 1);
    shaderLightingPass.setInt("gAlbedoSpec", 2);

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window))
    {
        ...
        // 1. geometry pass: render scene's geometry/color data into gbuffer
        // -----------------------------------------------------------------
        glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
            shaderGeometryPass.use(); // shaderGeometryPass("8.1.g_buffer.vs", "8.1.g_buffer.fs");
            ...
        glBindFramebuffer(GL_FRAMEBUFFER, 0);

        // 2. lighting pass: calculate lighting by iterating over a screen filled quad pixel-by-pixel using the gbuffer's content.
        // -----------------------------------------------------------------------------------------------------------------------
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        shaderLightingPass.use(); // shaderLightingPass("8.1.deferred_shading.vs", "8.1.deferred_shading.fs");
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, gPosition);
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, gNormal);
        glActiveTexture(GL_TEXTURE2);
        glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
        ...
        // finally render quad, render a 2D image
        renderQuad();

        // 2.5. copy content of geometry's depth buffer to default framebuffer's depth buffer
        // ----------------------------------------------------------------------------------
        glBindFramebuffer(GL_READ_FRAMEBUFFER, gBuffer);
        glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0); // write to default framebuffer
        // blit to default framebuffer. Note that this may or may not work as the internal formats of both the FBO and default framebuffer have to match.
        // the internal formats are implementation defined. This works on all of my systems, but if it doesn't on yours you'll likely have to write to the 		
        // depth buffer in another shader stage (or somehow see to match the default framebuffer's internal format with the FBO's internal format).
        glBlitFramebuffer(0, 0, SCR_WIDTH, SCR_HEIGHT, 0, 0, SCR_WIDTH, SCR_HEIGHT, GL_DEPTH_BUFFER_BIT, GL_NEAREST);
        glBindFramebuffer(GL_FRAMEBUFFER, 0);

        // 3. render lights on top of scene
        // --------------------------------
        shaderLightBox.use(); // shaderLightBox("8.1.deferred_light_box.vs", "8.1.deferred_light_box.fs");
        ...
    }
    ...
}
```

# Bloom
One way to distinguish bright light sources on a monitor is by making them glow; the light then bleeds around the light source.
__In one word, blooming is just bluring the light sources.__

Bloom works best in combination with HDR rendering. A common misconception is that HDR is the same as Bloom as many people use the terms interchangeably. 
They are however completely different techniques used for different purposes.

![image](https://user-images.githubusercontent.com/98029669/214472567-9e94bc2b-b67f-419c-9631-9692343d18a3.png)

In this case, we have four group of shaders.
```C++
Shader shader("7.bloom.vs", "7.bloom.fs");
Shader shaderLight("7.bloom.vs", "7.light_box.fs");
Shader shaderBlur("7.blur.vs", "7.blur.fs");
Shader shaderBloomFinal("7.bloom_final.vs", "7.bloom_final.fs");
```

## Extracting Bright Color
The first step requires us to extract two images from a rendered scene. 
We could render the scene twice, both rendering to a different framebuffer with different shaders, 
but we can also use a neat little trick called __Multiple Render Targets (MRT)__ that allows us to __specify more than one fragment shader output__; 
this gives us the option to extract the first two images in a single render pass. 
```C++
int main()
{
    ...
    // build and compile shaders
    // -------------------------
    Shader shader("7.bloom.vs", "7.bloom.fs");
    Shader shaderLight("7.bloom.vs", "7.light_box.fs");
    Shader shaderBlur("7.blur.vs", "7.blur.fs");
    Shader shaderBloomFinal("7.bloom_final.vs", "7.bloom_final.fs");

    ...
    // configure (floating point) framebuffers
    // ---------------------------------------
    unsigned int hdrFBO;
    glGenFramebuffers(1, &hdrFBO);
    glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
    // create 2 floating point color buffers (1 for normal rendering, other for brightness threshold values)
    // currently the hdrFBO is binded, so these two color attachments all bind to hdrFBO
    unsigned int colorBuffers[2];
    glGenTextures(2, colorBuffers);
    for (unsigned int i = 0; i < 2; i++)
    {
        glBindTexture(GL_TEXTURE_2D, colorBuffers[i]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);  // we clamp to the edge as the blur filter would otherwise sample repeated texture values!
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
        // attach texture to framebuffer
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0 + i, GL_TEXTURE_2D, colorBuffers[i], 0);
    }
    // create and attach depth buffer (renderbuffer)
    // Only one depth buffer
    unsigned int rboDepth;
    glGenRenderbuffers(1, &rboDepth);
    glBindRenderbuffer(GL_RENDERBUFFER, rboDepth);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT, SCR_WIDTH, SCR_HEIGHT);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, rboDepth);
    // tell OpenGL which color attachments we'll use (of this framebuffer) for rendering 
    // Otherwise, all results will be rendered on the first attachment
    unsigned int attachments[2] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1 };
    glDrawBuffers(2, attachments);

    ...

    // ping-pong-framebuffer for blurring
    unsigned int pingpongFBO[2];
    unsigned int pingpongColorbuffers[2];
    glGenFramebuffers(2, pingpongFBO);
    glGenTextures(2, pingpongColorbuffers);
    // Here, we prepare two FBO and their own color attachment for bluring process.
    for (unsigned int i = 0; i < 2; i++)
    {
        glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[i]);
        glBindTexture(GL_TEXTURE_2D, pingpongColorbuffers[i]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE); // we clamp to the edge as the blur filter would otherwise sample repeated texture values!
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, pingpongColorbuffers[i], 0);
        // also check if framebuffers are complete (no need for depth buffer)
        if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
            std::cout << "Framebuffer not complete!" << std::endl;
    }

    ...

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window))
    {
        ...

        // 1. render scene into floating point framebuffer
        // Here we bind two color attachment to the hdrFBO before, so the result of render cubes and render lights are stored separately
        // -----------------------------------------------
        glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
        ...
        shader.use(); // shader("7.bloom.vs", "7.bloom.fs");
        // Render cubes to hdrFBO
        ...
        // finally show all the light sources as bright cubes
        shaderLight.use(); // shaderLight("7.bloom.vs", "7.light_box.fs");
        ...
        // bind back
        glBindFramebuffer(GL_FRAMEBUFFER, 0);

        // 2. blur bright fragments with two-pass Gaussian Blur 
        // --------------------------------------------------
        bool horizontal = true, first_iteration = true;
        unsigned int amount = 10;
        shaderBlur.use(); // shaderBlur("7.blur.vs", "7.blur.fs");
        for (unsigned int i = 0; i < amount; i++) // run gaussian blur ten times
        {
            glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[horizontal]);
            shaderBlur.setInt("horizontal", horizontal);
            glBindTexture(GL_TEXTURE_2D, first_iteration ? colorBuffers[1] : pingpongColorbuffers[!horizontal]);  
            // bind texture of other framebuffer (or scene if first iteration), pingpongColorbuffers[0] and pingpongColorbuffers[1] switch, one horizontal blur and one vertical
            renderQuad();
            horizontal = !horizontal;
            if (first_iteration)
                first_iteration = false;
        }
        // bind back
        glBindFramebuffer(GL_FRAMEBUFFER, 0);

        // 3. now render floating point color buffer to 2D quad and tonemap HDR colors to default framebuffer's (clamped) color range
        // --------------------------------------------------------------------------------------------------------------------------
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        shaderBloomFinal.use(); // 
        glActiveTexture(GL_TEXTURE0);
        // colorBuffers[0] do not have light cube on it
        glBindTexture(GL_TEXTURE_2D, colorBuffers[0]);
        glActiveTexture(GL_TEXTURE1);
        // At last we do horizontal = !horizontal, to get final result of bluring pingpongColorbuffers[!horizontal]
        glBindTexture(GL_TEXTURE_2D, pingpongColorbuffers[!horizontal]);
        shaderBloomFinal.setInt("bloom", bloom);
        shaderBloomFinal.setFloat("exposure", exposure);
        renderQuad();
        ...
    }
    ...
}
```

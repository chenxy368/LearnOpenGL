# SSAO 
One type of indirect lighting approximation is called ambient occlusion that tries to approximate indirect lighting by darkening creases, holes, and surfaces 
that are close to each other. These areas are largely occluded by surrounding geometry and thus light rays have fewer places to escape to, hence the areas appear darker. 

![image](https://user-images.githubusercontent.com/98029669/214497045-9a1c930b-a326-471b-b1ca-f4d5eb03cc30.png)

It is clear the quality and precision of the effect directly relates to the number of surrounding samples we take. 
We can reduce the amount of samples we have to test by introducing some randomness into the sample kernel. 
By randomly rotating the sample kernel each fragment we can get high quality results with a much smaller amount of samples.
This does come at a price as the randomness introduces a noticeable noise pattern that we'll have to fix by blurring the results. 

![image](https://user-images.githubusercontent.com/98029669/214497440-bad7c633-ab30-4e23-acae-53c4a541b962.png)

Because the sample kernel used was a sphere, it caused flat walls to look gray as half of the kernel samples end up being in the surrounding geometry. 

![image](https://user-images.githubusercontent.com/98029669/214497673-8e75ec46-1644-465d-886d-62461b8c52ed.png)

__In fact, the essential idea of SSAO is the same as all convolution kernel method in image processing. Just sample around the pixel and change the pixel value with the sample result.
Here, we also need some trick to have a better sample quality.__

## G-buffer
For each fragment, we're going to need the following data:

1. A per-fragment position vector.  
2. A per-fragment normal vector.  
3. A per-fragment albedo color.  
4. A sample kernel.  
5. A per-fragment random rotation vector used to rotate the sample kernel.  

![image](https://user-images.githubusercontent.com/98029669/214498211-c3ef3d6f-02c0-4ce4-b4e5-bddd85dc4fae.png)

__For sampling, we only need depth buffer which is a 2D texture so no 3D information is needed here.__
## Generate Sample Semi-sphere
We're going to generate a sample kernel in tangent space, with the normal vector pointing in the positive z direction.
```C++
std::uniform_real_distribution<float> randomFloats(0.0, 1.0); // random floats between [0.0, 1.0]
std::default_random_engine generator;
std::vector<glm::vec3> ssaoKernel;
for (unsigned int i = 0; i < 64; ++i)
{
    // semi-sphere sampling
    glm::vec3 sample(
        randomFloats(generator) * 2.0 - 1.0, // [-1, 1]
        randomFloats(generator) * 2.0 - 1.0, // [-1, 1]
        randomFloats(generator) // [0, 1]
    );
    sample  = glm::normalize(sample);
    sample *= randomFloats(generator);
    ssaoKernel.push_back(sample);  
}
```
We want to sample more points that is closer to the kernel.
```C++
scale = lerp(0.1f, 1.0f, scale * scale);
sample *= scale;
ssaoKernel.push_back(sample);  

GLfloat lerp(GLfloat a, GLfloat b, GLfloat f)
{
    return a + f * (b - a);
}
```
![image](https://user-images.githubusercontent.com/98029669/214500028-7a21bd5f-f24e-487a-956e-9aecae803b9e.png)

## Random Kernel Rotations
We could create a random rotation vector for each fragment of a scene, but that quickly eats up memory. It makes more sense to create a small texture of random rotation vectors that we tile over the screen.(Because all our work is in 2D space)
```C++
std::vector<glm::vec3> ssaoNoise;
for (GLuint i = 0; i < 16; i++)
{
    // Rotate along z direction
    glm::vec3 noise(
        randomFloats(generator) * 2.0 - 1.0, 
        randomFloats(generator) * 2.0 - 1.0, 
        0.0f); 
    ssaoNoise.push_back(noise);
}

GLuint noiseTexture; 
glGenTextures(1, &noiseTexture);
glBindTexture(GL_TEXTURE_2D, noiseTexture);
// size of ssaoNoise is 16
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, 4, 4, 0, GL_RGB, GL_FLOAT, &ssaoNoise[0]);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
```
## C++ part structure
```C++

float ourLerp(float a, float b, float f)
{
    return a + f * (b - a);
}

int main()
{
    // build and compile shaders
    // -------------------------
    Shader shaderGeometryPass("9.ssao_geometry.vs", "9.ssao_geometry.fs");
    Shader shaderLightingPass("9.ssao.vs", "9.ssao_lighting.fs");
    Shader shaderSSAO("9.ssao.vs", "9.ssao.fs");
    Shader shaderSSAOBlur("9.ssao.vs", "9.ssao_blur.fs");

    // configure g-buffer framebuffer
    // ------------------------------
    ...
    // position color buffer
    ...
    // normal color buffer
    ...
    // color + specular color buffer
    ...
    // create and attach depth buffer (renderbuffer)
   
    // also create framebuffer to hold SSAO processing stage 
    // -----------------------------------------------------
    unsigned int ssaoFBO, ssaoBlurFBO; // one for ssao, one for blur afterward
    glGenFramebuffers(1, &ssaoFBO);  glGenFramebuffers(1, &ssaoBlurFBO);
    glBindFramebuffer(GL_FRAMEBUFFER, ssaoFBO);
    unsigned int ssaoColorBuffer, ssaoColorBufferBlur;
    // SSAO color buffer
    glGenTextures(1, &ssaoColorBuffer);
    // Attach a color
    glBindTexture(GL_TEXTURE_2D, ssaoColorBuffer);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RED, SCR_WIDTH, SCR_HEIGHT, 0, GL_RED, GL_FLOAT, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, ssaoColorBuffer, 0);
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
        std::cout << "SSAO Framebuffer not complete!" << std::endl;
    // and blur stage
    glBindFramebuffer(GL_FRAMEBUFFER, ssaoBlurFBO);
    glGenTextures(1, &ssaoColorBufferBlur);
    // Attach a color
    glBindTexture(GL_TEXTURE_2D, ssaoColorBufferBlur);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RED, SCR_WIDTH, SCR_HEIGHT, 0, GL_RED, GL_FLOAT, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, ssaoColorBufferBlur, 0);
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
        std::cout << "SSAO Blur Framebuffer not complete!" << std::endl;
    glBindFramebuffer(GL_FRAMEBUFFER, 0);

    // generate sample kernel
    // ----------------------
    std::uniform_real_distribution<GLfloat> randomFloats(0.0, 1.0); // generates random floats between 0.0 and 1.0
    std::default_random_engine generator;
    std::vector<glm::vec3> ssaoKernel;
    for (unsigned int i = 0; i < 64; ++i)
    {
        glm::vec3 sample(randomFloats(generator) * 2.0 - 1.0, randomFloats(generator) * 2.0 - 1.0, randomFloats(generator));
        sample = glm::normalize(sample);
        sample *= randomFloats(generator);
        float scale = float(i) / 64.0f;

        // scale samples s.t. they're more aligned to center of kernel
        scale = ourLerp(0.1f, 1.0f, scale * scale);
        sample *= scale;
        ssaoKernel.push_back(sample);
    }

    // generate noise texture
    // ----------------------
    std::vector<glm::vec3> ssaoNoise;
    for (unsigned int i = 0; i < 16; i++)
    {
        glm::vec3 noise(randomFloats(generator) * 2.0 - 1.0, randomFloats(generator) * 2.0 - 1.0, 0.0f); // rotate around z-axis (in tangent space)
        ssaoNoise.push_back(noise);
    }
    unsigned int noiseTexture; glGenTextures(1, &noiseTexture);
    glBindTexture(GL_TEXTURE_2D, noiseTexture);
    // An interpolation performed to have different rotation on different fragments in shader
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA32F, 4, 4, 0, GL_RGB, GL_FLOAT, &ssaoNoise[0]);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);

    ...

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window))
    {
        
        // 1. geometry pass: render scene's geometry/color data into gbuffer
        // -----------------------------------------------------------------
        glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
        ...
        glBindFramebuffer(GL_FRAMEBUFFER, 0);


        // 2. generate SSAO texture
        // ------------------------
        glBindFramebuffer(GL_FRAMEBUFFER, ssaoFBO);
            glClear(GL_COLOR_BUFFER_BIT);
            shaderSSAO.use();
            // Send kernel + rotation 
            for (unsigned int i = 0; i < 64; ++i)
                shaderSSAO.setVec3("samples[" + std::to_string(i) + "]", ssaoKernel[i]);
            shaderSSAO.setMat4("projection", projection);
            glActiveTexture(GL_TEXTURE0);
            glBindTexture(GL_TEXTURE_2D, gPosition);
            glActiveTexture(GL_TEXTURE1);
            glBindTexture(GL_TEXTURE_2D, gNormal);
            glActiveTexture(GL_TEXTURE2);
            glBindTexture(GL_TEXTURE_2D, noiseTexture);
            renderQuad();
        glBindFramebuffer(GL_FRAMEBUFFER, 0);


        // 3. blur SSAO texture to remove noise
        // ------------------------------------
        glBindFramebuffer(GL_FRAMEBUFFER, ssaoBlurFBO);
            glClear(GL_COLOR_BUFFER_BIT);
            shaderSSAOBlur.use();
            glActiveTexture(GL_TEXTURE0);
            glBindTexture(GL_TEXTURE_2D, ssaoColorBuffer);
            renderQuad();
        glBindFramebuffer(GL_FRAMEBUFFER, 0);


        // 4. lighting pass: traditional deferred Blinn-Phong lighting with added screen-space ambient occlusion
        // -----------------------------------------------------------------------------------------------------
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        shaderLightingPass.use();
        // send light relevant uniforms
        glm::vec3 lightPosView = glm::vec3(camera.GetViewMatrix() * glm::vec4(lightPos, 1.0));
        shaderLightingPass.setVec3("light.Position", lightPosView);
        shaderLightingPass.setVec3("light.Color", lightColor);
        // Update attenuation parameters
        const float linear    = 0.09f;
        const float quadratic = 0.032f;
        shaderLightingPass.setFloat("light.Linear", linear);
        shaderLightingPass.setFloat("light.Quadratic", quadratic);
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, gPosition);
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, gNormal);
        glActiveTexture(GL_TEXTURE2);
        glBindTexture(GL_TEXTURE_2D, gAlbedo);
        glActiveTexture(GL_TEXTURE3); // add extra SSAO texture to lighting pass
        glBindTexture(GL_TEXTURE_2D, ssaoColorBufferBlur);
        renderQuad();

      ...
    }
    ...
}
```

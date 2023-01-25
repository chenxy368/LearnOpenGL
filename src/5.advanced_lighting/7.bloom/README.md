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
        shaderBloomFinal.use(); // shaderBloomFinal("7.bloom_final.vs", "7.bloom_final.fs");
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
## Shaders
__bloow.vs__, this vertex shader is shared by bloow.fs and light_box.fs

nothing special
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

void main()
{
    vs_out.FragPos = vec3(model * vec4(aPos, 1.0));   
    vs_out.TexCoords = aTexCoords;
        
    mat3 normalMatrix = transpose(inverse(mat3(model)));
    vs_out.Normal = normalize(normalMatrix * aNormal);
    
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}
```
__bloom.fs__
```GLSL
#version 330 core
// Multiple Render Targets (MRT) 
// Specifying a layout location specifier before a fragment shader's output we can control to which color buffer a fragment shader writes to
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;
// FragColor to attachment 0 and BrightColor to attachment 1

[...]

void main()
{            
    [...] // first do normal lighting calculations and output results
    FragColor = vec4(lighting, 1.0f);
    // Check whether fragment output is higher than threshold, if so output as brightness color
    float brightness = dot(FragColor.rgb, vec3(0.2126, 0.7152, 0.0722));
    // Since we use HDR, the brightness is allowed to be larger than 1.0. If the brightness larger than 1.0, it need bloom(to attachment 1)
    if(brightness > 1.0)
        BrightColor = vec4(result, 1.0);
    else
        BrightColor = vec4(0.0, 0.0, 0.0, 1.0);
    FragColor = vec4(result, 1.0);
}
```
__light_box.fs__(similar to bloom.fs)
```GLSL
#version 330 core
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;

in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
} fs_in;

uniform vec3 lightColor;

void main()
{           
    FragColor = vec4(lightColor, 1.0);
    float brightness = dot(FragColor.rgb, vec3(0.2126, 0.7152, 0.0722));
    if(brightness > 1.0)
        BrightColor = vec4(FragColor.rgb, 1.0);
	else
		BrightColor = vec4(0.0, 0.0, 0.0, 1.0);
}
```
![image](https://user-images.githubusercontent.com/98029669/214477479-5590b569-9c1c-4d8a-b254-6f4f774ceace.png)

Decompose 2D gaussian blur to 1D. Run vertical and horizontal blur separately. One vertical and one horizontal. The result can be the same as 2D blur. The efficiency can be higher.  
![image](https://user-images.githubusercontent.com/98029669/214478377-9c0ef7e6-8e85-4448-8e9b-89acdaea8c06.png)
![image](https://user-images.githubusercontent.com/98029669/214479164-11afb71b-761a-4732-92f2-3ea13808dcca.png)

__blur.vs__(nothing special)
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoords;

out vec2 TexCoords;

void main()
{
    TexCoords = aTexCoords;
    gl_Position = vec4(aPos, 1.0);
}
```
__blur.fs__
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D image;

uniform bool horizontal;
uniform float weight[5] = float[] (0.2270270270, 0.1945945946, 0.1216216216, 0.0540540541, 0.0162162162);

void main()
{             
     vec2 tex_offset = 1.0 / textureSize(image, 0); // gets size of single texel
     vec3 result = texture(image, TexCoords).rgb * weight[0];
     if(horizontal)
     {
         for(int i = 1; i < 5; ++i)
         {
            result += texture(image, TexCoords + vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
         }
     }
     else
     {
         for(int i = 1; i < 5; ++i)
         {
             result += texture(image, TexCoords + vec2(0.0, tex_offset.y * i)).rgb * weight[i];
             result += texture(image, TexCoords - vec2(0.0, tex_offset.y * i)).rgb * weight[i];
         }
     }
     FragColor = vec4(result, 1.0);
}
```
Mix together  
__bloom_final.vs__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoords;

out vec2 TexCoords;

void main()
{
    TexCoords = aTexCoords;
    gl_Position = vec4(aPos, 1.0);
}
```
__bloom_final.fs__
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D scene;
uniform sampler2D bloomBlur;
uniform bool bloom;
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(scene, TexCoords).rgb;      
    vec3 bloomColor = texture(bloomBlur, TexCoords).rgb;
    if(bloom)
        hdrColor += bloomColor; // additive blending
    // tone mapping
    vec3 result = vec3(1.0) - exp(-hdrColor * exposure);
    // also gamma correct while we're at it       
    result = pow(result, vec3(1.0 / gamma));
    FragColor = vec4(result, 1.0);
}
```

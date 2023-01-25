# HDR
Brightness and color values, by default, are clamped between 0.0 and 1.0 when stored into a framebuffer. 

What happens if we walk in a really bright area with multiple bright light sources that as a total sum exceed 1.0? 
The answer is that all fragments that have a brightness or color sum over 1.0 get clamped to 1.0, which isn't pretty to look at
(losing a significant amount of detail and giving it a fake look):  
![image](https://user-images.githubusercontent.com/98029669/214453670-912cb01b-5ed3-48a2-97eb-5a01ee605548.png)

Monitors (non-HDR) are limited to display colors in the range of 0.0 and 1.0, but there is no such limitation in lighting equations. 
__By allowing fragment colors to exceed 1.0__ we have a much higher range of color values available to work in known as high dynamic range (HDR). 

High dynamic range rendering allow a much larger range of color values to render to, and at the end we transform all the HDR values __back to the low dynamic range (LDR) of [0.0, 1.0]__. 
This process of converting HDR values to LDR values is called __tone mapping__.
These tone mapping algorithms often involve an __exposure parameter__ that selectively favors dark or bright regions.  
Different exposure level:  
![image](https://user-images.githubusercontent.com/98029669/214454090-2cebf223-4928-44a4-9547-37176e508e12.png)

HDR gives us the ability to specify a light source's intensity by their real intensities. 
For instance, the sun has a much higher intensity than something like a flashlight so why not configure the sun as such __(e.g. a diffuse brightness of 100.0)__. 
This allows us to more properly configure a scene's lighting with more realistic lighting parameters.
 
## Floating Point Framebuffers
When framebuffers use a normalized fixed-point color format (like GL_RGB) as their color buffer's internal format, OpenGL automatically clamps the values between 0.0 and 1.0 before storing them in the framebuffer. 
 
To get rid of the limitation, use GL_RGB16F, GL_RGBA16F, GL_RGB32F, or GL_RGBA32F. The framebuffer is known as a floating point framebuffer.
```C++
glBindTexture(GL_TEXTURE_2D, colorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);  
```
First render a lit scene into the floating point framebuffer and then display the framebuffer's color buffer on a screen-filled quad:
```C++
// 1. render scene into floating point framebuffer
// -----------------------------------------------
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glm::mat4 projection = glm::perspective(glm::radians(camera.Zoom), (GLfloat)SCR_WIDTH / (GLfloat)SCR_HEIGHT, 0.1f, 100.0f);
    glm::mat4 view = camera.GetViewMatrix();
    shader.use();
    shader.setMat4("projection", projection);
    shader.setMat4("view", view);
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, woodTexture);
    // set lighting uniforms
    for (unsigned int i = 0; i < lightPositions.size(); i++)
    {
         shader.setVec3("lights[" + std::to_string(i) + "].Position", lightPositions[i]);
         shader.setVec3("lights[" + std::to_string(i) + "].Color", lightColors[i]);
    }
    shader.setVec3("viewPos", camera.Position);
    // render tunnel
    glm::mat4 model = glm::mat4(1.0f);
    model = glm::translate(model, glm::vec3(0.0f, 0.0f, 25.0));
    model = glm::scale(model, glm::vec3(2.5f, 2.5f, 27.5f));
    shader.setMat4("model", model);
    shader.setInt("inverse_normals", true);
    renderCube();
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// 2. now render floating point color buffer to 2D quad and tonemap HDR colors to default framebuffer's (clamped) color range
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
hdrShader.use();
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, colorBuffer);
hdrShader.setInt("hdr", hdr);
hdrShader.setFloat("exposure", exposure);
renderQuad();
```
Set a hight light color source
```C++
std::vector<glm::vec3> lightColors;
lightColors.push_back(glm::vec3(200.0f, 200.0f, 200.0f));
lightColors.push_back(glm::vec3(0.1f, 0.0f, 0.0f));
lightColors.push_back(glm::vec3(0.0f, 0.0f, 0.2f));
lightColors.push_back(glm::vec3(0.0f, 0.1f, 0.0f));  
```
To render the hdr quad, set a Pass-through Fragment Shader
```GLSL
#version 330 core
out vec4 color;
in vec2 TexCoords;

uniform sampler2D hdrBuffer;

void main()
{             
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
    color = vec4(hdrColor, 1.0);
}  
```
## Tone mapping
__Reinhard tone mapping__: (r, g, b) / ((r, g, b) + (1, 1, 1))
```GLSL
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
    if(hdr) // press space to change using reinhard or not
    {
        // reinhard
        vec3 result = hdrColor / (hdrColor + vec3(1.0));
        // also gamma correct while we're at it       
        result = pow(result, vec3(1.0 / gamma));
        FragColor = vec4(result, 1.0);
    }
    else
    {
        vec3 result = pow(hdrColor, vec3(1.0 / gamma));
        FragColor = vec4(result, 1.0);
    }
}
```
__Exposure tone mapping__: (1, 1, 1) - e^(-(r, g, b)*exposure) 
```GLSL
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
    if(hdr) // press space to change using reinhard or not
    {
        // exposure
        // tune the exposure parameter with Q and E
        vec3 result = vec3(1.0) - exp(-hdrColor * exposure);
        // also gamma correct while we're at it       
        result = pow(result, vec3(1.0 / gamma));
        FragColor = vec4(result, 1.0);
    }
    else
    {
        vec3 result = pow(hdrColor, vec3(1.0 / gamma));
        FragColor = vec4(result, 1.0);
    }
}
```

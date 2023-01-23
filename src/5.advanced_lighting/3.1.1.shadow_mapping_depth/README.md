# Shadow Mapping
![image](https://user-images.githubusercontent.com/98029669/214085676-f354880a-0201-47ad-ae89-66105edee65d.png)
## Shadow Mapping
We render the scene from the light's point of view and everything we see from the light's perspective is lit and everything we can't see must be in shadow.
(If camera at the light's position, what cannot be seen should be in shadow.)  
![image](https://user-images.githubusercontent.com/98029669/214086029-28d7b15c-19a2-471f-8258-a8c79a20eda1.png)  
Here blue line are lit and black line are in shadow.

Naively, we want to get the point on the ray where it first hit an object and compare this closest point to other points on this ray. 
We then do a basic test to see if a test point's ray position is further down the ray than the closest point and if so, the test point must be in shadow. __(Too computational expensive)__

__!!!If camera at the light's position, what cannot be seen should be in shadow.!!!__ Why not use the depth buffer when we put the camera at the light's position to construct shadow.

You may remember from the depth testing chapter that a value in the depth buffer corresponds to the depth of a fragment clamped to [0,1] from the camera's point of view. 
After all, the depth values show the first fragment visible from the light's perspective. 
We store __all these depth values in a texture__ that we call a __depth map or shadow map__.

![image](https://user-images.githubusercontent.com/98029669/214135086-659daaab-f311-4352-bc31-161cccc83d83.png)

In brief, Shadow mapping consists of two passes: first we render the depth map, and in the second pass we render the scene as normal and use the generated depth map to calculate whether fragments are in shadow.

## Step One: Generate Depth Map
1. Shadow on a plane
```C++
float planeVertices[] = {
    // positions            // normals         // texcoords
    25.0f, -0.5f,  25.0f,  0.0f, 1.0f, 0.0f,  25.0f,  0.0f,
   -25.0f, -0.5f,  25.0f,  0.0f, 1.0f, 0.0f,   0.0f,  0.0f,
   -25.0f, -0.5f, -25.0f,  0.0f, 1.0f, 0.0f,   0.0f, 25.0f,

    25.0f, -0.5f,  25.0f,  0.0f, 1.0f, 0.0f,  25.0f,  0.0f,
   -25.0f, -0.5f, -25.0f,  0.0f, 1.0f, 0.0f,   0.0f, 25.0f,
    25.0f, -0.5f, -25.0f,  0.0f, 1.0f, 0.0f,  25.0f, 25.0f
};
```
2. First we'll create a framebuffer object for rendering the depth map:  
```C++
GLuint depthMapFBO;
glGenFramebuffers(1, &depthMapFBO);
```
3. Next we create a 2D texture that we'll use as the framebuffer's depth buffer:
```C++
const unsigned int SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;
unsigned int depthMap;
glGenTextures(1, &depthMap);
glBindTexture(GL_TEXTURE_2D, depthMap);
// Specify the texture's formats as GL_DEPTH_COMPONENT
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT); 
```
4. With the generated depth texture we can attach it as the framebuffer's depth buffer:
P.S. We __only need the depth information__ when rendering the scene from the light's perspective so there is __no need for a color buffer__. 
However, a framebuffer object is __not complete__ without a color buffer so we need to __explicitly tell OpenGL__ we're not going to render any color data. 
We do this by __setting both the read and draw buffer to GL_NONE with glDrawBuffer and glReadbuffer__.
```C++
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);
glDrawBuffer(GL_NONE); // No color buffer
glReadBuffer(GL_NONE); // No color buffer
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
5. In render loop, generate perspective and view matrix based on light position and direction:
__ortho projection here, littel difference between ortho and perspective projection here, discuss later__
```C++
// 1. render depth of scene to texture (from light's perspective)
glm::mat4 lightProjection, lightView;
glm::mat4 lightSpaceMatrix;
float near_plane = 1.0f, far_plane = 7.5f;
lightProjection = glm::ortho(-10.0f, 10.0f, -10.0f, 10.0f, near_plane, far_plane); // Ortho projection
lightView = glm::lookAt(lightPos, glm::vec3(0.0f), glm::vec3(0.0, 1.0, 0.0));
lightSpaceMatrix = lightProjection * lightView;
// render scene from light's point of view
simpleDepthShader.use();
simpleDepthShader.setMat4("lightSpaceMatrix", lightSpaceMatrix);
```
6. Vertex shader for shadow map genenration:
__See what is lightSpaceMatrix above.__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 lightSpaceMatrix;
uniform mat4 model;

void main()
{
    gl_Position = lightSpaceMatrix * model * vec4(aPos, 1.0);
}
```
7. Fragment shader for shadow map genenration:
__In fact, nothing is needed.__
```GLSL
#version 330 core

void main()
{             
    // gl_FragDepth = gl_FragCoord.z;
}
```
8. Get Depth map
```C++
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glClear(GL_DEPTH_BUFFER_BIT);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, woodTexture);
renderScene(simpleDepthShader);
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```
### Let's check the depth map
Vertex Shader
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
Fragment Shader(Difference are between a perspective and orthogonal projection, dicuss later)
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D depthMap;
uniform float near_plane;
uniform float far_plane;

// required when using a perspective projection matrix
float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0; // Back to NDC 
    return (2.0 * near_plane * far_plane) / (far_plane + near_plane - z * (far_plane - near_plane));	
}

void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    // FragColor = vec4(vec3(LinearizeDepth(depthValue) / far_plane), 1.0); // perspective
    FragColor = vec4(vec3(depthValue), 1.0); // orthographic
}
```
Render(A texture image)
```C++
    // render Depth map to quad for visual debugging

    debugDepthQuad.use();
    debugDepthQuad.setFloat("near_plane", near_plane);
    debugDepthQuad.setFloat("far_plane", far_plane);
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, depthMap);
    renderQuad();
    ...
...
// renderQuad() renders a 1x1 XY quad in NDC
unsigned int quadVAO = 0;
unsigned int quadVBO;
void renderQuad()
{
    if (quadVAO == 0)
    {
        float quadVertices[] = {
            // positions        // texture Coords
            -1.0f,  1.0f, 0.0f, 0.0f, 1.0f,
            -1.0f, -1.0f, 0.0f, 0.0f, 0.0f,
             1.0f,  1.0f, 0.0f, 1.0f, 1.0f,
             1.0f, -1.0f, 0.0f, 1.0f, 0.0f,
        };
        // setup plane VAO
        glGenVertexArrays(1, &quadVAO);
        glGenBuffers(1, &quadVBO);
        glBindVertexArray(quadVAO);
        glBindBuffer(GL_ARRAY_BUFFER, quadVBO);
        glBufferData(GL_ARRAY_BUFFER, sizeof(quadVertices), &quadVertices, GL_STATIC_DRAW);
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)0);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)(3 * sizeof(float)));
    }
    glBindVertexArray(quadVAO);
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    glBindVertexArray(0);
}
```

### Difference between ortho and perspective projection in generating shadow map
![image](https://user-images.githubusercontent.com/98029669/214149184-f5426c0f-1428-464e-85d4-f23d7af3cda1.png)

An __orthographic projection__ matrix does not deform the scene with perspective so all view/light rays are parallel. This makes it a great projection matrix for __directional lights__. 

__Perspective projections__ make most sense for light sources that have actual locations, unlike directional lights. Perspective projections are most often used __with spotlights and point lights__.

Another subtle difference with using a perspective projection matrix is that visualizing the depth buffer will often give an almost completely white result. This happens because with perspective projection the depth is transformed to non-linear depth values with most of its noticeable range close to the near plane. To be able to properly view the depth values as we did with the orthographic projection you first want to transform the non-linear depth values to linear.(Do a linearize when you want to check depth map)
```GLSL
float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0; // Back to NDC 
    return (2.0 * near_plane * far_plane) / (far_plane + near_plane - z * (far_plane - near_plane));
}

void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    FragColor = vec4(vec3(LinearizeDepth(depthValue) / far_plane), 1.0); // perspective
}  
```

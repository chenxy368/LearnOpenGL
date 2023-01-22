# Framebuffers
The combination of color buffer, depth buffer and stencil buffer is stored somewhere in GPU memory and is called a framebuffer. 
OpenGL gives us the flexibility to define our own framebuffers and thus define our own color (and optionally a depth and stencil) buffer.
The rendering operations we've done so far were all done on top of the render buffers attached to the default framebuffer.
The default framebuffer is created and configured when you create your window (GLFW does this for us). 

__!!!It's important to know what we have done for rendering is just draw a picture. It's quite like a texture. 
Framebuffer defines a memory space in GPU to store this picture or texture. 
Great benefit is we can process this image after rendering, which is much cheaper than working in 3D.__
## Creat a Framebuffer
1. Initializa a framebuffer
```C++
unsigned int fbo;
glGenFramebuffers(1, &fbo);
```
2. Bind a framebuffer(same as VAO, VBO and EBO)
```C++
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
```
P.S. It is also possible to bind a framebuffer to a read or write target specifically by binding to GL_READ_FRAMEBUFFER or GL_DRAW_FRAMEBUFFER respectively. 
The framebuffer bound to GL_READ_FRAMEBUFFER is then used for all read operations like glReadPixels and the framebuffer bound to GL_DRAW_FRAMEBUFFER is used as the
destination for rendering, clearing and other write operations.

3. For a framebuffer __to be complete__ the following requirements have to be satisfied:
1. We have to attach at least one buffer (color, depth or stencil buffer).
2. There should be at least one color attachment.
3. All attachments should be complete as well (reserved memory).
4. Each buffer should have the same number of samples.

Check whether complete
```C++
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE)
  // execute victory dance
```

4. All subsequent rendering operations will now render to the attachments of the currently bound framebuffer. 
Since our framebuffer is not the default framebuffer, the rendering commands will have no impact on the visual output of your window. 
For this reason it is called __off-screen rendering__ when rendering to a different framebuffer. 
If you want all rendering operations to have a visual impact again on the main window we need to make the default framebuffer active by binding to 0:
```C++
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

5. Delete to release
```C++
glDeleteFramebuffers(1, &fbo);
```
## Texture Attachments
__When attaching a texture to a framebuffer, all rendering commands will write to the texture as if it was a normal color/depth or stencil buffer. I.e. Render output to a texture image.__  
1. Creating a texture for a framebuffer is roughly the same as creating a normal texture:
```C++
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
  
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  
```
The main differences here is that we set the dimensions equal to the screen size (although this is not required) and we pass NULL as the texture's data parameter.
For this texture, we're __only allocating memory and not actually filling it.__ Filling the texture will happen as soon as we render to the framebuffer. 
Also note that we do not care about any of the wrapping methods or mipmapping since we won't be needing those in most cases.  
2. Attach it to the framebuffer:
```C++
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
```
The __glFrameBufferTexture2D__ function has the following parameters:

target: the framebuffer type we're targeting (draw, read or both).  
attachment: the type of attachment we're going to attach. Right now we're attaching a color attachment. 
Note that the 0 at the end suggests we can attach more than 1 color attachment.  
textarget: the type of the texture you want to attach.  
texture: the actual texture to attach.  
level: the mipmap level. We keep this at 0.  
3. To attach a depth attachment we specify the attachment type as GL_DEPTH_ATTACHMENT.
Note that the texture's format and internalformat type should then become GL_DEPTH_COMPONENT to reflect the depth buffer's storage format. 
To attach a stencil buffer you use GL_STENCIL_ATTACHMENT as the second argument and specify the texture's formats as GL_STENCIL_INDEX.  
To attach both a depth buffer and a stencil buffer as a single texture. 
Each 32 bit value of the texture then contains 24 bits of depth information and 8 bits of stencil information. 
To attach a depth and stencil buffer as one texture we use the GL_DEPTH_STENCIL_ATTACHMENT type.
```C++
glTexImage2D(
  GL_TEXTURE_2D, 0, GL_DEPTH24_STENCIL8, 800, 600, 0, 
  GL_DEPTH_STENCIL, GL_UNSIGNED_INT_24_8, NULL
);

glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_TEXTURE_2D, texture, 0);
```
## Renderbuffer Object Attachments
Quite the same as texture object attachments, but OpenGL can do a few memory optimizations that can give it a performance edge over textures for off-screen rendering to a framebuffer.
__(faster)__

You cannot read from them directly, but it is possible to read from them via the slow glReadPixels. 
This returns a specified area of pixels from the currently bound framebuffer, but not directly from the attachment itself.
 
You cannot read from them directly, but it is possible to read from them via the slow glReadPixels. 
This returns a specified area of pixels from the currently bound framebuffer, but not directly from the attachment itself.

1. Creating a renderbuffer object
```C++
unsigned int rbo;
glGenRenderbuffers(1, &rbo);
```
2. Bind the renderbuffer object 
```C++
bind the renderbuffer object 
```
3. Creating a depth and stencil renderbuffer object
P.S. Since renderbuffer objects are write-only they are often used as depth and stencil attachments, 
since most of the time we don't really need to read values from them, but we do care about depth and stencil testing.
We need the depth and stencil values for testing, but don't need to sample these values so a renderbuffer object suits this perfectly. 
When we're not sampling from these buffers, a renderbuffer object is generally preferred.
```C++
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);
```
4. Attach the renderbuffer object
```C++
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo); 
```
## Rendering to a Texture
We're going to render the scene into a color texture attached to a framebuffer object we created and then draw this texture over a simple quad that spans the whole screen. 
The visual output is then exactly the same as without a framebuffer, but this time it's all printed on top of a single quad.
__I.e. Render a picture and save. Then, draw the picture.__
1. Initialize
```C++
unsigned int framebuffer;
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);    
```
2. Create a texture image that we attach as a color attachment to the framebuffer(only color in this case)
```C++
// generate texture
unsigned int textureColorbuffer;
glGenTextures(1, &textureColorbuffer);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR );
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glBindTexture(GL_TEXTURE_2D, 0);

// attach it to currently bound framebuffer object
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, textureColorbuffer, 0); 
```
3. Creating a renderbuffer object(for depth and stencil)
```C++
unsigned int rbo;
glGenRenderbuffers(1, &rbo);
glBindRenderbuffer(GL_RENDERBUFFER, rbo); 
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);  
glBindRenderbuffer(GL_RENDERBUFFER, 0);
```
4. Attach the renderbuffer object to the depth and stencil attachment of the framebuffer
```C++
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
```
5. Check complete and bind framebuffer to default
```C++
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
	std::cout << "ERROR::FRAMEBUFFER:: Framebuffer is not complete!" << std::endl;
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```    
So, to draw the scene to a single texture we'll have to take the following steps:

1. Render the scene as usual with the new framebuffer bound as the active framebuffer.  
2. Bind to the default framebuffer.  
3. Draw a quad that spans the entire screen with the new framebuffer's color buffer as its texture.  

__Draw the texture:__
1. A simple vertex shader
```GLSL
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec2 aTexCoords;

out vec2 TexCoords;

void main()
{
    gl_Position = vec4(aPos.x, aPos.y, 0.0, 1.0); // In screen space
    TexCoords = aTexCoords;
}  
```
2. A simple fragment shader to sample texture
```GLSL
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D screenTexture;

void main()
{ 
    FragColor = texture(screenTexture, TexCoords);
}
```
3. Create and configure a VAO for the screen quad in render loop
```C++
// first pass
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // we're not using the stencil buffer now
glEnable(GL_DEPTH_TEST);
DrawScene();	
```
```C++
// second pass
glBindFramebuffer(GL_FRAMEBUFFER, 0); // back to default
glClearColor(1.0f, 1.0f, 1.0f, 1.0f); 
glClear(GL_COLOR_BUFFER_BIT);
  
screenShader.use();  
glBindVertexArray(quadVAO);
glDisable(GL_DEPTH_TEST);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glDrawArrays(GL_TRIANGLES, 0, 6); 
```

Note: 
1. Since each framebuffer we're using has its own set of buffers, we want to clear each of those buffers with the appropriate bits set by calling glClear. 
2. When drawing the quad, we're disabling depth testing since we want to make sure the quad always renders in front of everything else; we'll have to enable depth testing again when we draw the normal scene though.

## Post-processing
__Some computer vision process, now we are working on a image.__  
__Inversion:__  
```GLSL
void main()
{
    FragColor = vec4(vec3(1.0 - texture(screenTexture, TexCoords)), 1.0);
}  
```
![image](https://user-images.githubusercontent.com/98029669/213900150-6175a244-1756-41ad-ab24-4e668f6b3c55.png)  
__Grayscale:__  
```GLSL
void main()
{
    FragColor = texture(screenTexture, TexCoords);
    float average = 0.2126 * FragColor.r + 0.7152 * FragColor.g + 0.0722 * FragColor.b;
    FragColor = vec4(average, average, average, 1.0);
} 
```
![image](https://user-images.githubusercontent.com/98029669/213900169-0432a67f-4978-4c82-978b-07a216f0e9ba.png)  
__Kernel effects:__  
Convolution matrix/kernel
```GLSL
const float offset = 1.0 / 300.0;  

void main()
{
    vec2 offsets[9] = vec2[](
        vec2(-offset,  offset), // top-left
        vec2( 0.0f,    offset), // top-center
        vec2( offset,  offset), // top-right
        vec2(-offset,  0.0f),   // center-left
        vec2( 0.0f,    0.0f),   // center-center
        vec2( offset,  0.0f),   // center-right
        vec2(-offset, -offset), // bottom-left
        vec2( 0.0f,   -offset), // bottom-center
        vec2( offset, -offset)  // bottom-right    
    );

    float kernel[9] = float[](
        -1, -1, -1,
        -1,  9, -1,
        -1, -1, -1
    );
    
    vec3 sampleTex[9];
    for(int i = 0; i < 9; i++)
    {
        sampleTex[i] = vec3(texture(screenTexture, TexCoords.st + offsets[i]));
    }
    vec3 col = vec3(0.0);
    for(int i = 0; i < 9; i++)
        col += sampleTex[i] * kernel[i];
    
    FragColor = vec4(col, 1.0);
}  
```
![image](https://user-images.githubusercontent.com/98029669/213900203-f3b9bf38-16be-410f-8bda-d519dfd98e23.png)  
__Blur:__  
A blur kernel:
```GLSL
float kernel[9] = float[](
    1.0 / 16, 2.0 / 16, 1.0 / 16,
    2.0 / 16, 4.0 / 16, 2.0 / 16,
    1.0 / 16, 2.0 / 16, 1.0 / 16  
);
```
![image](https://user-images.githubusercontent.com/98029669/213900223-06aa077d-bf29-426e-a7b3-8ac8b6f965a8.png)  
__Edge detection:__  
An edge detection kernel:
```GLSL
float kernel[9] = float[](
    1.0, 1.0, 1.0,
    1.0, -8.0, 1.0,
    1.0, 1.0, 1.0,
);
```
![image](https://user-images.githubusercontent.com/98029669/213900276-c3b87ebd-567d-4c88-afb5-025ec9ca6356.png)  



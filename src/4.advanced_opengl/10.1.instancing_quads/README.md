# Instancing
Think of drawing a lot of models where most of these models contain the same set of vertex data, but with different world transformations. 
```C++
for(unsigned int i = 0; i < amount_of_models_to_draw; i++)
{
    DoSomePreparations(); // bind VAO, bind textures, set uniforms etc.
    glDrawArrays(GL_TRIANGLES, 0, amount_of_vertices);
}
```
You'll quickly reach a performance bottleneck because of the many draw calls. 

It would be much more convenient if we could send data over to the GPU once, and then tell OpenGL to draw multiple objects using this data with a single drawing call. 
Enter instancing. (Just do not need to do a lot of bind and data exchange between GPU and CPU)

To render using instancing all we need to do is change the render calls glDrawArrays and glDrawElements to __glDrawArraysInstanced and glDrawElementsInstanced__ respectively.
These instanced versions of the classic rendering functions __take an extra parameter called the instance count__ that sets the number of instances we want to render. 

However, rendering the same object a thousand times is of no use to us since __each of the rendered objects is rendered exactly the same__
and thus also at the same location. For this reason GLSL added another built-in variable in the vertex shader called __gl_InstanceID__.

__gl_InstanceID__ is incremented for each instance being rendered starting from 0. 
Having a unique value per instance means we could now for example index into __large array of position values to position each instance__ 
at a different location in the world.

## A Simple Example
![image](https://user-images.githubusercontent.com/98029669/213950698-17ffe813-7909-4d25-99b5-3a2fb6a239f5.png)
```C++
float quadVertices[] = {
    // positions     // colors
    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,
    -0.05f, -0.05f,  0.0f, 0.0f, 1.0f,

    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,   
     0.05f,  0.05f,  0.0f, 1.0f, 1.0f		    		
};  
```
__Fragment Shader Not Special__
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 fColor;

void main()
{
    FragColor = vec4(fColor, 1.0);
}
```
__Vertex Shader__
```GLSL
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;

out vec3 fColor;

// Contain a total of 100 offset vectors.
uniform vec2 offsets[100];

void main()
{
    // Within the vertex shader we retrieve an offset vector for each instance by indexing the offsets array using gl_InstanceID. 
    vec2 offset = offsets[gl_InstanceID];
    gl_Position = vec4(aPos + offset, 0.0, 1.0);
    fColor = aColor;
}
```
Initialize the offset array
```C++
glm::vec2 translations[100];
int index = 0;
float offset = 0.1f;
for(int y = -10; y < 10; y += 2)
{
    for(int x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (float)x / 10.0f + offset;
        translation.y = (float)y / 10.0f + offset;
        translations[index++] = translation;
    }
}  
```
Pass to shader
```C++
shader.use();
for(unsigned int i = 0; i < 100; i++)
{
    shader.setVec2(("offsets[" + std::to_string(i) + "]")), translations[i]);
} 
```
Draw instances
```C++
glBindVertexArray(quadVAO);
glDrawArraysInstanced(GL_TRIANGLES, 0, 6, 100); 
```

## Instanced Arrays
When there are a lot more than 100 instances, we will eventually hit a limit on the amount of uniform data we can send to the shaders. 
One alternative option is known as instanced arrays. 
Instanced arrays are defined as a __vertex attribute (allowing us to store much more data) that are updated per instance instead of per vertex__.
This allows us to use the standard vertex attributes for data per vertex and use the instanced array for storing data that is unique per instance.

__Vertex Shader__
```GLSL
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aOffset; // Instance array as a vertex attribute

out vec3 fColor;

void main()
{
    // no longer use gl_InstanceID and can directly use the offset attribute without first indexing into a large uniform array.
    gl_Position = vec4(aPos + aOffset, 0.0, 1.0); 
    fColor = aColor;
}  
```
Store the instance array's content in a vertex buffer object and configure its attribute pointer
```C++
glm::vec2 translations[100];
...
unsigned int instanceVBO;
glGenBuffers(1, &instanceVBO);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, &translations[0], GL_STATIC_DRAW);
glBindBuffer(GL_ARRAY_BUFFER, 0); 
...
glEnableVertexAttribArray(2);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)0);
glBindBuffer(GL_ARRAY_BUFFER, 0);	
glVertexAttribDivisor(2, 1);  
```
__glVertexAttribDivisor__ tells OpenGL when to update the content of a vertex attribute to the next element. 
Its first parameter is __the vertex attribute(pointer ID)__ in question and the second parameter the attribute divisor. 
By default, the attribute divisor is _0_ which tells OpenGL __to update the content of the vertex attribute each iteration of the vertex shader__. 
By setting this attribute to _1_ we're telling OpenGL that we want to __update the content of the vertex attribute when we start to render a new instance__.
By setting it to _2_ we'd update the content __every 2 instances__ and so on. 
By setting the attribute divisor to 1 we're effectively telling OpenGL that the vertex attribute at attribute __location 2 is an instanced array__.

![image](https://user-images.githubusercontent.com/98029669/213951926-842088a1-9163-4a99-9087-068987373f8e.png)
```GLSL
void main()
{
    vec2 pos = aPos * (gl_InstanceID / 100.0); // Zoom the instance
    gl_Position = vec4(pos + aOffset, 0.0, 1.0);
    fColor = aColor;
} 
```

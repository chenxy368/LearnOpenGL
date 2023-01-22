# Advanced Data
A buffer in OpenGL is, at its core, an object that manages a certain piece of GPU memory and nothing more.
We give meaning to a buffer when binding it to a specific buffer target. 
A buffer is only a vertex array buffer when we bind it to GL_ARRAY_BUFFER, but we could just as easily bind it to GL_ELEMENT_ARRAY_BUFFER.

So far we've been filling the buffer's memory by calling glBufferData, which allocates a piece of GPU memory and adds data into this memory. 
If we were to pass NULL as its data argument, the function would only allocate memory and not fill it. 
This is useful if we first want to reserve a specific amount of memory and later come back to this buffer.

## glBufferSubData
Fill specific regions of the buffer.
Do note that the buffer should have enough allocated memory so a call to glBufferData is necessary before calling.  
```C++
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(data), &data); // range: [24, 24 + sizeof(data)]
```

## glMapBuffer 
Directly use memcpy etc. to manage the gpu buffer by get a pointer.
```C++
float data[] = {
  0.5f, 1.0f, -0.35f
  [...]
};
glBindBuffer(GL_ARRAY_BUFFER, buffer);
// get pointer
void *ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// now copy data into memory
memcpy(ptr, data, sizeof(data));
// make sure to tell OpenGL we're done with the pointer
glUnmapBuffer(GL_ARRAY_BUFFER);
```

## Batching Vertex Attributes
Instead of an interleaved layout 123123123123 we take a batched approach 111122223333.
```C++
float positions[] = { ... };
float normals[] = { ... };
float tex[] = { ... };
// fill buffer
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(positions), &positions);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions), sizeof(normals), &normals);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals), sizeof(tex), &tex);
...

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), 0);  
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)(sizeof(positions)));  
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)(sizeof(positions) + sizeof(normals)));  
```

## Copying Buffers
```C++
void glCopyBufferSubData(GLenum readtarget, GLenum writetarget, GLintptr readoffset,
                         GLintptr writeoffset, GLsizeiptr size);
```
We could for example copy from a VERTEX_ARRAY_BUFFER buffer to a VERTEX_ELEMENT_ARRAY_BUFFER buffer by specifying those 
buffer targets as the read and write targets respectively.
If we wanted to read and write data into two different buffers that are both vertex array buffers, 
OpenGL gives us two more buffer targets called GL_COPY_READ_BUFFER and GL_COPY_WRITE_BUFFER.
```C++
float vertexData[] = { ... };
glBindBuffer(GL_COPY_READ_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```
We could've also done this by only binding the writetarget buffer to one of the new buffer target types:
```C++
float vertexData[] = { ... };
glBindBuffer(GL_ARRAY_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_ARRAY_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```
# Advanced GLSL
## GLSL's Built-in Variables
There are a few extra variables defined by GLSL prefixed with gl_ that give us an extra means to gather and/or write data. We've already seen two of them in the chapters so far: gl_Position that is the output vector of the vertex shader, and the fragment shader's gl_FragCoord.
### Vertex shader variables
__gl_PointSize__  
One of the render primitives we're able to choose from is GL_POINTS in which case each single vertex is a primitive and rendered as a point. We can also influence this value in the vertex shader.  
To enable  
```C++
glEnable(GL_PROGRAM_POINT_SIZE);
```
```GLSL
void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    gl_PointSize = gl_Position.z; // Farther, smaller   
}
```  
__gl_VertexID__  
The integer variable gl_VertexID holds the current ID of the vertex we're drawing. When doing indexed rendering (with glDrawElements) this variable holds the current index of the vertex we're drawing.  When drawing without indices (via glDrawArrays) this variable holds the number of the currently processed vertex since the start of the render call.
### Fragment shader variables
__gl_FrontFacing__  
The gl_FragCoord's x and y component are the window- or screen-space coordinates of the fragment, originating from the bottom-left of the window. The gl_FragCoord's z is equal to the depth value of that particular fragment.
```GLSL
void main()
{             
    if(gl_FragCoord.x < 400)
        FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    else
        FragColor = vec4(0.0, 1.0, 0.0, 1.0);        
}  
```
__gl_FrontFacing__  
The gl_FrontFacing variable tells us if the current fragment is part of a front-facing or a back-facing face. 
The gl_FrontFacing variable is a bool that is true if the fragment is part of a front face and false otherwise.  
```GLSL
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D frontTexture;
uniform sampler2D backTexture;

void main()
{             
    if(gl_FrontFacing)
        FragColor = texture(frontTexture, TexCoords);
    else
        FragColor = texture(backTexture, TexCoords);
}  
```
__gl_FragDepth__  
The input variable gl_FragCoord is a read-only variable. We can't influence the screen-space coordinates of the fragment, but it is possible to set the depth value of the fragment. GLSL gives us an output variable called gl_FragDepth that we can use to manually set the depth value of the fragment within the shader.

To set the depth value in the shader we write any value between 0.0 and 1.0 to the output variable.
```GLSL
gl_FragDepth = 0.0; // this fragment now has a depth value of 0.0
```
If the shader does not write anything to gl_FragDepth, the variable will automatically take its value from gl_FragCoord.z.  

Setting the depth value manually has __a major disadvantage__ however. That is because OpenGL __disables early depth testing__ (as discussed in the depth testing chapter) as soon as we __write to gl_FragDepth in the fragment shader__. It is disabled, because OpenGL __cannot know what depth value the fragment will have before we run the fragment shader__, since the fragment shader may actually change this value. 

By writing to gl_FragDepth you should take this performance penalty into consideration. __From OpenGL 4.2__ however, we can still sort of mediate between both sides by redeclaring the gl_FragDepth variable at the top of the fragment shader with a depth condition:
```GLSL
layout (depth_<condition>) out float gl_FragDepth;
```
![1674414689334](https://user-images.githubusercontent.com/98029669/213935238-193b9249-7390-4c9b-a11b-2a5654d73a04.png)  
By specifying greater or less as the depth condition, OpenGL can make the assumption that you'll only write depth values larger or smaller than the fragment's depth value.  
An example of where we increase the depth value in the fragment shader, but still want to preserve some of the early depth testing is shown in the fragment shader below:
```GLSL
#version 420 core // note the GLSL version!
out vec4 FragColor;
layout (depth_greater) out float gl_FragDepth;

void main()
{             
    FragColor = vec4(1.0);
    gl_FragDepth = gl_FragCoord.z + 0.1;
}  
```



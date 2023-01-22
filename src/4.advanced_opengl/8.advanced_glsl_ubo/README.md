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
## Interface Blocks
Interface blocks that allows us to group variables together.   
__Vertex Shader__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoords;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

out VS_OUT
{
    vec2 TexCoords;
} vs_out;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    vs_out.TexCoords = aTexCoords;
}  
```
__Fragment Shader__  
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoords;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

out VS_OUT
{
    vec2 TexCoords;
} vs_out;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    vs_out.TexCoords = aTexCoords;
}  
```
## Uniform Buffer Objects(UBO)
OpenGL gives us a tool called uniform buffer objects that allow us to declare a set of global uniform variables that remain the same over any number of shader programs. When using uniform buffer objects we set the relevant uniforms only once in fixed GPU memory. __I.e. the global const__  
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};

uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}  
```
In most of our samples we set a projection and view uniform matrix every frame for each shader we're using. This is a perfect example of where uniform buffer objects become useful since now we only have to store these matrices once.  
The layout (std140) statement represents the currently defined uniform block uses a specific memory layout for its content; this statement sets the uniform block layout.
## Uniform Block Layout
The content of a uniform block is a reserved piece of global GPU memory. Because this piece of memory holds no information on what kind of data it holds, we need to tell OpenGL what parts of the memory correspond to which uniform variables in the shader.

The size of each of the elements is clearly stated in OpenGL and directly corresponds to C++ data types. 
What OpenGL doesn't clearly state is the __spacing between the variables__. This allows the hardware to position or pad variables as it sees fit. The hardware is able to place a vec3 adjacent to a float for example. 

By default, GLSL uses a __uniform memory layout called a shared layout__ - shared because once the offsets are defined by the hardware, they are consistently shared between multiple programs. With a shared layout GLSL is allowed to __reposition the uniform variables__ for optimization as long as the variables' order remains intact.

Because we don't know at what offset each uniform variable will be we don't know how to precisely fill our uniform buffer. We can __query this information with functions like glGetUniformIndices__.

The general practice however is to not use the shared layout, but to __use the std140 layout__. The std140 layout __explicitly states the memory layout for each variable type by standardizing their respective offsets__ governed by a set of rules. 

Each variable has a __base alignment__ equal to the space a variable takes __(including padding)__ within a uniform block using the __std140__ layout rules. For each variable, we __calculate its aligned offset__: the byte offset of a variable from the start of the block. 

Each variable type in GLSL such as int, float and bool are defined to be four-byte quantities with each entity of 4 bytes represented as N. Check alignment below:  
![1674417527269](https://user-images.githubusercontent.com/98029669/213937505-69cb86a2-92f2-4e31-a062-0ad1c6a07c17.png)  
Note: vec4 is 16 bytes for int, float and bool.  
An example
```GLSL
layout (std140) uniform ExampleBlock
{
                     // base alignment  // aligned offset
    float value;     // 4               // 0 
    vec3 vector;     // 16              // 16  (offset must be multiple of 16 so 4->16)
    mat4 matrix;     // 16              // 32  (column 0)
                     // 16              // 48  (column 1)
                     // 16              // 64  (column 2)
                     // 16              // 80  (column 3)
    float values[3]; // 16              // 96  (values[0])
                     // 16              // 112 (values[1])
                     // 16              // 128 (values[2])
    bool boolean;    // 4               // 144
    int integer;     // 4               // 148
}; 
```
Apart __shared__ layout, the other remaining layout being __packed__. When using the packed layout, there is __no guarantee that the layout remains the same between programs (not shared, shared is the same between different program)__ because it allows the compiler to optimize uniform variables away from the uniform block which may differ per shader.
## Using UBO
```C++
unsigned int uboExampleBlock;
glGenBuffers(1, &uboExampleBlock);
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
glBufferData(GL_UNIFORM_BUFFER, 152, NULL, GL_STATIC_DRAW); // allocate 152 bytes of memory
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```
Now whenever we want to update or insert data into the buffer, we __bind to uboExampleBlock and use glBufferSubData__ to update its memory. We only have to update this uniform buffer once, and __all shaders that use this buffer__ now use its updated data.

How does OpenGL know what uniform buffers correspond to which uniform blocks?
In the OpenGL context there is a number of binding points defined where we can link a uniform buffer to. 
![image](https://user-images.githubusercontent.com/98029669/213937709-624c7371-351d-4217-9669-d8d6bddacc6a.png)  
Because shader A and shader B both have a uniform block __linked to the same binding point 0__, their uniform blocks share the same uniform data found in uboMatrices; a requirement being that __both shaders defined the same Matrices uniform block__.

```C++
unsigned int lights_index = glGetUniformBlockIndex(shaderA.ID, "Lights"); // Get target shader ID and the uniform's ID  
glUniformBlockBinding(shaderA.ID, lights_index, 2); // Bind to the NO.2 UBO data channel
```

P.S. From OpenGL version 4.2, you can direct declare in shader
```GLSL
layout(std140, binding = 2) uniform Lights { ... };
```

Bind an UBO to the UBO data channel(binding point)
```C++
glBindBufferBase(GL_UNIFORM_BUFFER, 2, uboExampleBlock); 
// or
glBindBufferRange(GL_UNIFORM_BUFFER, 2, uboExampleBlock, 0, 152);
```
The function __glBindbufferBase__ expects a __target__, a __binding point__ index and a __uniform buffer object__. This function links uboExampleBlock to binding point 2; from this point on, both sides of the binding point are linked. You can also use __glBindBufferRange__ that expects __an extra offset and size parameter__ - this way you can bind only __a specific range of__ the uniform buffer to a binding point. Using glBindBufferRange you could __have multiple different uniform blocks linked to a single uniform buffer object.__

To update data in UBO
```C++
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock); // first bind target UBO
int b = true; // bools in GLSL are represented as 4 bytes, so we store it in an integer
glBufferSubData(GL_UNIFORM_BUFFER, 144, 4, &b);  // target, offset, size, location of variable
glBindBuffer(GL_UNIFORM_BUFFER, 0); // back to default
```

## An Example of Using UBO
__Vertex Shader__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
}; // uniform to store global projection and view matrices
uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}
```
Bind all uniform to the same bingding point.
```C++
unsigned int uniformBlockIndexRed    = glGetUniformBlockIndex(shaderRed.ID, "Matrices");
unsigned int uniformBlockIndexGreen  = glGetUniformBlockIndex(shaderGreen.ID, "Matrices");
unsigned int uniformBlockIndexBlue   = glGetUniformBlockIndex(shaderBlue.ID, "Matrices");
unsigned int uniformBlockIndexYellow = glGetUniformBlockIndex(shaderYellow.ID, "Matrices");  

glUniformBlockBinding(shaderRed.ID,    uniformBlockIndexRed, 0);
glUniformBlockBinding(shaderGreen.ID,  uniformBlockIndexGreen, 0);
glUniformBlockBinding(shaderBlue.ID,   uniformBlockIndexBlue, 0);
glUniformBlockBinding(shaderYellow.ID, uniformBlockIndexYellow, 0);
```
Create an UBO and bind to the 0 binding point
```C++
unsigned int uboMatrices
glGenBuffers(1, &uboMatrices);

glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferData(GL_UNIFORM_BUFFER, 2 * sizeof(glm::mat4), NULL, GL_STATIC_DRAW);
glBindBuffer(GL_UNIFORM_BUFFER, 0);

glBindBufferRange(GL_UNIFORM_BUFFER, 0, uboMatrices, 0, 2 * sizeof(glm::mat4));
```
Fill the buffer. (Here, we keep the field of view value constant of the projection matrix (so no more camera zoom))
```C++
glm::mat4 projection = glm::perspective(glm::radians(45.0f), (float)width/(float)height, 0.1f, 100.0f); // Fixed FOV
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, 0, sizeof(glm::mat4), glm::value_ptr(projection));
glBindBuffer(GL_UNIFORM_BUFFER, 0);

glm::mat4 view = camera.GetViewMatrix();           
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, sizeof(glm::mat4), sizeof(glm::mat4), glm::value_ptr(view));
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```
Draw four cubes and their shaders shared the same porjection and view matrices
```C++
glBindVertexArray(cubeVAO);
shaderRed.use();
glm::mat4 model = glm::mat4(1.0f);
model = glm::translate(model, glm::vec3(-0.75f, 0.75f, 0.0f));	// move top-left
shaderRed.setMat4("model", model);
glDrawArrays(GL_TRIANGLES, 0, 36);        
// ... draw Green Cube
// ... draw Blue Cube
// ... draw Yellow Cube	 
```

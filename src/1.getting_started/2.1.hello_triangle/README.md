# Hello Triangle
顶点数组对象：Vertex Array Object，VAO  
顶点缓冲对象：Vertex Buffer Object，VBO  
元素缓冲对象：Element Buffer Object，EBO 或 索引缓冲对象 Index Buffer Object，IBO
 ![pipeline](https://user-images.githubusercontent.com/98029669/212928856-1a7c1b24-d998-4067-b133-4e888cb202d9.png)
 ![NDC](https://user-images.githubusercontent.com/98029669/212930015-27f631a2-53de-4d5d-a4d9-478b76e1429a.png)
 ## Vertex Shader
 To set the output of the vertex shader we have to assign the position data to the predefined gl_Position variable which is a vec4 behind the scenes. 
```shell
#version 330 core
layout (location = 0) in vec3 aPos;

void main()
{
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```
```shell
unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);
// check for shader compile errors
int success;
char infoLog[512];
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
if (!success)
{
    glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
}
```
![glshadersource](https://user-images.githubusercontent.com/98029669/212934681-4c77ed92-0007-486d-8a34-64e3a0d83440.png)

 ## Fragment Shader
 The fragment shader only requires one output variable and that is a vector of size 4 that defines the final color output that we should calculate ourselves.
 We can declare output values with the out keyword, that we here promptly named FragColor.
```shell
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
} 
```
```shell
unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
glCompileShader(fragmentShader);
// check for shader compile errors
glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &success);
if (!success)
{
    glGetShaderInfoLog(fragmentShader, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::FRAGMENT::COMPILATION_FAILED\n" << infoLog << std::endl;
}
```

## Shader Program Object
```shell
// link shaders
unsigned int shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);
// check for linking errors
glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
if (!success) {
    glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
}
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);
```

## Vertex Input
```shell
float vertices[] = {
    -0.5f, -0.5f, 0.0f, // left  
    0.5f, -0.5f, 0.0f, // right 
    0.0f,  0.5f, 0.0f  // top   
}; 
```
With the vertex data defined we'd like to send it as input to the first process of the graphics pipeline: the vertex shader. 
This is done by creating memory on the GPU where we store the vertex data, configure how OpenGL should interpret the memory and specify how to send the data to the graphics card. 
The vertex shader then processes as much vertices as we tell it to from its memory.
We manage this memory via so called vertex buffer objects (VBO) that can store a large number of vertices in the GPU's memory. 
The advantage of using those buffer objects is that we can send large batches of data all at once to the graphics card, and keep it there if there's enough memory left, without having to send data one vertex at a time. 
Sending data to the graphics card from the CPU is relatively slow, so wherever we can we try to send as much data as possible at once.
```shell
unsigned int VBO, VAO;
glGenVertexArrays(1, &VAO);
glGenBuffers(1, &VBO);
// bind the Vertex Array Object first, then bind and set vertex buffer(s), and then configure vertex attributes(s).
glBindVertexArray(VAO);

glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```
glBufferData is a function specifically targeted to copy user-defined data into the currently bound buffer. 
Its first argument is the type of the buffer we want to copy data into: the vertex buffer object currently bound to the GL_ARRAY_BUFFER target. 
The second argument specifies the size of the data (in bytes) we want to pass to the buffer; a simple sizeof of the vertex data suffices. 
The third parameter is the actual data we want to send.
The fourth parameter specifies how we want the graphics card to manage the given data. This can take 3 forms:  
GL_STREAM_DRAW: the data is set only once and used by the GPU at most a few times.  
GL_STATIC_DRAW: the data is set only once and used many times.  
GL_DYNAMIC_DRAW: the data is changed a lot and used many times.  



```shell
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

// note that this is allowed, the call to glVertexAttribPointer registered VBO as the vertex attribute's bound vertex buffer object so afterwards we can safely unbind
glBindBuffer(GL_ARRAY_BUFFER, 0); 

// You can unbind the VAO afterwards so other VAO calls won't accidentally modify this VAO, but this rarely happens. Modifying other
// VAOs requires a call to glBindVertexArray anyways so we generally don't unbind VAOs (nor VBOs) when it's not directly necessary.
glBindVertexArray(0); 
```
The function glVertexAttribPointer has quite a few parameters so let's carefully walk through them:

The first parameter specifies which vertex attribute we want to configure. Remember that we specified the location of the position vertex attribute in the vertex shader with layout (location = 0). This sets the location of the vertex attribute to 0 and since we want to pass data to this vertex attribute, we pass in 0.  
The next argument specifies the size of the vertex attribute. The vertex attribute is a vec3 so it is composed of 3 values.  
The third argument specifies the type of the data which is GL_FLOAT (a vec* in GLSL consists of floating point values).  
The next argument specifies if we want the data to be normalized. If we're inputting integer data types (int, byte) and we've set this to GL_TRUE, the integer data is normalized to 0 (or -1 for signed data) and 1 when converted to float. This is not relevant for us so we'll leave this at GL_FALSE.  
The fifth argument is known as the stride and tells us the space between consecutive vertex attributes. Since the next set of position data is located exactly 3 times the size of a float away we specify that value as the stride. Note that since we know that the array is tightly packed (there is no space between the next vertex attribute value) we could've also specified the stride as 0 to let OpenGL determine the stride (this only works when values are tightly packed). Whenever we have more vertex attributes we have to carefully define the spacing between each vertex attribute but we'll get to see more examples of that later on.  
The last parameter is of type void* and thus requires that weird cast. This is the offset of where the position data begins in the buffer. Since the position data is at the start of the data array this value is just 0. We will explore this parameter in more detail later on.  
![vertex_attribute_pointer](https://user-images.githubusercontent.com/98029669/212988872-2b77f087-e8b7-43cf-8706-391edfe1b805.png)

## VAO
```shell
glUseProgram(shaderProgram);
glBindVertexArray(VAO); // seeing as we only have a single VAO there's no need to bind it every time, but we'll do so to keep things a bit more organized
glDrawArrays(GL_TRIANGLES, 0, 3);
// glBindVertexArray(0); // no need to unbind it every time 
```
![vertex_array_objects](https://user-images.githubusercontent.com/98029669/212989494-4f25b0f9-21e6-4b38-ab0e-c109e48d2261.png)

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

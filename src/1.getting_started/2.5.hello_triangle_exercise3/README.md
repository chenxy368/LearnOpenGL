# Hello Triangle
 Draw with two fragment shader
 ```C++
 //fragmentShader1Source
#version 330 core
out vec4 FragColor;
void main()
{
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
};

//fragmentShader2Source
#version 330 core
out vec4 FragColor;
void main()
{
    FragColor = vec4(1.0f, 1.0f, 0.0f, 1.0f);
};
 ```
 
Compile separately
```C++
unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
unsigned int fragmentShaderOrange = glCreateShader(GL_FRAGMENT_SHADER);
unsigned int fragmentShaderYellow = glCreateShader(GL_FRAGMENT_SHADER); 

unsigned int shaderProgramOrange = glCreateProgram();
unsigned int shaderProgramYellow = glCreateProgram(); 

glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);
glShaderSource(fragmentShaderOrange, 1, &fragmentShader1Source, NULL);
glCompileShader(fragmentShaderOrange);
glShaderSource(fragmentShaderYellow, 1, &fragmentShader2Source, NULL);
glCompileShader(fragmentShaderYellow);

glAttachShader(shaderProgramOrange, vertexShader);
glAttachShader(shaderProgramOrange, fragmentShaderOrange);
glLinkProgram(shaderProgramOrange);

glAttachShader(shaderProgramYellow, vertexShader);
glAttachShader(shaderProgramYellow, fragmentShaderYellow);
glLinkProgram(shaderProgramYellow);
```
Use different GLSL program
```C++
glUseProgram(shaderProgramOrange);
glBindVertexArray(VAOs[0]);
glDrawArrays(GL_TRIANGLES, 0, 3);	
glUseProgram(shaderProgramYellow);
glBindVertexArray(VAOs[1]);
glDrawArrays(GL_TRIANGLES, 0, 3);
```

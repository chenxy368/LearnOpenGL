# Shaders
 ## More Attributes in vertex shaders
 Add color
 ```shell
 float vertices[] = {
    // positions         // colors
     0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,   // bottom right
    -0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,   // bottom left
     0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f    // top 
}; 
 ```
 
layout 1 in vertex shader is color. Different attribute position.
 ```shell
#version 330 core
layout (location = 0) in vec3 aPos;   // the position variable has attribute position 0
layout (location = 1) in vec3 aColor; // the color variable has attribute position 1
  
out vec3 ourColor; // output a color to the fragment shader

void main()
{
    gl_Position = vec4(aPos, 1.0);
    ourColor = aColor; // set ourColor to the input color we got from the vertex data
} 
 ```
 From vertex shader to fragment shader: ourColor
 ```shell
#version 330 core
out vec4 FragColor;  
in vec3 ourColor;
  
void main()
{
    FragColor = vec4(ourColor, 1.0);
}
 ```
 ![vertex_attribute_pointer_interleaved](https://user-images.githubusercontent.com/98029669/213026091-f90383aa-552b-404e-bb3f-429c9bbff455.png)  
0: position stride = 6 * sizeof(float) offset = 0  
1: color stride = 6 * sizeof(float) offset = 3 * sizeof(float)  
```shell
// position attribute
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// color attribute
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3* sizeof(float)));
glEnableVertexAttribArray(1);
```
 

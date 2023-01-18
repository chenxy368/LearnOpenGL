# Shaders
 ## Shaders Template
 ```GLSL
 #version version_number
in type in_variable_name;
in type in_variable_name;

out type out_variable_name;
  
uniform type uniform_name;
  
void main()
{
    // process input(s) and do some weird graphics stuff
    ...
    // output processed stuff to output variable
    out_variable_name = weird_stuff_we_processed;
}
 ```
 ## Types
vecn: the default vector of n floats.  
bvecn: a vector of n booleans.  
ivecn: a vector of n integers.  
uvecn: a vector of n unsigned integers.  
dvecn: a vector of n double components.  
```GLSL
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;

vec2 vect = vec2(0.5, 0.7);
vec4 result = vec4(vect, 0.0, 0.0);
vec4 otherResult = vec4(result.xyz, 1.0);
```

## In and Out
Vertex shader to fragment shader: vertexColor
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos; // the position variable has attribute position 0
  
out vec4 vertexColor; // specify a color output to the fragment shader

void main()
{
    gl_Position = vec4(aPos, 1.0); // see how we directly give a vec3 to vec4's constructor
    vertexColor = vec4(0.5, 0.0, 0.0, 1.0); // set the output variable to a dark-red color
}
```
```GLSL
#version 330 core
out vec4 FragColor;
  
in vec4 vertexColor; // the input variable from the vertex shader (same name and same type)  

void main()
{
    FragColor = vertexColor;
}
```

## Uniform
Uniforms are another way to pass data from our application on the CPU to the shaders on the GPU.
```GLSL
#version 330 core
out vec4 FragColor;
  
uniform vec4 ourColor; // we set this variable in the OpenGL code.

void main()
{
    FragColor = ourColor;
}   
```
Because OpenGL is in its core a C library it does not have native support for function overloading, 
so wherever a function can be called with different types OpenGL defines new functions for each type required;
glUniform is a perfect example of this. The function requires a specific postfix for the type of the uniform you want to set. 
A few of the possible postfixes are:  
f: the function expects a float as its value.  
i: the function expects an int as its value.  
ui: the function expects an unsigned int as its value.  
3f: the function expects 3 floats as its value.  
fv: the function expects a float vector/array as its value.  
Whenever you want to configure an option of OpenGL simply pick the overloaded function that corresponds with your type. 
In our case we want to set 4 floats of the uniform individually so we pass our data via glUniform4f (note that we also could've used the fv version).  
```C++
double timeValue = glfwGetTime();
float greenValue = static_cast<float>(sin(timeValue) / 2.0 + 0.5);
// Find uniform position by its name
int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```

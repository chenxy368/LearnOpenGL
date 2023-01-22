# Geometry Shader
![pipeline](https://user-images.githubusercontent.com/98029669/213938508-88a6166e-3cdb-4040-91d0-88bffe2a0862.png)  
Between the vertex and the fragment shader there is an optional shader stage called the geometry shader. 
it is able to convert the original primitive (set of vertices) to completely different primitives, possibly generating more vertices than were initially given.

An example of get points and turn to line strips.
```GLSL
#version 330 core
layout (points) in;
layout (line_strip, max_vertices = 2) out;

void main() {    
    gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0); 
    EmitVertex();

    gl_Position = gl_in[0].gl_Position + vec4( 0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    EndPrimitive();
}
```

This __input layout qualifier__ can take any of the following primitive values(Least required vertices in the parenthesis):

points: when drawing GL_POINTS primitives (1).  
lines: when drawing GL_LINES or GL_LINE_STRIP (2).  
lines_adjacency: GL_LINES_ADJACENCY or GL_LINE_STRIP_ADJACENCY (4).  
triangles: GL_TRIANGLES, GL_TRIANGLE_STRIP or GL_TRIANGLE_FAN (3).  
triangles_adjacency : GL_TRIANGLES_ADJACENCY or GL_TRIANGLE_STRIP_ADJACENCY (6).  

The __output layout qualifier__:

points(1)  
line_strip(2)  
triangle_strip(3)  

__max_vertices = 2:__  Do within the layout qualifier of the out keyword.(maximum number of 2 vertices)

__What is gl_in[0]?__
```GLSL
in gl_Vertex
{
    vec4  gl_Position;
    float gl_PointSize;
    float gl_ClipDistance[];
} gl_in[];
```

Each time we call __EmitVertex__, the vector currently set to __gl_Position__ is added to the output primitive. 
Whenever __EndPrimitive__ is called, __all emitted vertices__ for this primitive are combined into the specified output render primitive. 

We can use this line to draw line strips with point primitive based on the geometry shader
```C++
glDrawArrays(GL_POINTS, 0, 4);
```
## Using Geometry Shaders
```C++
geometryShader = glCreateShader(GL_GEOMETRY_SHADER);
glShaderSource(geometryShader, 1, &gShaderCode, NULL);
glCompileShader(geometryShader);  
...
glAttachShader(program, geometryShader);
glLinkProgram(program);
```
Check in includes/learnopengl/shader.h, the geometry shader is an optional field.
```C++
Shader(const char* vertexPath, const char* fragmentPath, const char* geometryPath = nullptr)
{
    ...
    std::string geometryCode;
    ...
    std::ifstream gShaderFile;
    // ensure ifstream objects can throw exceptions:
    ...
    gShaderFile.exceptions (std::ifstream::failbit | std::ifstream::badbit);
    ...
    // if geometry shader is given, compile geometry shader
    unsigned int geometry;
    if(geometryPath != nullptr)
    {
        const char * gShaderCode = geometryCode.c_str();
        geometry = glCreateShader(GL_GEOMETRY_SHADER);
        glShaderSource(geometry, 1, &gShaderCode, NULL);
        glCompileShader(geometry);
        checkCompileErrors(geometry, "GEOMETRY");
    }
    // shader Program
    ID = glCreateProgram();
    glAttachShader(ID, vertex);
    glAttachShader(ID, fragment);
    if(geometryPath != nullptr)
        glAttachShader(ID, geometry);
    glLinkProgram(ID);
    checkCompileErrors(ID, "PROGRAM");
    ...
    if(geometryPath != nullptr)
        glDeleteShader(geometry);
}
 ```
 ## Build a House Example
 A triangle strip in OpenGL is a more efficient way to draw triangles with fewer vertices. 
 After the first triangle is drawn, each subsequent vertex generates another triangle next to the first triangle: every 3 adjacent vertices will form a triangle.   
 ![image](https://user-images.githubusercontent.com/98029669/213939976-b1b2c6fb-6f49-40cb-8ebc-df3d2397f1a6.png)  
 __Vertex Shader__
 ```GLSL
 #version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;

out VS_OUT {
    vec3 color;
} vs_out; // Output the color to geometry shader

void main()
{
    vs_out.color = aColor;
    gl_Position = vec4(aPos.x, aPos.y, 0.0, 1.0); 
}
```
 __Geometry Shader__
 ```GLSL
#version 330 core
layout (points) in;
layout (triangle_strip, max_vertices = 5) out;

in VS_OUT {
    vec3 color;
} gs_in[]; // Get color from vertex shader

out vec3 fColor; // Output color to fragment shader

void build_house(vec4 position)
{    
    fColor = gs_in[0].color; // gs_in[0] since there's only one input vertex
    gl_Position = position + vec4(-0.2, -0.2, 0.0, 0.0); // 1:bottom-left   
    EmitVertex();   
    gl_Position = position + vec4( 0.2, -0.2, 0.0, 0.0); // 2:bottom-right
    EmitVertex();
    gl_Position = position + vec4(-0.2,  0.2, 0.0, 0.0); // 3:top-left
    EmitVertex();
    gl_Position = position + vec4( 0.2,  0.2, 0.0, 0.0); // 4:top-right
    EmitVertex();
    gl_Position = position + vec4( 0.0,  0.4, 0.0, 0.0); // 5:top
    fColor = vec3(1.0, 1.0, 1.0);
    EmitVertex();
    EndPrimitive();
}

void main() {    
    build_house(gl_in[0].gl_Position);
}
```
__Fargment Shader__
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 fColor; // Get color from geometry shader

void main()
{
    FragColor = vec4(fColor, 1.0);   
}
```

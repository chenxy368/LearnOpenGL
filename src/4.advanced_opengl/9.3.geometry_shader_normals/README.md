# Geometry Shader
## Visualizing Normal Vectors
First do not use geometry shader to render. Second use geometry shader to render normal.
```C++
shader.use();
DrawScene();
normalDisplayShader.use();
DrawScene();
```
_For the second step:_
__Vertex Shader__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal; // Use given normal

out VS_OUT {
    vec3 normal;
} vs_out; // Output normal to geometry shader

uniform mat4 view;
uniform mat4 model;

void main()
{
    gl_Position = view * model * vec4(aPos, 1.0); 
    mat3 normalMatrix = mat3(transpose(inverse(view * model))); // Remember deal with scaling
    vs_out.normal = normalize(vec3(vec4(normalMatrix * aNormal, 0.0)));
}
```
__Fragment Shader__
```GLSL
#version 330 core
layout (triangles) in;
layout (line_strip, max_vertices = 6) out;

in VS_OUT {
    vec3 normal;
} gs_in[]; // Get normal

const float MAGNITUDE = 0.4;
  
uniform mat4 projection;

// Draw a line to represent normal
void GenerateLine(int index)
{
    gl_Position = projection * gl_in[index].gl_Position;
    EmitVertex();
    gl_Position = projection * (gl_in[index].gl_Position + vec4(gs_in[index].normal, 0.0) * MAGNITUDE);
    EmitVertex();
    EndPrimitive();
}

void main()
{
    GenerateLine(0); // first vertex normal
    GenerateLine(1); // second vertex normal
    GenerateLine(2); // third vertex normal
}  
```
__Fragment Shader__
```
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0, 1.0, 0.0, 1.0);
}
```

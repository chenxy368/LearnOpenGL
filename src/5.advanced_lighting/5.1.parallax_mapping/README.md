# Parallax Mapping
The idea behind parallax mapping is to alter the texture coordinates in such a way that it looks like a fragment's surface is higher or lower than it actually is, all based on the view direction and a heightmap.
![image](https://user-images.githubusercontent.com/98029669/214394380-d3f43a99-1903-4272-9e83-ce85fa2033c6.png)  
When there is a hight difference, the texture will have an offset since when you look at some point on the bottom plane, you will see the texture on a point that is not perpendicular to it.  
![image](https://user-images.githubusercontent.com/98029669/214394724-6455e9fc-91a9-41da-8ff7-ad74c71507b4.png)
It's just a trick to divide x and y by z. When z is smaller, our view direction is almost parallel to the bottom plane, which means we need a larger offset.
![image](https://user-images.githubusercontent.com/98029669/214395300-0c3270e3-a055-4643-af27-5d180974cc94.png)  
__Vertex Shader:__
Multiply inverse TBN to transform to the TBN space
```GLSL
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
layout (location = 3) in vec3 tangent;
layout (location = 4) in vec3 bitangent;

out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

uniform vec3 lightPos;
uniform vec3 viewPos;

void main()
{
    gl_Position      = projection * view * model * vec4(position, 1.0f);
    vs_out.FragPos   = vec3(model * vec4(position, 1.0));   
    vs_out.TexCoords = texCoords;    

    vec3 T   = normalize(mat3(model) * tangent);
    vec3 B   = normalize(mat3(model) * bitangent);
    vec3 N   = normalize(mat3(model) * normal);
    mat3 TBN = transpose(mat3(T, B, N));

    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vs_out.FragPos;
}
```
__Fragment Shader__
```GLSL
#version 330 core
out vec4 FragColor;

in VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} fs_in;

uniform sampler2D diffuseMap;
uniform sampler2D normalMap;
uniform sampler2D depthMap;

uniform float height_scale;

vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir);

void main()
{           
    // Offset texture coordinates with Parallax Mapping
    vec3 viewDir   = normalize(fs_in.TangentViewPos - fs_in.TangentFragPos);
    vec2 texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);

    // then sample textures with new texture coords
    vec3 diffuse = texture(diffuseMap, texCoords);
    vec3 normal  = texture(normalMap, texCoords);
    normal = normalize(normal * 2.0 - 1.0);
    // proceed with lighting code
    [...]    
}

vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    float height =  texture(depthMap, texCoords).r;    
    vec2 p = viewDir.xy / viewDir.z * (height * height_scale);
    // Simply offset from the original texCoords, remember depthMap store the height
    return texCoords - p;    
}
```
__Import: near the boundary, the texCoord can get out of range, need a filter.__
```GLSL
texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
if(texCoords.x > 1.0 || texCoords.y > 1.0 || texCoords.x < 0.0 || texCoords.y < 0.0)
    discard;
``` 
Note that this trick doesn't work on all types of surfaces.

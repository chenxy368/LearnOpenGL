# Normal Mapping
Most real-life surface aren't flat however and exhibit a lot of (bumpy) details.
The lighting doesn't take any of the small cracks and holes into account and completely ignores the deep stripes between the bricks; the surface looks perfectly flat. 
. What if we, instead of a per-surface normal that is the same for each fragment, use a per-fragment normal that is different for each fragment? This way we can slightly deviate the normal vector based on a surface's little details; 
this gives the illusion the surface is a lot more complex:  
![image](https://user-images.githubusercontent.com/98029669/214222896-e01ca6b8-236a-40d5-93c1-b63e7d6d8c4c.png)  

## Normal Mapping
Color vectors in a texture are represented as a 3D vector with an r, g, and b component.
We can similarly store a normal vector's x, y and z component in the respective color components. 
Normal vectors range between -1 and 1 so they're first mapped to [0,1]:
```GLSL
vec3 rgb_normal = normal * 0.5 + 0.5; // transforms from [-1,1] to [0,1]  
```
![image](https://user-images.githubusercontent.com/98029669/214223174-9c5b0a15-e81e-4958-8e28-52ff93a6d9c6.png)  
This (and almost all normal maps you find online) will have a blue-ish tint. 
This is because the normals are all closely pointing outwards towards the positive z-axis  (0,0,1).
__However we do not know the specific position of object in world space when we create the normal map.__

Use in a naive way:
```GLSL
uniform sampler2D normalMap;  

void main()
{           
    // obtain normal from normal map in range [0,1]
    normal = texture(normalMap, fs_in.TexCoords).rgb;
    // transform normal vector to range [-1,1]
    normal = normalize(normal * 2.0 - 1.0);   
  
    [...]
    // proceed with lighting as normal
}  
```
![image](https://user-images.githubusercontent.com/98029669/214223679-6957e168-0776-4734-96e5-880f85b0d720.png)  
Cannot correct because the normal vectors are still roughly point to the positive z direction.  
## Tangent Space
Tangent space is a space that's local to the surface of a triangle: the normals are relative to the local reference frame of the individual triangles.
Think of it as the local space of the normal map's vectors; they're all defined pointing in the positive z direction regardless of the final transformed direction.
A change-of-basis matrix is called a TBN matrix where the letters depict a Tangent, Bitangent and Normal vector. It transforms a tangent-space vector to a different coordinate space, 
we need three perpendicular vectors that are aligned along the surface of a normal map: an up, right, and forward vector.  
![image](https://user-images.githubusercontent.com/98029669/214226466-6d7a071a-2dc5-4fca-94e2-3d3a364d89aa.png)  
![image](https://user-images.githubusercontent.com/98029669/214228864-7794e3d6-c3d0-4ceb-bc57-9b86f8bcb7fb.png)

## Manual calculation of tangents and bitangents
Assume
```GLSL
// positions
glm::vec3 pos1(-1.0,  1.0, 0.0);
glm::vec3 pos2(-1.0, -1.0, 0.0);
glm::vec3 pos3( 1.0, -1.0, 0.0);
glm::vec3 pos4( 1.0,  1.0, 0.0);
// texture coordinates
glm::vec2 uv1(0.0, 1.0);
glm::vec2 uv2(0.0, 0.0);
glm::vec2 uv3(1.0, 0.0);
glm::vec2 uv4(1.0, 1.0);
// normal vector
glm::vec3 nm(0.0, 0.0, 1.0);  
```
TBN can be calculated:
```GLSL
glm::vec3 edge1 = pos2 - pos1;
glm::vec3 edge2 = pos3 - pos1;
glm::vec2 deltaUV1 = uv2 - uv1;
glm::vec2 deltaUV2 = uv3 - uv1;  

float f = 1.0f / (deltaUV1.x * deltaUV2.y - deltaUV2.x * deltaUV1.y);

tangent1.x = f * (deltaUV2.y * edge1.x - deltaUV1.y * edge2.x);
tangent1.y = f * (deltaUV2.y * edge1.y - deltaUV1.y * edge2.y);
tangent1.z = f * (deltaUV2.y * edge1.z - deltaUV1.y * edge2.z);

bitangent1.x = f * (-deltaUV2.x * edge1.x + deltaUV1.x * edge2.x);
bitangent1.y = f * (-deltaUV2.x * edge1.y + deltaUV1.x * edge2.y);
bitangent1.z = f * (-deltaUV2.x * edge1.z + deltaUV1.x * edge2.z);
  
[...] // similar procedure for calculating tangent/bitangent for plane's second triangle
```
In renderQuad, vectors are computed:
```C++
float quadVertices[] = {
    // positions            // normal         // texcoords  // tangent                          // bitangent
    pos1.x, pos1.y, pos1.z, nm.x, nm.y, nm.z, uv1.x, uv1.y, tangent1.x, tangent1.y, tangent1.z, bitangent1.x, bitangent1.y, bitangent1.z,
    pos2.x, pos2.y, pos2.z, nm.x, nm.y, nm.z, uv2.x, uv2.y, tangent1.x, tangent1.y, tangent1.z, bitangent1.x, bitangent1.y, bitangent1.z,
    pos3.x, pos3.y, pos3.z, nm.x, nm.y, nm.z, uv3.x, uv3.y, tangent1.x, tangent1.y, tangent1.z, bitangent1.x, bitangent1.y, bitangent1.z,

    pos1.x, pos1.y, pos1.z, nm.x, nm.y, nm.z, uv1.x, uv1.y, tangent2.x, tangent2.y, tangent2.z, bitangent2.x, bitangent2.y, bitangent2.z,
    pos3.x, pos3.y, pos3.z, nm.x, nm.y, nm.z, uv3.x, uv3.y, tangent2.x, tangent2.y, tangent2.z, bitangent2.x, bitangent2.y, bitangent2.z,
    pos4.x, pos4.y, pos4.z, nm.x, nm.y, nm.z, uv4.x, uv4.y, tangent2.x, tangent2.y, tangent2.z, bitangent2.x, bitangent2.y, bitangent2.z
};
// configure plane VAO
glGenVertexArrays(1, &quadVAO);
glGenBuffers(1, &quadVBO);
glBindVertexArray(quadVAO);
glBindBuffer(GL_ARRAY_BUFFER, quadVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(quadVertices), &quadVertices, GL_STATIC_DRAW);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 14 * sizeof(float), (void*)0);
glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 14 * sizeof(float), (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(2);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 14 * sizeof(float), (void*)(6 * sizeof(float)));
glEnableVertexAttribArray(3);
glVertexAttribPointer(3, 3, GL_FLOAT, GL_FALSE, 14 * sizeof(float), (void*)(8 * sizeof(float)));
glEnableVertexAttribArray(4);
glVertexAttribPointer(4, 3, GL_FLOAT, GL_FALSE, 14 * sizeof(float), (void*)(11 * sizeof(float)));
```
## Tangent space normal mapping
1. Get vertex attributes
```GLSL
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
layout (location = 3) in vec3 tangent;
layout (location = 4) in vec3 bitangent;
```
2. Create TBN
```GLSL
mat3 normalMatrix = transpose(inverse(mat3(model)));
vec3 T = normalize(normalMatrix * aTangent);
vec3 N = normalize(normalMatrix * aNormal);
// Gram-Schmidt process, re-orthogonalize T with respect to N
// This, albeit by a little, generally improves the normal mapping results with a little extra cost.
T = normalize(T - dot(T, N) * N);
// B is perpendicular to both N and T
vec3 B = cross(N, T);
    
mat3 TBN = transpose(mat3(T, B, N));   
```
3. Change all required position to tangent space, prevent any matrix compuation in fragment shader: 
__usually run more fragment shader, thinking about one triangle with 3 vertices but can have many fragments.__
```GLSL
vs_out.TangentLightPos = TBN * lightPos;
vs_out.TangentViewPos  = TBN * viewPos;
vs_out.TangentFragPos  = TBN * vs_out.FragPos;
```
4. No change in fragment shader
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

uniform vec3 lightPos;
uniform vec3 viewPos;

void main()
{           
     // obtain normal from normal map in range [0,1]
    vec3 normal = texture(normalMap, fs_in.TexCoords).rgb;
    // transform normal vector to range [-1,1]
    normal = normalize(normal * 2.0 - 1.0);  // this normal is in tangent space
   
    // get diffuse color
    vec3 color = texture(diffuseMap, fs_in.TexCoords).rgb;
    // ambient
    vec3 ambient = 0.1 * color;
    // diffuse
    vec3 lightDir = normalize(fs_in.TangentLightPos - fs_in.TangentFragPos);
    float diff = max(dot(lightDir, normal), 0.0);
    vec3 diffuse = diff * color;
    // specular
    vec3 viewDir = normalize(fs_in.TangentViewPos - fs_in.TangentFragPos);
    vec3 reflectDir = reflect(-lightDir, normal);
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    float spec = pow(max(dot(normal, halfwayDir), 0.0), 32.0);

    vec3 specular = vec3(0.2) * spec;
    FragColor = vec4(ambient + diffuse + specular, 1.0);
}
```





# Basic Lightning
## Phong Lighting Model
![basic_lighting_phong](https://user-images.githubusercontent.com/98029669/213760281-795585ee-cc66-408a-bb1d-a558aa0222c2.png)
![Phong_components_version_4](https://user-images.githubusercontent.com/98029669/213760493-f4f8346b-4209-49c5-aaf4-9691c12bd710.png)
__Ambient lighting__: even when it is dark there is usually still some light somewhere in the world (the moon, a distant light) so objects are almost never completely dark. 
To simulate this we use an ambient lighting constant that always gives the object some color.  
__Diffuse lighting:__ simulates the directional impact a light object has on an object. This is the most visually significant component of the lighting model. 
The more a part of an object faces the light source, the brighter it becomes.  
__Specular lighting:__ simulates the bright spot of a light that appears on shiny objects. 
Specular highlights are more inclined to the color of the light than the color of the object. 
## Ambient Color
To effeiciently simulate background lightning, result equals to a factor(ambientStrength) multiply the lightColor and then multiply objectColor.(Compared to last section, just on more factor.)
```GLSL
void main()
{
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;

    vec3 result = ambient * objectColor;
    FragColor = vec4(result, 1.0);
}
```
![ambient_lighting](https://user-images.githubusercontent.com/98029669/213761662-53d44705-0658-4a81-91d7-f70a7cb107ef.png)
## Diffuse Color
![phong_diffuse](https://user-images.githubusercontent.com/98029669/213771498-77d59b7b-0a1a-4a46-b93f-7fa51f39d439.png)
## Add Normal to Vertex Attributes Array and Process in the shaders 
```C++
float vertices[] = {
    // Position           // Normal
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
     0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
        
     ...

    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,
     0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
     0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
    -0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f
};
```  
Add pointer to normals.
```C++
// first, configure the cube's VAO (and VBO)
...
// normal attribute
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1);

// second, configure the light's VAO(the wight cube) (VBO stays the same; the vertices are the same for the light object which is also a 3D cube)
...
// note that we update the lamp's position attribute's stride to reflect the updated buffer data, only position
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```
Add arguments in shaders  
__Vectex Shader:__
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
...
out vec3 Normal;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
    Normal = aNormal;
}
```
__Fragment Shader:__
```GLSL
...
in vec3 Normal;
...
```
## Compute Diffuse
__Fragment Shader:__  
1. Output the vertex color at the end  
2. Get vertex position and its normal from vertex shader  
3. Get light position, light color and object color from program as uniform  
4. First compute ambent color as shown before with multipling ambientStrength  
5. Compute dot product as coeffiecient(use max to filter negative) and multiply light color to represent the diffuse  
6. result = (ambient + diffuse) * objectColor  
P.S. Since we do not set FragPos, the FragPos always equals (0, 0, 0), which means parallel light.
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 Normal;  
in vec3 FragPos;  
  
uniform vec3 lightPos; 
uniform vec3 lightColor;
uniform vec3 objectColor;

void main()
{
    // ambient
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;
  	
    // diffuse 
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;
            
    vec3 result = (ambient + diffuse) * objectColor;
    FragColor = vec4(result, 1.0);
} 
```
![basic_lighting_diffuse](https://user-images.githubusercontent.com/98029669/213771900-750d49c0-77be-4164-acd2-cd63ba23d80e.png)

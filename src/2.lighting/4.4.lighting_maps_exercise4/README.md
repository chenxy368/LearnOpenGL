Also add something they call an emission map which is a texture that stores emission values per fragment.
Emission values are colors an object may emit as if it contains a light source itself; this way an object can glow regardless of the light conditions.   
![matrix](https://user-images.githubusercontent.com/98029669/213847411-2e09c36a-a2f6-47dc-a8a7-97da929d5f43.jpg)
![emissive](https://user-images.githubusercontent.com/98029669/213847397-546d1548-ade0-4f3e-bf06-836d8ce7e39c.png)
__Fragment Shader__
```GLSL
...
struct Material {
    sampler2D diffuse;
    sampler2D specular;    
    sampler2D emission;// Add emission same as diffuse and specular to use texture channel
    float shininess;
}; 
...
void main()
{
    ...
    // emission
    vec3 emission = texture(material.emission, TexCoords).rgb;
        
    vec3 result = ambient + diffuse + specular + emission;
    FragColor = vec4(result, 1.0);
} 
```
Add emision map
```C++
unsigned int emissionMap = loadTexture(FileSystem::getPath("resources/textures/matrix.jpg").c_str());
...
lightingShader.setInt("material.emission", 2);
...
    // bind emission map
    glActiveTexture(GL_TEXTURE2);
    glBindTexture(GL_TEXTURE_2D, emissionMap);
```

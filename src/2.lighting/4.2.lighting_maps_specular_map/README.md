## Lighting Maps
# Specular Map
Same as diffuse map, change specular to sample2D and use texture to get coefficient.
__Fragment Shader__
```GLSL
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    float     shininess;
};
...
void main()
{
    ...
    vec3 ambient  = light.ambient  * vec3(texture(material.diffuse, TexCoords));
    ...
    vec3 diffuse  = light.diffuse  * diff * vec3(texture(material.diffuse, TexCoords));  
    ...
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    
    FragColor = vec4(ambient + diffuse + specular, 1.0);
  ...
}
```
Also remeber to pass the uniform specular(pointer ID) in program
```C++
lightingShader.setInt("material.specular", 1);
...
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, specularMap);
```

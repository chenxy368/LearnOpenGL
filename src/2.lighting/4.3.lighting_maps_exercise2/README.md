Try inverting the color values of the specular map in the fragment shader so that the wood shows specular highlights and the steel borders do not.
```GLSL
...
    // here we inverse the sampled specular color. Black becomes white and white becomes black.
    vec3 specular = light.specular * spec * (vec3(1.0) - vec3(texture(material.specular, TexCoords))); 
``` 

Do Phong shading in view space instead of world space.   
__Vertex shader:__
```GLSL
#version 330 core
...
out vec3 LightPos; // Output the light position

uniform vec3 lightPos; // we now define the uniform in the vertex shader and pass the 'view space' lightpos to the fragment shader. lightPos is currently in world space.
...
void main()
{
    ...
    LightPos = vec3(view * vec4(lightPos, 1.0)); // Transform world-space light position to view-space light position
}
```

__Fragment shader:__
```GLSL
...
in vec3 LightPos;   // extra in variable, since we need the light position in view space we calculate this in the vertex shader
...
void main()
{
    ...    
    // specular
    float specularStrength = 0.5;
    vec3 viewDir = normalize(-FragPos); // the viewer is always at (0,0,0) in view-space, so viewDir is (0,0,0) - Position => -Position
    vec3 reflectDir = reflect(-lightDir, norm);  
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
    vec3 specular = specularStrength * spec * lightColor; 
    
    vec3 result = (ambient + diffuse + specular) * objectColor;
    FragColor = vec4(result, 1.0);
}
```

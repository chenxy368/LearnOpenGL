# Light Casters
## Spot Light
![spot_light.png](https://user-images.githubusercontent.com/98029669/213871050-3b87554c-65fc-46ea-a592-fa8c9bfd9598.jpg)  
A spotlight is a light source that is located somewhere in the environment that, instead of shooting light rays in all directions, only shoots them in a specific direction.
The result is that only the objects within a certain radius of the spotlight's direction are lit and everything else stays dark.  
LightDir: the vector pointing from the fragment to the light source.  
SpotDir: the direction the spotlight is aiming at.  
Phi ϕ: the cutoff angle that specifies the spotlight's radius. Everything outside this angle is not lit by the spotlight.  
Theta θ: the angle between the LightDir vector and the SpotDir vector. The θ value should be smaller than Φ to be inside the spotlight.  
## A Example: Flash Light
A flashlight is a spotlight located at the viewer's position and usually aimed straight ahead from the player's perspective. 
A flashlight is basically a normal spotlight, but with its position and direction continually updated based on the player's position and orientation.  
__Fragment Shader__
```GLSL
...
struct Light {
    vec3 position;  
    vec3 direction;
    float cutOff;
    float outerCutOff;
  
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
	
    float constant;
    float linear;
    float quadratic;
};
...
void main()
{
    vec3 lightDir = normalize(light.position - FragPos);
    
    // check if lighting is inside the spotlight cone
    float theta = dot(lightDir, normalize(-light.direction)); 
    
    if(theta > light.cutOff) // remember that we're working with angles as cosines instead of degrees so a '>' is used.
    {    
        // do lighting calculations
        // ambient
        vec3 ambient = light.ambient * texture(material.diffuse, TexCoords).rgb;
        
        // diffuse 
        vec3 norm = normalize(Normal);
        float diff = max(dot(norm, lightDir), 0.0);
        vec3 diffuse = light.diffuse * diff * texture(material.diffuse, TexCoords).rgb;  
        
        // specular
        vec3 viewDir = normalize(viewPos - FragPos);
        vec3 reflectDir = reflect(-lightDir, norm);  
        float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
        vec3 specular = light.specular * spec * texture(material.specular, TexCoords).rgb;  
        
        // attenuation
        float distance    = length(light.position - FragPos);
        float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));    

        // ambient  *= attenuation; // remove attenuation from ambient, as otherwise at large distances the light would be darker inside than outside the spotlight due the ambient term in the else branche
        diffuse   *= attenuation;
        specular *= attenuation;   
            
        vec3 result = ambient + diffuse + specular;
        FragColor = vec4(result, 1.0);
    }
    else 
    {
        // else, use ambient light so scene isn't completely dark outside the spotlight.
        FragColor = vec4(light.ambient * texture(material.diffuse, TexCoords).rgb, 1.0);
    }
} 
```
Remember angle values are represented as cosine values and an angle of 0 degrees is represented as the cosine value of 1.0 while an angle of 
90 degrees is represented as the cosine value of 0.0. Monotonically decreasing in this range.
Pass all uniform in program:
```C++
lightingShader.setVec3("light.position",  camera.Position);
lightingShader.setVec3("light.direction", camera.Front);
lightingShader.setFloat("light.cutOff",   glm::cos(glm::radians(12.5f)));
```

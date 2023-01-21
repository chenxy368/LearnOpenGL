# Light Casters
## Point lights
A point light is a light source with a given position somewhere in a world that illuminates in all directions, where the light rays fade out over distance.  
P.S. In previous section, we actually use the shaders for point lights but regard all lights as directional lights. The main difference is we do not consider attenuation. 
The correct directional light shader is in 5.1.    
![light_casters_point](https://user-images.githubusercontent.com/98029669/213848542-9fc62b78-5182-4172-99eb-7af0edf8a835.png)
## Add Attenuation
![light_attenuation_formula](https://user-images.githubusercontent.com/98029669/213848601-65924762-8511-4966-9e01-93f5e6b7b61e.png)  
Here d represents the distance from the fragment to the light source. 3 (configurable) terms: a constant term Kc, a linear term Kl and a quadratic term Kq.

The constant term is usually kept at 1.0 which is mainly there to make sure the denominator never gets smaller than 1 since it would otherwise boost the intensity with certain distances, which is not the effect we're looking for.  
The linear term is multiplied with the distance value that reduces the intensity in a linear fashion.  
The quadratic term is multiplied with the quadrant of the distance and sets a quadratic decrease of intensity for the light source. The quadratic term will be less significant compared to the linear term when the distance is small, but gets much larger as the distance grows.  
![attenuation](https://user-images.githubusercontent.com/98029669/213848634-55c6d78d-994c-42fd-9324-8aae7010ea16.png)  
P.S. Almost linear near 0. Quadratic term plays a more important as distance increasing.  
[Choose correct coefficient](https://wiki.ogre3d.org/tiki-index.php?page=-Point+Light+Attenuation)
__Fragment Shader__
```GLSL
#version 330 core
...
struct Light {
    vec3 position;  
  
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
	  
    // Add coefficient of attenuation formula
    float constant;
    float linear;
    float quadratic;
};
...
void main()
{
    ...
    // attenuation
    float distance    = length(light.position - FragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));    

    ambient  *= attenuation;  
    diffuse   *= attenuation;
    specular *= attenuation;   
        
    vec3 result = ambient + diffuse + specular;
    FragColor = vec4(result, 1.0);
} 
```
Pass all required coefficient in program
```C++
lightingShader.setFloat("light.constant", 1.0f);
lightingShader.setFloat("light.linear", 0.09f);
lightingShader.setFloat("light.quadratic", 0.032f);

//The white light cube is required.
...
```

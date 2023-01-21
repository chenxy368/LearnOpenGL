# Light Casters
## Edge Smooth for Spot Light
To create the effect of a smoothly-edged spotlight we want to simulate a spotlight having an inner and an outer cone. 
We can set the inner cone as the cone defined in the previous section, but we also want an outer cone that gradually dims the light from the inner to the edges of the outer cone.  
![spot_light_edge_smooth](https://user-images.githubusercontent.com/98029669/213871960-36ec7461-0eaa-43e9-9adc-1d16b37b758f.jpg)
```C++
// Pass cutOff and outerCutoff
lightingShader.setFloat("light.cutOff", glm::cos(glm::radians(12.5f)));
lightingShader.setFloat("light.outerCutOff", glm::cos(glm::radians(17.5f)));
```
```GLSL
// spotlight (soft edges), apply to diffuse and specular
float theta = dot(lightDir, normalize(-light.direction)); 
float epsilon = (light.cutOff - light.outerCutOff);
float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);
diffuse  *= intensity;
specular *= intensity;
```

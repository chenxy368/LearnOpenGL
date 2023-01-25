# Parallax Mapping
## Steep Parallax Mapping
__The offset is a crude approximation to get to point B. When heights change rapidly over a surface the results tend to look unrealistic as the vector P
 will not end up close to B as you can see below:__  
![image](https://user-images.githubusercontent.com/98029669/214443812-6ad77e08-5c1b-4e36-9249-c2457cd51042.png)

To solve this, use a series level and find the nearest point for offset.  
![image](https://user-images.githubusercontent.com/98029669/214445395-f4f69feb-a266-4df8-aef3-3372e5faf4ec.png)  
```GLSL
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    // number of depth layers
    const float numLayers = 10;
    // calculate the size of each layer
    float layerDepth = 1.0 / numLayers;
    // depth of current layer
    float currentLayerDepth = 0.0;
    // the amount to shift the texture coordinates per layer (from vector P)
    // height_scale is a uniform given by C++ side, adjust by keyboard
    // np / viewDir.z here
    vec2 P = viewDir.xy * height_scale; 
    float deltaTexCoords = P / numLayers;

    // get initial values
    vec2  currentTexCoords     = texCoords;
    float currentDepthMapValue = texture(depthMap, currentTexCoords).r;

    while(currentLayerDepth < currentDepthMapValue)
    {
        // shift texture coordinates along direction of P
        currentTexCoords -= deltaTexCoords;
        // get depthmap value at current texture coordinates
        currentDepthMapValue = texture(depthMap, currentTexCoords).r;  
        // get depth of next layer
        currentLayerDepth += layerDepth;  
    }

    return texCoords - currentTexCoords;   
}
```
We don't divide z cooradinate to adjust offsets here. Instead, we can change the number of layers. 
Taking less samples when looking straight at a surface and more samples when looking at an angle we only sample the necessary amount.  
![image](https://user-images.githubusercontent.com/98029669/214448033-4b61aba1-a58e-478e-8cc6-1d1b5b744e4b.png)  

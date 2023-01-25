# Parallax Mapping
## Parallax Occlusion Mapping
![image](https://user-images.githubusercontent.com/98029669/214448247-19684245-21cb-434a-895e-1e401f46cb8f.png)  
Steep Parallax Mapping also comes with its problems though. Because the technique is based on a finite number of samples, we get aliasing effects and the clear distinctions between layers can easily be spotted:

Add a linear interpolation(weighted average T2 and T3)  
![image](https://user-images.githubusercontent.com/98029669/214448373-555d3f99-83f3-478f-9bbe-c5e5f2a52f9c.png)
```GLSL
[...] // steep parallax mapping code here

// get texture coordinates before collision (reverse operations)
vec2 prevTexCoords = currentTexCoords + deltaTexCoords;

// get depth after and before collision for linear interpolation
float afterDepth  = currentDepthMapValue - currentLayerDepth;
float beforeDepth = texture(depthMap, prevTexCoords).r - currentLayerDepth + layerDepth;

// interpolation of texture coordinates
float weight = afterDepth / (afterDepth - beforeDepth);
vec2 finalTexCoords = prevTexCoords * weight + currentTexCoords * (1.0 - weight);

return finalTexCoords;
```

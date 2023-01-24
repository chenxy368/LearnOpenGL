# Shadow Mapping
## Improving Shadow Maps

### Shadow acne 
Shows a very obvious Moiré-like pattern:  
![image](https://user-images.githubusercontent.com/98029669/214163662-5755273e-f755-4d5a-93fe-0bfc76f4d2f5.png)  
![image](https://user-images.githubusercontent.com/98029669/214173822-82f4e255-329f-4d25-9784-b7624bfde712.png)


We can solve this issue with a small little hack called a shadow bias where we simply offset the depth of the surface 
(or the shadow map) by a small bias amount such that the fragmentsare not incorrectly considered above the surface.  
![image](https://user-images.githubusercontent.com/98029669/214163816-17950c5f-fd7a-43aa-9177-c3910d65d5a2.png)  
```GLSL
float bias = 0.005;
float shadow = currentDepth - bias > closestDepth  ? 1.0 : 0.0;
```
To further prevent fail at some highly dependent planes, a solid approach would be to change the amount of bias based on the surface angle towards the light
```GLSL
float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005); 
```

### Peter panning
A __disadvantage of using a shadow bias__ is that you're applying an offset to the actual depth of objects. 
As a result, the bias may become large enough to see a visible offset of shadows compared to the actual object locations as you can see below 
(with an exaggerated bias value). In theoretic, if you want to exactly right and do not go over, you need to know more information about sampling.
However, we are kind off abrapt to choos the offset bias:    
![image](https://user-images.githubusercontent.com/98029669/214165910-9078b224-fd8a-4ba8-8b2a-e56635e7fe1d.png)  
![image](https://user-images.githubusercontent.com/98029669/214170511-bd184973-087d-4972-b2c6-681d2a1f2c4c.png)  
![image](https://user-images.githubusercontent.com/98029669/214183935-8b9a9929-1557-4d90-aa6b-999389ac85e8.png)
![image](https://user-images.githubusercontent.com/98029669/214183969-5050b96c-674a-4c2d-9645-036396ac4b8e.png)
![image](https://user-images.githubusercontent.com/98029669/214184017-6c0abc97-e155-42dd-8b29-500331c55bc1.png)
![image](https://user-images.githubusercontent.com/98029669/214184046-9bde9ef9-9126-44f8-994e-b8053bc17dc4.png)

Add a front cutoff when generating shadow maps
```C++
glCullFace(GL_FRONT);
RenderSceneToDepthMap();
glCullFace(GL_BACK); // remember later render front
```
### Over sampling
Another visual discrepancy which you may like or dislike is that regions outside the light's visible frustum are considered to be in shadow while they're (usually) not. This happens because projected coordinates outside the light's frustum are higher than 1.0 and will thus sample the depth texture outside its default range of [0,1]. (Been clipped when try to generate depth map)   
![image](https://user-images.githubusercontent.com/98029669/214185540-2d5e1220-3fcd-4e6a-84c1-20becfbf783d.png)  
We can do this by configuring a texture border color and set the depth map's texture wrap options to GL_CLAMP_TO_BORDER(set outside 1)
```C++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
float borderColor[] = { 1.0f, 1.0f, 1.0f, 1.0f };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);  
```
This is succeed except fail to concern where depth > 1 in light space, which also has been cutoff.
```GLSL
float ShadowCalculation(vec4 fragPosLightSpace)
{
    [...]
    if(projCoords.z > 1.0)
        shadow = 0.0;
    
    return shadow;
}  
```
### PCF
Just aliasing, can average to remove.  
![image](https://user-images.githubusercontent.com/98029669/214185835-2a8d87ee-b7c5-444f-b7d9-a00eefc2e8b3.png)  
On example is sample around and average the depth rather than only check one sample.
```GLSL
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for(int x = -1; x <= 1; ++x)
{
    for(int y = -1; y <= 1; ++y)
    {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r; 
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;        
    }    
}
shadow /= 9.0;
```
Here textureSize returns a vec2 of the width and height of the given sampler texture at mipmap level 0.




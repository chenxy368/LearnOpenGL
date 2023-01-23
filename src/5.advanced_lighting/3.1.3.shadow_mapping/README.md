# Shadow Mapping
## Improving Shadow Maps

### Shadow acne 
Shows a very obvious Moiré-like pattern:  
![image](https://user-images.githubusercontent.com/98029669/214163662-5755273e-f755-4d5a-93fe-0bfc76f4d2f5.png)  
![image](https://user-images.githubusercontent.com/98029669/214163721-e07a7727-54b5-4a2d-ab65-6c25e384a06c.png)

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


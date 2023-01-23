# Advanced Lighting
## Phong lighting's specular reflections break down in certain conditions, specifically when the shininess property is low resulting in a large (rough) specular area.
![image](https://user-images.githubusercontent.com/98029669/214064035-b2c11d8f-9d1f-47e3-8366-ac1238915521.png)  
In Fragment Shader:
```GLSL
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
```
We filter the negative dot product. However, in practice, it's possible to see the light while the dot product of these two vectors is negative. E.g. the right image:  
![image](https://user-images.githubusercontent.com/98029669/214064587-a901aaf9-d6c6-4fb0-a473-b00653384314.png)
## Blinn Phong Model
Instead of relying on a reflection vector we're using a so called __halfway vector__ that is a unit vector __exactly halfway between the view direction and the light direction__. 
The __closer__ this halfway vector aligns with the surface's normal vector, the __higher__ the specular contribution.  
![image](https://user-images.githubusercontent.com/98029669/214065531-38e3892a-0c8f-4609-826a-240840aa1aee.png)  
Get halfway vector(Remember to normalize):  
```GLSL
vec3 lightDir   = normalize(lightPos - FragPos);
vec3 viewDir    = normalize(viewPos - FragPos);
vec3 halfwayDir = normalize(lightDir + viewDir);
```
Compute specular
```GLSL
float spec = pow(max(dot(normal, halfwayDir), 0.0), shininess);
vec3 specular = lightColor * spec;
```
P.S.
A subtle difference between Phong and Blinn-Phong shading is that the angle between the halfway vector and the surface normal is __often shorter__ than the angle between the view and reflection vector.
As a result, to get visuals similar to Phong shading the __specular shininess exponent has to be set a bit higher__. 
A general rule of thumb is to set it between __2 and 4 times__ the Phong shininess exponent.

Switch between phong and blinn phong
```GLSL
void main()
{
    [...]
    float spec = 0.0;
    if(blinn)
    {
        vec3 halfwayDir = normalize(lightDir + viewDir);  
        spec = pow(max(dot(normal, halfwayDir), 0.0), 16.0);
    }
    else
    {
        vec3 reflectDir = reflect(-lightDir, normal);
        spec = pow(max(dot(viewDir, reflectDir), 0.0), 8.0);
    }
}
```
```C++
void processInput(GLFWwindow *window)
{
    ...

    if (glfwGetKey(window, GLFW_KEY_B) == GLFW_PRESS && !blinnKeyPressed) 
    {
        blinn = !blinn;
        blinnKeyPressed = true;
    }
    if (glfwGetKey(window, GLFW_KEY_B) == GLFW_RELEASE) 
    {
        blinnKeyPressed = false;
    }
}
```

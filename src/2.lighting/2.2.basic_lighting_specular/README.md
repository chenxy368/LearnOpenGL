# Basic Color
## Normal Vector with Translation and Scaling
Change in vertex shader from
```GLSL
Normal = aNormal;  
```
To
```GLSL
Normal = mat3(transpose(inverse(model))) * aNormal; 
```
First of all, normal vectors are only direction vectors and do not represent a specific position in space.
Second, normal vectors do not have a homogeneous coordinate (the w component of a vertex position). 
This means that translations should not have any effect on the normal vectors. 
So if we want to multiply the normal vectors with a model matrix we want to remove the translation part of the matrix by taking the 
upper-left 3x3 matrix of the model matrix (note that we could also set the w component of a normal vector to 0 and multiply with the 4x4 matrix).  
Second, if the model matrix would perform a non-uniform scale, the vertices would be changed in such a way that the normal vector is not perpendicular
to the surface anymore. The following image shows the effect such a model matrix (with non-uniform scaling) has on a normal vector:  
![basic_lighting_normal_transformation](https://user-images.githubusercontent.com/98029669/213773620-573efd1f-5c4f-4754-87dc-b127b40642e3.png)  
Obviously, you only need to rotate the vector after scaling.  
__inverse translate to solve this problem__   
```GLSL
Normal = mat3(transpose(inverse(model))) * aNormal;
```
Remember first change to mat3 to truncate translation part(last row and column).
## Specular Highlight
![specular](https://user-images.githubusercontent.com/98029669/213830798-d1807d26-3229-4349-8c13-9b09cceee8e3.png)
1. Compute view vector  
2. Compute reflect vector  
3. Compute specular intensity
4. Compute color, remember multiply a strength coefficient
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 Normal;  
in vec3 FragPos;  
  
uniform vec3 lightPos; 
uniform vec3 viewPos; 
uniform vec3 lightColor;
uniform vec3 objectColor;

void main()
{
    // ambient
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;
  	
    // diffuse 
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;
    
    // specular
    float specularStrength = 0.5;
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);  
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
    vec3 specular = specularStrength * spec * lightColor;  
        
    vec3 result = (ambient + diffuse + specular) * objectColor;
    FragColor = vec4(result, 1.0);
} 
```

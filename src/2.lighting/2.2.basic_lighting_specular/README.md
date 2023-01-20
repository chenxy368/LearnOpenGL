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
![inverse_trans](https://user-images.githubusercontent.com/98029669/213807198-a970683b-a466-409c-be51-555d528d07d7.png)
```GLSL
Normal = mat3(transpose(inverse(model))) * aNormal;
```
Remember first change to mat3 to truncate translation part(last row and column).
## Specular Highlight

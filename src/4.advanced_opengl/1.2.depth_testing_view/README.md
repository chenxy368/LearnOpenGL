# Depth Testing
## Visualizing the Depth Buffer
Directly use gl_FragCoord:
```GLSL
void main()
{             
    FragColor = vec4(vec3(gl_FragCoord.z), 1.0);
}  
``` 
![depth_testing_visible_depth](https://user-images.githubusercontent.com/98029669/213887253-3aab943e-072d-4daf-82ae-67c6576c07da.png)  
Almost white because non-linear push to far(1).  
To go back to linear, first back to NDC(depth values from the range [0,1] to normalized device coordinates in the range [-1,1]).
```GLSL
float ndc = depth * 2.0 - 1.0; 
```
Inverse non-linear
```GLSL
float linearDepth = (2.0 * near * far) / (far + near - ndc * (far - near));	
```
__Fragment Shader__
```GLSL
#version 330 core
out vec4 FragColor;

float near = 0.1; 
float far  = 100.0; 
  
float LinearizeDepth(float depth) 
{
    float z = depth * 2.0 - 1.0; // back to NDC 
    return (2.0 * near * far) / (far + near - z * (far - near));	
}

void main()
{             
    float depth = LinearizeDepth(gl_FragCoord.z) / far; // divide by far for demonstration
    FragColor = vec4(vec3(depth), 1.0);
}
```
## Z-fighting
A common visual artifact may occur when two planes or triangles are so closely aligned to each other that the depth buffer does not have enough precision to figure out which one of the two shapes is in front of the other. 
The result is that the two shapes continually seem to switch order which causes weird glitchy patterns. 
__Prevent Z-fighting:__  
1. Never place objects too close to each other in a way that some of their triangles closely overlap.  
2. Set the near plane as far as possible. Near plane has highe depth precision. Better to let more element close to the near plane.  
3. Use a higher precision depth buffer: 24bit->36bit.

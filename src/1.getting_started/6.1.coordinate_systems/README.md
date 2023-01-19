# Coordinate System
## Overview
![coordinate_systems](https://user-images.githubusercontent.com/98029669/213333975-26dff00a-e56f-4e70-b8c8-45e7ffe9d21b.png)
## Local Space
The origin of object is probably at (0,0,0) as imported. 
## World Space
All objects are probably located at (0, 0, 0) of world space. With a world space coordinate, move them to differect positions.
## View space
The view space is what people usually refer to as the camera of OpenGL (it is sometimes also known as camera space or eye space). 
The view space is the result of transforming your world-space coordinates to coordinates that are in front of the user's view. 
The view space is thus the space as seen from the camera's point of view. (More detail in 7)
## Clip space
At the end of each vertex shader run, OpenGL expects the coordinates to be within a specific range and any coordinate that falls outside this range is clipped. 
Coordinates that are clipped are discarded, so the remaining coordinates will end up as fragments visible on your screen. 
This is also where clip space gets its name from.  
This viewing box a projection matrix creates is called a frustum and each coordinate that ends up inside this frustum will end up on the user's screen.  
Once all the vertices are transformed to clip space a final operation called perspective division is performed where we divide the x, y and z components of the 
position vectors by the vector's homogeneous w component; perspective division is what transforms the 4D clip space coordinates to 3D normalized device coordinates. 
This step is performed automatically at the end of the vertex shader step.  
__Orthographic projection:__  
![orthoproject](https://user-images.githubusercontent.com/98029669/213354356-8ddecc2a-a3c4-4156-b17c-02a2d3e7e9c9.jpg)
```C++
glm::ortho(0.0f, 800.0f, 0.0f, 600.0f, 0.1f, 100.0f);
```
The first two parameters specify the left and right coordinate of the frustum and the third and fourth parameter specify the bottom and top part of the frustum. 
With those 4 points we've defined the size of the near and far planes and the 5th and 6th parameter then define the distances between the near and far plane.  
__Perspective projection:__  
![perspective](https://user-images.githubusercontent.com/98029669/213354750-d924cd19-9e72-4b50-a08d-19bb75f5f334.jpg)  
The projection matrix maps a given frustum range to clip space, but also manipulates the w value of each vertex coordinate in such a way that the further away 
a vertex coordinate is from the viewer, the higher this w component becomes. Once the coordinates are transformed to clip space they are in the range -w to w 
(anything outside this range is clipped).
```C++
glm::mat4 proj = glm::perspective(glm::radians(45.0f), (float)width/(float)height, 0.1f, 100.0f);
```
Its first parameter defines the fov value, that stands for field of view and sets how large the viewspace is. 
For a realistic view it is usually set to 45 degrees, but for more doom-style results you could set it to a higher value. 
The second parameter sets the aspect ratio which is calculated by dividing the viewport's width by its height. 
The third and fourth parameter set the near and far plane of the frustum.  
![perspective1](https://user-images.githubusercontent.com/98029669/213559965-626958bd-894d-48b3-ab97-2100e12b087d.jpg)  
![perspective2](https://user-images.githubusercontent.com/98029669/213559978-f7a0022d-6e00-4f74-95ad-33ac4b2f01b1.jpg)
## Combination
![viewcombine](https://user-images.githubusercontent.com/98029669/213560168-6a3b7d88-a8e1-4e73-8dd4-baf2b8251348.png)  
Add to shader
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
...
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
    ...
}
```
```C++
glm::mat4 model;
model = glm::rotate(model, glm::radians(-55.0f), glm::vec3(1.0f, 0.0f, 0.0f));
glm::mat4 view;
view = glm::translate(view, glm::vec3(0.0f, 0.0f, -3.0f));
glm::mat4 projection;
projection = glm::perspective(glm::radians(45.0f), screenWidth / screenHeight, 0.1f, 100.0f);
...
int modelLoc = glGetUniformLocation(ourShader.ID, "model"));
glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
... // same for projection and translate
```

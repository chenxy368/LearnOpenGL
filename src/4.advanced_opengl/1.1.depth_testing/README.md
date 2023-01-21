# Depth Testing
## Introduction
In the coordinate systems chapter we've rendered a 3D container and made use of a depth buffer to prevent triangles rendering in the front while they're supposed to 
be behind other triangles. The depth-buffer is a buffer that, just like the color buffer (that stores all the fragment colors: the visual output), __stores information per 
fragment and has the same width and height as the color buffer__. When depth testing is enabled, OpenGL tests the depth value of a fragment against the content of the depth buffer. 
OpenGL performs a depth test and __if this test passes, the fragment is rendered and the depth buffer is updated with the new depth value. 
If the depth test fails, the fragment is discarded.__  


Depth testing is done in screen space after the fragment shader has run (and after the stencil test).
The screen space coordinates relate directly to the viewport defined by OpenGL's glViewport function and can be accessed via GLSL's built-in gl_FragCoord variable in the fragment shader. 
The x and y components of gl_FragCoord represent the fragment's screen-space coordinates (with (0,0) being the bottom-left corner). 
__The gl_FragCoord variable also contains a z-component which contains the depth value of the fragment.
This z value is the value that is compared to the depth buffer's content.__

Today most GPUs support a hardware feature called early depth testing. Early depth testing allows the depth test to run before the fragment shader runs. 
Whenever it is clear a fragment isn't going to be visible (it is behind other objects) we can prematurely discard the fragment.
__A restriction on the fragment shader for early depth testing is that you shouldn't write to the fragment's depth value.__  

To enable:
```C++
glEnable(GL_DEPTH_TEST);  
```
To clear
```C++
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);  
```
There are certain scenarios imaginable where you want to perform the depth test on all fragments and discard them accordingly, 
__but not update the depth buffer__. Basically, you're (temporarily) using a read-only depth buffer. 
```C++
glDepthMask(GL_FALSE);
```

## Depth Flag
```C++
// configure global opengl state
glEnable(GL_DEPTH_TEST);
glDepthFunc(GL_ALWAYS); // always pass the depth test (same effect as glDisable(GL_DEPTH_TEST))
```
![depth_flag](https://user-images.githubusercontent.com/98029669/213886389-213f8ffe-3116-4a9f-9cfa-99f4101f41ef.png)  
E.g. GL_ALWAYS  
![depth_testing_func_always](https://user-images.githubusercontent.com/98029669/213886414-36467950-40ff-4bab-91e3-c5ad00bbe819.png)  
GL_LESS  
![depth_testing_func_less](https://user-images.githubusercontent.com/98029669/213886420-5c27a77d-16bb-41ea-a074-cc6c30cd34ee.png)

## Depth Value Precision
The depth buffer contains depth values between 0.0 and 1.0 and it compares its content with the z-values of all the objects in the scene as seen from the viewer. 
These z-values in view space can be any value between the projection-frustum's near and far plane. 
We thus need some way to transform these view-space z-values to the range of [0,1].  
__Notice: Using a linear transform is a bad idea because the near and far have same precision. In practice, we more care about near things rather than far things.__  
Formula for transforming z to depth value:  
![depth_value](https://user-images.githubusercontent.com/98029669/213886840-f49a8d38-70ea-4c7a-9960-80c22bdcb587.png)  
__In the perspective projection matrix, we actually achieve this transformation.__  
![image](https://user-images.githubusercontent.com/98029669/213887037-640f7c37-71f1-4710-83c9-675fc9251c2c.png)

# Stencil Testing
## Introduction
Just like the depth test, stencil testing has the option to discard fragments. 
The stencil test is based on the content of yet another buffer called the stencil buffer that we're allowed to update during rendering.  
![stencil_buffer](https://user-images.githubusercontent.com/98029669/213887646-d73500c9-436c-4b2f-a2bb-668b451c6464.png)  
The fragments of the scene are then only rendered (the others are discarded) wherever the stencil value of that fragment contains a 1.  
__General outline:__  
1. Enable writing to the stencil buffer.  
2. Render objects, updating the content of the stencil buffer.  
3. Disable writing to the stencil buffer.  
4. Render (other) objects, this time discarding certain fragments based on the content of the stencil buffer.  
To enable
```C++
glEnable(GL_STENCIL_TEST);
```
To clear
```C++
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
```
The function glStencilMask allows us to set a bitmask that is ANDed with the stencil value about to be written to the buffer. 
By default this is set to a bitmask of all 1s not affecting the output, but if we were to set this to 0x00 all the stencil values written to the buffer end up as 0s.
```C++
glStencilMask(0xFF); // each bit is written to the stencil buffer as is
glStencilMask(0x00); // each bit ends up as 0 in the stencil buffer (disabling writes)
```
## glStencilFunc(GLenum func, GLint ref, GLuint mask)
func: sets the stencil test function that determines whether a fragment passes or is discarded. 
This test function is applied to the __stored stencil value and the glStencilFunc's ref value__. 
Possible options are: GL_NEVER, GL_LESS, GL_LEQUAL, GL_GREATER, GL_GEQUAL, GL_EQUAL, GL_NOTEQUAL and GL_ALWAYS. 
The semantic meaning of these __is similar to the depth buffer's functions__.
ref: specifies the __reference value__ for the stencil test. The stencil buffer's content is compared to this value.
mask: specifies __a mask that is ANDed with both the reference value and the stored stencil value before the test compares them__. Initially set to all 1s.  
E.g. for this case:  
![stencil_buffer](https://user-images.githubusercontent.com/98029669/213887646-d73500c9-436c-4b2f-a2bb-668b451c6464.png)  
```C++
glStencilFunc(GL_EQUAL, 1, 0xFF)
```
We need keep 1s. This tells OpenGL that whenever the stencil value of a fragment is equal (GL_EQUAL) to the reference value 1, 
the fragment passes the test and is drawn, otherwise discarded.  
## glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)
1. sfail: action to take if the stencil test __fails__.  
2. dpfail: action to take if the __stencil test passes__, but the __depth test fails__.  
3. dppass: action to take if __both the stencil and the depth test pass__.  
![glStencilOp ](https://user-images.githubusercontent.com/98029669/213888103-c0347ccb-610b-4d8d-bdc0-b68489b6e886.png)
## Exmaple: Object Outlining
The routine for outlining your objects is as follows:
1. Enable stencil writing.  
2. Set the stencil op to GL_ALWAYS before drawing the (to be outlined) objects, updating the stencil buffer with 1s wherever the objects' fragments are rendered.
3. Render the objects.  
4. Disable stencil writing and depth testing.  
5. Scale each of the objects by a small amount.  
6. Use a different fragment shader that outputs a single (border) color.  
7. Draw the objects again, but only if their fragments' stencil values are not equal to 1.  
8. Enable depth testing again and restore stencil func to GL_KEEP.  


1. Create a very basic fragment shader that outputs a border color
```GLSL
// shaderSingleColor
void main()
{
    FragColor = vec4(0.04, 0.28, 0.26, 1.0);
}
```
 We want to first draw the floor, then the two containers (while writing to the stencil buffer), and then draw the scaled-up containers 
 (while discarding the fragments that write over the previously drawn container fragments).
2. Enable stencil testing  
```C++
glEnable(GL_STENCIL_TEST);
```
3. Specify the action to take whenever any of the stencil tests succeed or fail
```C++
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);
```
If any of the tests fail we do nothing; we simply keep the currently stored value that is in the stencil buffer (which will not be drawn).
If both the stencil test and the depth test succeed however, we want to replace the stored stencil value with the reference value set via glStencilFunc
which we later __set to 1__.
4. Clear the stencil buffer to 0s at the start of the frame and for the containers we update the stencil buffer to 1 for each fragment drawn
```C++
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  
glStencilFunc(GL_ALWAYS, 1, 0xFF); // all fragments should pass the stencil test
glStencilMask(0xFF); // enable writing to the stencil buffer
normalShader.use();
DrawTwoContainers();
```
5.  We're going to draw the upscaled containers, but this time with __the appropriate test function and disabling writes to the stencil buffer__
```C++
glStencilFunc(GL_NOTEQUAL, 1, 0xFF);
glStencilMask(0x00); // disable writing to the stencil buffer
glDisable(GL_DEPTH_TEST);
shaderSingleColor.use(); 
DrawTwoScaledUpContainers();
```
We set the stencil function to GL_NOTEQUAL to make sure that we're __only drawing parts of the containers that are not equal to 1__. 
This way we only draw the part of the containers that are __outside the previously drawn containers__. 
Note that we also __disable depth testing so the scaled up containers (e.g. the borders) do not get overwritten by the floor.__  
![stencil_scene_outlined](https://user-images.githubusercontent.com/98029669/213888352-77797693-a214-4c8a-be6a-0f71e9417978.png)

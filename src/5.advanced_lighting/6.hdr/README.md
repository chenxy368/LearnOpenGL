# HDR
Brightness and color values, by default, are clamped between 0.0 and 1.0 when stored into a framebuffer. 

What happens if we walk in a really bright area with multiple bright light sources that as a total sum exceed 1.0? 
The answer is that all fragments that have a brightness or color sum over 1.0 get clamped to 1.0, which isn't pretty to look at
(losing a significant amount of detail and giving it a fake look):  
![image](https://user-images.githubusercontent.com/98029669/214453670-912cb01b-5ed3-48a2-97eb-5a01ee605548.png)

Monitors (non-HDR) are limited to display colors in the range of 0.0 and 1.0, but there is no such limitation in lighting equations. 
__By allowing fragment colors to exceed 1.0__ we have a much higher range of color values available to work in known as high dynamic range (HDR). 

High dynamic range rendering allow a much larger range of color values to render to, and at the end we transform all the HDR values __back to the low dynamic range (LDR) of [0.0, 1.0]__. 
This process of converting HDR values to LDR values is called __tone mapping__.
These tone mapping algorithms often involve an __exposure parameter__ that selectively favors dark or bright regions.  
Different exposure level:  
![image](https://user-images.githubusercontent.com/98029669/214454090-2cebf223-4928-44a4-9547-37176e508e12.png)

HDR gives us the ability to specify a light source's intensity by their real intensities. 
For instance, the sun has a much higher intensity than something like a flashlight so why not configure the sun as such __(e.g. a diffuse brightness of 100.0)__. 
This allows us to more properly configure a scene's lighting with more realistic lighting parameters.
 
## Floating Point Framebuffers
When framebuffers use a normalized fixed-point color format (like GL_RGB) as their color buffer's internal format, OpenGL automatically clamps the values between 0.0 and 1.0 before storing them in the framebuffer. 
 
To get rid of the limitation, use GL_RGB16F, GL_RGBA16F, GL_RGB32F, or GL_RGBA32F. The framebuffer is known as a floating point framebuffer.
```C++
glBindTexture(GL_TEXTURE_2D, colorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);  
```

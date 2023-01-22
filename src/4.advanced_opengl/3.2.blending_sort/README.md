# Blending
## Blending
To render images with different levels of transparency we have to enable blending.
```C++
glEnable(GL_BLEND);  
```
__Blending Interpolation__  
![blending_equation](https://user-images.githubusercontent.com/98029669/213897081-5fe8f61d-e0bd-43cc-9a50-cfa7b3697cbd.png)  
After the fragment shader has run and all the tests have passed, this blend equation is let loose on the fragment's color
output and with whatever is currently in the color buffer.
![image](https://user-images.githubusercontent.com/98029669/213897524-a9f60a50-910d-4c2e-a15e-8e445f377d05.png)
__glBlendFunc(GLenum sfactor, GLenum dfactor)__  
![glblendfunc](https://user-images.githubusercontent.com/98029669/213897547-94867f13-362c-4ed2-829c-fec755739146.png)  
We want to take the alpha of the source color vector for the source factor and 1−alpha of the same color vector for the destination factor.
```C++
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```
It is also possible to set different options for the RGB and alpha channel individually using glBlendFuncSeparate:
```C++
glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA, GL_ONE, GL_ZERO);
```
Right now, the source and destination components are added together, but we could also subtract them if we want. 
glBlendEquation(GLenum mode) allows us to set this operation and has 5 possible options:
![glblendequation](https://user-images.githubusercontent.com/98029669/213897764-b3ad8e27-fbf0-4a6e-9c7b-28e94a7a768b.png)
## Premultiplied alpha
![1674354340770](https://user-images.githubusercontent.com/98029669/213897801-c7a9ae2d-8b9a-4233-b0b7-6d5647df676d.png)
![1674354367006](https://user-images.githubusercontent.com/98029669/213897811-f7fc6540-66be-44ac-b8a8-9130543f0967.png)
![1674354413850](https://user-images.githubusercontent.com/98029669/213897830-0a833d9d-7c08-40a9-bd45-00d97e5590cf.png)
![1674354435216](https://user-images.githubusercontent.com/98029669/213897840-9202043c-6295-4fae-95cf-54ddafaea7a1.png)
![1674354448693](https://user-images.githubusercontent.com/98029669/213897846-58a85aa7-776f-4b47-a455-cd5e383a0097.png)
![1674354461521](https://user-images.githubusercontent.com/98029669/213897853-468426c0-084b-4d26-8fd8-7a937bfce25b.png)  
Think about conditional probability, the two part are not two independent event.  
## Rendering Semi-transparent Textures
1. Enable blending and set the appropriate blending function:
```C++
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA); 
```
2. No change in fragment shader:
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D texture1;

void main()
{             
    FragColor = texture(texture1, TexCoords);
}  
```
3. Deal with the order:  
![image](https://user-images.githubusercontent.com/98029669/213898033-66375bcf-11c8-4f27-a2db-c7520009353a.png)  
If you take a closer look however, you may notice something is off. 
The transparent parts of the front window are occluding the windows in the background. 
When writing to the depth buffer, the depth test does not care if the fragment has transparency or not, so the transparent parts are written to the depth buffer as any other value. 
The result is that the background windows are tested on depth as any other opaque object would be, ignoring transparency. 
Even though the transparent part should show the windows behind it, the depth test discards them.
__When drawing a scene with non-transparent and transparent objects the general outline is usually as follows:__
1. Draw all opaque objects first.
2. Sort all the transparent objects.
3. Draw all the transparent objects in sorted order.
A map automatically sorts its values based on its keys:
```C++
std::map<float, glm::vec3> sorted;
for (unsigned int i = 0; i < windows.size(); i++)
{
    float distance = glm::length(camera.Position - windows[i]);
    sorted[distance] = windows[i];
}
```
Reverse order, from far to near
```C++
for(std::map<float,glm::vec3>::reverse_iterator it = sorted.rbegin(); it != sorted.rend(); ++it) 
{
    model = glm::mat4();
    model = glm::translate(model, it->second);              
    shader.setMat4("model", model);
    glDrawArrays(GL_TRIANGLES, 0, 6);
}
```
However, blending can be a more complicated problem. There are many other method including order independent transparency to solve blending problem under more complicated situation.

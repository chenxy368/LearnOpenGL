# Cubemaps
## Environment Mapping
Using a cubemap with an environment, we could give objects reflective or refractive properties. 
## Reflection
![image](https://user-images.githubusercontent.com/98029669/213928255-f4f65c5d-61cd-468a-bca4-229361bb96c7.png)  
We need normal now
```C++
float cubeVertices[] = {
    // positions          // normals
   -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
   -0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
   -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
   ...
};
```
__In fragment shader:__
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 Normal;
in vec3 Position;

uniform vec3 cameraPos;
uniform samplerCube skybox;

void main()
{             
    vec3 I = normalize(Position - cameraPos);
    vec3 R = reflect(I, normalize(Normal)); // find sample position
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}
```
We need normal and fragment position now, so let's get tem __in vertex shader__.
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;

out vec3 Normal;
out vec3 Position;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    Normal = mat3(transpose(inverse(model))) * aNormal; // Remember keep normal correct with scaling
    Position = vec3(model * vec4(aPos, 1.0));
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}
```
The skybox texture is now needed in scene(a cube) rendering. Bind it.
```C++
glBindVertexArray(cubeVAO);
glBindTexture(GL_TEXTURE_CUBE_MAP, skyboxTexture);          
glDrawArrays(GL_TRIANGLES, 0, 36);
```
## Refraction
Same as the reflection, try to find the sample position. We need to use snell's rule to find it.  
![image](https://user-images.githubusercontent.com/98029669/213929398-5aed0e28-5e0d-4625-95f5-ceb735722e00.png)  
In our case, We use these refractive indices to calculate the ratio between both materials the light passes through. 
The light/view ray goes from air to glass (if we assume the object is made of glass) so the ratio becomes 1.00/1.52=0.658.  
__Fragment Shader(not hard to mix with reflection)__
```GLSL
void main()
{             
    float ratio = 1.00 / 1.52;
    vec3 I = normalize(Position - cameraPos);
    vec3 R = refract(I, normalize(Normal), ratio);
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}
```

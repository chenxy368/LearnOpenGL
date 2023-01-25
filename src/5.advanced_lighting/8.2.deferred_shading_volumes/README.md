# Deffered Shading
## Light Volumes
Normally when we render a fragment in a large lit scene we'd calculate the contribution of each light source in a scene, regardless of their distance to the fragment. 
The idea behind light volumes is to calculate the radius, or volume, of a light source i.e. the area where its light is able to reach fragments.(Attenuation)
### Calculating a light's volume or radius
Attenuation formula:  
![1674626086981(1)](https://user-images.githubusercontent.com/98029669/214490864-e9c8e66d-a3ee-4f95-8b7d-49fdceacc205.png)  
What we want to do is solve this equation for when Flight. However, this equation will never exactly reach the value 0.0 is 0.0. 
So we discard it when it is dark enough. E.g. > 5/256  
![1674626234461(1)](https://user-images.githubusercontent.com/98029669/214491223-72c860d1-3f57-459a-bdfa-399638d5c167.png)  
Solve(Here Imax is the light source's brightest color component.)  
![1674626285345](https://user-images.githubusercontent.com/98029669/214491381-85f04dbe-cb26-45cf-b68e-fc74a52bed20.png)  
Get d(radius):  
![1674626325360](https://user-images.githubusercontent.com/98029669/214491457-8bd4312d-2ee6-4e8b-8888-5f557553e884.png)
Naive implementation:  
```C++
// Compute in program
float constant  = 1.0; 
float linear    = 0.7;
float quadratic = 1.8;
float lightMax  = std::fmaxf(std::fmaxf(lightColor.r, lightColor.g), lightColor.b);
float radius    = 
  (-linear +  std::sqrtf(linear * linear - 4 * quadratic * (constant - (256.0 / 5.0) * lightMax))) 
  / (2 * quadratic);  
 ```
 ```GLSL
 struct Light {
    [...]
    float Radius;
}; 
  
void main()
{
    [...]
    for(int i = 0; i < NR_LIGHTS; ++i)
    {
        // calculate distance between light source and current fragment
        float distance = length(lights[i].Position - FragPos);
        if(distance < lights[i].Radius)
        {
            // do expensive lighting
            [...]
        }
    }   
}
```
__!!!However The reality is that your GPU and GLSL are pretty bad at optimizing loops and branches. 
The reason for this is that shader execution on the GPU is highly parallel and most architectures 
have a requirement that for large collection of threads they need to run the exact same shader code for it to be efficient.(Just cannot loop)__

The appropriate approach to using light volumes is to render actual spheres, scaled by the light volume radius. 
As a rendered sphere produces fragment shader invocations that exactly match the pixels the light source affects, we only render the relevant pixels and skip all other pixels. 
![image](https://user-images.githubusercontent.com/98029669/214492264-7146cdd7-d66a-4926-b70a-9e2967b748c0.png)

There is still an issue with this approach: face culling should be enabled (otherwise we'd render a light's effect twice) and 
when it is enabled the user may enter a light source's volume after which the volume isn't rendered anymore (due to back-face culling), 
removing the light source's influence; we can solve that by only rendering the spheres' back faces.

## Code
```C++
int main()
{
    ...
    // build and compile shaders
    // -------------------------
    Shader shaderGeometryPass("8.2.g_buffer.vs", "8.2.g_buffer.fs");
    Shader shaderLightingPass("8.2.deferred_shading.vs", "8.2.deferred_shading.fs");
    Shader shaderLightBox("8.2.deferred_light_box.vs", "8.2.deferred_light_box.fs");

    ...

    // configure g-buffer framebuffer
    // ------------------------------
    unsigned int gBuffer;
    ...

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window))
    {
        ...

        // 1. geometry pass: render scene's geometry/color data into gbuffer
        // -----------------------------------------------------------------
        ...

        // 2. lighting pass: calculate lighting by iterating over a screen filled quad pixel-by-pixel using the gbuffer's content.
        // -----------------------------------------------------------------------------------------------------------------------
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        shaderLightingPass.use();
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, gPosition);
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, gNormal);
        glActiveTexture(GL_TEXTURE2);
        glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
        // send light relevant uniforms, here we want to construct a sphere representing the lighting volume for a light source
        for (unsigned int i = 0; i < lightPositions.size(); i++)
        {
            shaderLightingPass.setVec3("lights[" + std::to_string(i) + "].Position", lightPositions[i]); // position
            shaderLightingPass.setVec3("lights[" + std::to_string(i) + "].Color", lightColors[i]); // color
            
            // update attenuation parameters and calculate radius
            const float constant = 1.0f; // note that we don't send this to the shader, we assume it is always 1.0 (in our case)
            const float linear = 0.7f;
            const float quadratic = 1.8f;
            shaderLightingPass.setFloat("lights[" + std::to_string(i) + "].Linear", linear);
            shaderLightingPass.setFloat("lights[" + std::to_string(i) + "].Quadratic", quadratic);
            
            // then calculate radius of light volume/sphere
            const float maxBrightness = std::fmaxf(std::fmaxf(lightColors[i].r, lightColors[i].g), lightColors[i].b);
            float radius = (-linear + std::sqrt(linear * linear - 4 * quadratic * (constant - (256.0f / 5.0f) * maxBrightness))) / (2.0f * quadratic);
            shaderLightingPass.setFloat("lights[" + std::to_string(i) + "].Radius", radius);
        }
        shaderLightingPass.setVec3("viewPos", camera.Position);
        // finally render quad
        renderQuad();

        // 2.5. copy content of geometry's depth buffer to default framebuffer's depth buffer
        // ----------------------------------------------------------------------------------
        ...

        // 3. render lights on top of scene
        // --------------------------------
        ...
    }
    ...
}
```
Similar as before but we now pass all required uniform for construct a light volume.


__Naive way of deffered_shadding.fs__
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D gAlbedoSpec;

struct Light {
    vec3 Position;
    vec3 Color;
    
    float Linear;
    float Quadratic;
    float Radius;
};
const int NR_LIGHTS = 32;
uniform Light lights[NR_LIGHTS];
uniform vec3 viewPos;

void main()
{             
    // retrieve data from gbuffer
    vec3 FragPos = texture(gPosition, TexCoords).rgb;
    vec3 Normal = texture(gNormal, TexCoords).rgb;
    vec3 Diffuse = texture(gAlbedoSpec, TexCoords).rgb;
    float Specular = texture(gAlbedoSpec, TexCoords).a;
    
    // then calculate lighting as usual
    vec3 lighting  = Diffuse * 0.1; // hard-coded ambient component
    vec3 viewDir  = normalize(viewPos - FragPos);
    for(int i = 0; i < NR_LIGHTS; ++i)
    {
        // calculate distance between light source and current fragment
        float distance = length(lights[i].Position - FragPos);
        if(distance < lights[i].Radius) // just an if to filter
        {
            // diffuse
            vec3 lightDir = normalize(lights[i].Position - FragPos);
            vec3 diffuse = max(dot(Normal, lightDir), 0.0) * Diffuse * lights[i].Color;
            // specular
            vec3 halfwayDir = normalize(lightDir + viewDir);  
            float spec = pow(max(dot(Normal, halfwayDir), 0.0), 16.0);
            vec3 specular = lights[i].Color * spec * Specular;
            // attenuation
            float attenuation = 1.0 / (1.0 + lights[i].Linear * distance + lights[i].Quadratic * distance * distance);
            diffuse *= attenuation;
            specular *= attenuation;
            lighting += diffuse + specular;
        }
    }    
    FragColor = vec4(lighting, 1.0);
}
```

__To really use__
1. Don't do any lighting calculation when first deffered render the scene.
```C++
glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
// ...render the scene without light
glBindFramebuffer(GL_FRAMEBUFFER, 0); // 绑定默认缓冲，渲染球体并着色
```
3. Construct the spheres in the C++ program. The spheres do not have any color in practice.(Imaginary)
4. Render the sphere
```C++
lightVolumeShader.use(); // In this shader consider the light, only the fragment on sphere will be rendered
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, gPosition);
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, gNormal);
glActiveTexture(GL_TEXTURE2);
glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
for (int i = 0; i < lightPositions.size(); i++) {
lightVolumeShader.setVec3("lightPos", lightPositions[i]);
renderSphere();
```


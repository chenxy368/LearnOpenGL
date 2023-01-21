# Light Casters
## 10 Cubes
```C++
float vertices[] = {
    // positions          // normals           // texture coords
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  0.0f,  0.0f,
     0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  1.0f,  0.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  1.0f,  1.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  1.0f,  1.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  0.0f,  1.0f,
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  0.0f,  0.0f,
    
    ...
    
    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,  0.0f,  1.0f,
     0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,  1.0f,  1.0f,
     0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,  1.0f,  0.0f,
     0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,  1.0f,  0.0f,
    -0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,  0.0f,  0.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,  0.0f,  1.0f
};
// positions all containers
glm::vec3 cubePositions[] = {
    glm::vec3( 0.0f,  0.0f,  0.0f),
    glm::vec3( 2.0f,  5.0f, -15.0f),
    glm::vec3(-1.5f, -2.2f, -2.5f),
    glm::vec3(-3.8f, -2.0f, -12.3f),
    glm::vec3( 2.4f, -0.4f, -3.5f),
    glm::vec3(-1.7f,  3.0f, -7.5f),
    glm::vec3( 1.3f, -2.0f, -2.5f),
    glm::vec3( 1.5f,  2.0f, -2.5f),
    glm::vec3( 1.5f,  0.2f, -1.5f),
    glm::vec3(-1.3f,  1.0f, -1.5f)
};
```
## Directional
When a light source is far away the light rays coming from the light source are close to parallel to each other. 
It looks like all the light rays are coming from the same direction, regardless of where the object and/or the viewer is. 
When a light source is modeled to be infinitely far away it is called a directional light since all its light rays have the same direction; 
it is independent of the location of the light source.  
![light_casters_directional](https://user-images.githubusercontent.com/98029669/213847809-0851b066-097f-469a-b82e-2c05dd48733b.png)  
__Fragment Shader__
```GLSL
struct Light {
    // vec3 position; // no longer necessary when using directional lights.
    vec3 direction;
  
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};
[...]
void main()
{
  vec3 lightDir = normalize(-light.direction);
  [...]
}
```
__Render Loop__
```C++
while (!glfwWindowShouldClose(window))
{
    ...
    // be sure to activate shader when setting uniforms/drawing objects
    lightingShader.use();
    lightingShader.setVec3("light.direction", -0.2f, -1.0f, -0.3f);
    ...
    
     // render containers(10 cubes)
     glBindVertexArray(cubeVAO);
     for (unsigned int i = 0; i < 10; i++)
     {
        // calculate the model matrix for each object and pass it to shader before drawing
        glm::mat4 model = glm::mat4(1.0f);
        model = glm::translate(model, cubePositions[i]);
        float angle = 20.0f * i;
        model = glm::rotate(model, glm::radians(angle), glm::vec3(1.0f, 0.3f, 0.5f));
        lightingShader.setMat4("model", model);

        glDrawArrays(GL_TRIANGLES, 0, 36);
     }


    // a lamp object is weird when we only have a directional light, don't render the light object(the white cube)
    // lightCubeShader.use();
    // lightCubeShader.setMat4("projection", projection);
    // lightCubeShader.setMat4("view", view);
    // model = glm::mat4(1.0f);
    // model = glm::translate(model, lightPos);
    // model = glm::scale(model, glm::vec3(0.2f)); // a smaller cube
    // lightCubeShader.setMat4("model", model);

    // glBindVertexArray(lightCubeVAO);
    // glDrawArrays(GL_TRIANGLES, 0, 36);

}
```
We've been passing the light's position and direction vectors as vec3s for a while now, but some people tend to prefer to keep all the vectors defined as vec4. 
When defining position vectors as a vec4 it is important to set the w component to 1.0 so translation and projections are properly applied. 
However, when defining a direction vector as a vec4 we don't want translations to have an effect (since they just represent directions, nothing more) so then we define the w component to be 0.0.

```GLSL
if(lightVector.w == 0.0) // note: be careful for floating point errors
  // do directional light calculations
else if(lightVector.w == 1.0)
  // do light calculations using the light's position (as in previous chapters)
``` 
 

# Colors
## How to get color with different light
__Note:__ Technically it's a bit more complicated, but we'll get to that in the PBR chapters.  
In the previous paragraph we had a white color so we'll give the light source a white color as well. 
If we would then multiply the light source's color with an object's color value, the resulting color would be the reflected color of the object (and thus its perceived color).  
__Note:__ * means element multiply
```C++
glm::vec3 lightColor(1.0f, 1.0f, 1.0f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (1.0f, 0.5f, 0.31f);

glm::vec3 lightColor(0.0f, 1.0f, 0.0f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (0.0f, 0.5f, 0.0f);

glm::vec3 lightColor(0.33f, 0.42f, 0.18f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (0.33f, 0.21f, 0.06f);
```
## Set a VAO for light
We want to have a light cube. Initialize a VAO for it.(The wight one if you run the exe.)  
```C++
unsigned int lightVAO;
glGenVertexArrays(1, &lightVAO);
glBindVertexArray(lightVAO);
// VBO is the same as the object cube
glBindBuffer(GL_ARRAY_BUFFER, VBO);
// Only position for light cube
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```
## Change Shader
Plus light color
```GLSL
#version 330 core
out vec4 FragColor;

uniform vec3 objectColor;
uniform vec3 lightColor;

void main()
{
    FragColor = vec4(lightColor * objectColor, 1.0);
}
```
```C++
// set uniform
lightingShader.use();
lightingShader.setVec3("objectColor", 1.0f, 0.5f, 0.31f);
lightingShader.setVec3("lightColor",  1.0f, 1.0f, 1.0f);
```
## Set light cube
For the light cube, just set wight.
```GLSL
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0);
}
```
```C++
glm::vec3 lightPos(1.2f, 1.0f, 2.0f);
...
model = glm::mat4();
model = glm::translate(model, lightPos);
model = glm::scale(model, glm::vec3(0.2f));
...
lampShader.use();
...
glBindVertexArray(lightVAO);
glDrawArrays(GL_TRIANGLES, 0, 36);
```

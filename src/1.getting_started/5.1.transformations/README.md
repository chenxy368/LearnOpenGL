# Transformations
## GLM
```C++
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>
```
## Translate
![translate](https://user-images.githubusercontent.com/98029669/213216787-f8213b69-1962-4ab5-b995-ec6f27f24aa4.png)
```C++
glm::vec4 vec(1.0f, 0.0f, 0.0f, 1.0f);
// above 0.9.9
// glm::mat4 trans = glm::mat4(1.0f)
glm::mat4 trans;
trans = glm::translate(trans, glm::vec3(1.0f, 1.0f, 0.0f));
vec = trans * vec;
```
## Scaling
![scaling](https://user-images.githubusercontent.com/98029669/213216609-eeb031eb-173d-42ac-a17c-073beb842dff.png)
## Rotation
![rotation](https://user-images.githubusercontent.com/98029669/213216996-0d4bbdfa-8422-4d63-8570-37f6907ce0b6.png)
![rotation1](https://user-images.githubusercontent.com/98029669/213217011-4934c869-7f02-4ddb-8af0-c90d85568508.png)
Quaternion:
![quaternion1](https://user-images.githubusercontent.com/98029669/213289556-887adf74-3aef-4e1d-8224-b64c2e1df22d.jpg)
![quaternion2](https://user-images.githubusercontent.com/98029669/213289573-2453c028-75e7-4214-a191-41aa89aebb46.jpg)
![quaternion3](https://user-images.githubusercontent.com/98029669/213289657-80289a5a-b9be-4abb-ab4a-1be331428c41.jpg)
![quaternion4](https://user-images.githubusercontent.com/98029669/213289680-23198984-848d-4577-ad9a-1751c6fd67a0.jpg)
![quaternion5](https://user-images.githubusercontent.com/98029669/213289690-825bd736-1b50-428a-9d43-da09962e051f.jpg)
![quaternion6](https://user-images.githubusercontent.com/98029669/213289712-6558395b-0736-47e4-a2f8-fc29c401c0cf.jpg)
![quaternion7](https://user-images.githubusercontent.com/98029669/213289719-e2387f22-e3e6-4e27-a905-a52752d3eaf2.jpg)

## Combination
![combination](https://user-images.githubusercontent.com/98029669/213218203-1fdcb365-465d-4fc4-ab79-58dd2918b13d.png)
Add a uniform to tansform: uniform mat4 transform
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;

out vec2 TexCoord;

uniform mat4 transform;

void main()
{
    gl_Position = transform * vec4(aPos, 1.0f);
    TexCoord = vec2(aTexCoord.x, 1.0 - aTexCoord.y);
}
```
Initialize transform matrix and pass
```C++
...
glm::mat4 trans;
trans = glm::rotate(trans, glm::radians(90.0f), glm::vec3(0.0, 0.0, 1.0));
trans = glm::scale(trans, glm::vec3(0.5, 0.5, 0.5));
...
    glm::mat4 trans;
    trans = glm::translate(trans, glm::vec3(0.5f, -0.5f, 0.0f));
    trans = glm::rotate(trans, (float)glfwGetTime(), glm::vec3(0.0f, 0.0f, 1.0f));
    
    unsigned int transformLoc = glGetUniformLocation(ourShader.ID, "transform");
    glUniformMatrix4fv(transformLoc, 1, GL_FALSE, glm::value_ptr(trans));
```
In most case, the sequence of transformations should flow: scale->rotation->translate, i.e. (translate * rotation * scale) * v 

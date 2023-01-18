Translate to another place and scale
```C++
// second transformation
transform = glm::mat4(1.0f); // reset it to identity matrix
transform = glm::translate(transform, glm::vec3(-0.5f, 0.5f, 0.0f));
float scaleAmount = static_cast<float>(sin(glfwGetTime()));
transform = glm::scale(transform, glm::vec3(scaleAmount, scaleAmount, scaleAmount));
glUniformMatrix4fv(transformLoc, 1, GL_FALSE, &transform[0][0]); // this time take the matrix value array's first element as its memory pointer value
```

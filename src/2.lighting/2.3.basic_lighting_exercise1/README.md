Move light source
```C++
// change the light's position values over time (can be done anywhere in the render loop actually, but try to do it at least before using the light source positions)
lightPos.x = 1.0f + sin(glfwGetTime()) * 2.0f;
lightPos.y = sin(glfwGetTime() / 2.0f) * 1.0f;
```

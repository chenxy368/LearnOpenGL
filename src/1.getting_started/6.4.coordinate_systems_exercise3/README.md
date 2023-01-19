```C++
if(i % 3 == 0)  // every 3rd iteration (including the first) we set the angle using GLFW's time function.
    angle = glfwGetTime() * 25.0f;
```

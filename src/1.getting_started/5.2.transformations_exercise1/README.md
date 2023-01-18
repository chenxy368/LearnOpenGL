First rotate then translate  
```C++
[...]
    while(!glfwWindowShouldClose(window))
    {
        [...]        
        // create transformations
        glm::mat4 transform = glm::mat4(1.0f);
        transform = glm::rotate(transform, (float)glfwGetTime(), glm::vec3(0.0f, 0.0f, 1.0f)); // switched the order
        transform = glm::translate(transform, glm::vec3(0.5f, -0.5f, 0.0f)); // switched the order               
        [...]
    }
```

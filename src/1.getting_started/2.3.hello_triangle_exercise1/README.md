# Hello Triangle
 Draw two triangles with six vertices
 ```C++
 float vertices[] = {
    // first triangle
    -0.9f, -0.5f, 0.0f,  // left 
    -0.0f, -0.5f, 0.0f,  // right
    0.45f, 0.5f, 0.0f,  // top 
    // second triangle
    0.0f, -0.5f, 0.0f,  // left
    0.9f, -0.5f, 0.0f,  // right
    0.45f, 0.5f, 0.0f   // top 
}; 
 ```
 
```C++
glDrawArrays(GL_TRIANGLES, 0, 6);
 ```

# Hello Triangle
 ## EBO
From
 ```C++
float vertices[] = {
    // first triangle
     0.5f,  0.5f, 0.0f,  // top right
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f,  0.5f, 0.0f,  // top left 
    // second triangle
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f, -0.5f, 0.0f,  // bottom left
    -0.5f,  0.5f, 0.0f   // top left
}; 
```
To
 ```C++
float vertices[] = {
     0.5f,  0.5f, 0.0f,  // top right
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f, -0.5f, 0.0f,  // bottom left
    -0.5f,  0.5f, 0.0f   // top left 
};
unsigned int indices[] = {  // note that we start from 0!
    0, 1, 3,   // first triangle
    1, 2, 3    // second triangle
};  
```

 ```C++
unsigned int EBO;
glGenBuffers(1, &EBO);

// note that this is allowed, the call to glVertexAttribPointer registered VBO as
//the vertex attribute's bound vertex buffer object so afterwards we can safely unbind
glBindBuffer(GL_ARRAY_BUFFER, 0); 

// remember: do NOT unbind the EBO while a VAO is active as the bound element buffer 
// object IS stored in the VAO; keep the EBO bound.
//glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);

// You can unbind the VAO afterwards so other VAO calls won't accidentally modify this VAO, 
// but this rarely happens. Modifying other VAOs requires a call to glBindVertexArray anyways 
// so we generally don't unbind VAOs (nor VBOs) when it's not directly necessary.
glBindVertexArray(0); 
```

```C++
glUseProgram(shaderProgram);
// seeing as we only have a single VAO there's no need to bind it every time, 
// but we'll do so to keep things a bit more organized
glBindVertexArray(VAO); 
//glDrawArrays(GL_TRIANGLES, 0, 6);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
// glBindVertexArray(0); // no need to unbind it every time 
```
![vertex_array_objects_ebo](https://user-images.githubusercontent.com/98029669/213012850-c821740c-4000-4461-8f64-d583117c4dcb.png)
![context](https://user-images.githubusercontent.com/98029669/213014165-37157cb1-6abd-48fa-a528-7f38a5a7a12b.jpg)

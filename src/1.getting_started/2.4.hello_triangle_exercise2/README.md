# Hello Triangle
 Draw two triangles with two VAOs and two VBOs
 ```C++
float firstTriangle[] = {
    -0.9f, -0.5f, 0.0f,  // left 
    -0.0f, -0.5f, 0.0f,  // right
    -0.45f, 0.5f, 0.0f,  // top 
};
float secondTriangle[] = {
    0.0f, -0.5f, 0.0f,  // left
    0.9f, -0.5f, 0.0f,  // right
    0.45f, 0.5f, 0.0f   // top 
};
unsigned int VBOs[2], VAOs[2];
// we can also generate multiple VAOs or buffers at the same time
glGenVertexArrays(2, VAOs); 
glGenBuffers(2, VBOs);
 ```
Remember only bind one VAO in context
```C++
glBindVertexArray(VAOs[0]);
glBindBuffer(GL_ARRAY_BUFFER, VBOs[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(firstTriangle), firstTriangle, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);	// Vertex attributes stay the same
glEnableVertexAttribArray(0);

glBindVertexArray(VAOs[1]);	// note that we bind to a different VAO now
glBindBuffer(GL_ARRAY_BUFFER, VBOs[1]);	// and a different VBO
glBufferData(GL_ARRAY_BUFFER, sizeof(secondTriangle), secondTriangle, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, (void*)0); // because the vertex data is tightly packed we can also specify 0 as the vertex attribute's stride to let OpenGL figure it out
glEnableVertexAttribArray(0);
 ```
 ```C++
glBindVertexArray(VAOs[0]);
glDrawArrays(GL_TRIANGLES, 0, 3);

glBindVertexArray(VAOs[1]);
glDrawArrays(GL_TRIANGLES, 0, 3);
 ```

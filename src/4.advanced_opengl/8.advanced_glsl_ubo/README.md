# Advanced Data
A buffer in OpenGL is, at its core, an object that manages a certain piece of GPU memory and nothing more.
We give meaning to a buffer when binding it to a specific buffer target. 
A buffer is only a vertex array buffer when we bind it to GL_ARRAY_BUFFER, but we could just as easily bind it to GL_ELEMENT_ARRAY_BUFFER.

So far we've been filling the buffer's memory by calling glBufferData, which allocates a piece of GPU memory and adds data into this memory. 
If we were to pass NULL as its data argument, the function would only allocate memory and not fill it. 
This is useful if we first want to reserve a specific amount of memory and later come back to this buffer.

## glBufferSubData
Fill specific regions of the buffer.
Do note that the buffer should have enough allocated memory so a call to glBufferData is necessary before calling.  
```C++
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(data), &data); // range: [24, 24 + sizeof(data)]
```

## glMapBuffer 
Directly use memcpy etc. to manage the gpu buffer by get a pointer.
```C++
float data[] = {
  0.5f, 1.0f, -0.35f
  [...]
};
glBindBuffer(GL_ARRAY_BUFFER, buffer);
// get pointer
void *ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// now copy data into memory
memcpy(ptr, data, sizeof(data));
// make sure to tell OpenGL we're done with the pointer
glUnmapBuffer(GL_ARRAY_BUFFER);
```

## Batching Vertex Attributes
Instead of an interleaved layout 123123123123 we take a batched approach 111122223333.
```C++
float positions[] = { ... };
float normals[] = { ... };
float tex[] = { ... };
// fill buffer
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(positions), &positions);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions), sizeof(normals), &normals);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals), sizeof(tex), &tex);
...

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), 0);  
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)(sizeof(positions)));  
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)(sizeof(positions) + sizeof(normals)));  
```

## Copying Buffers
```C++
void glCopyBufferSubData(GLenum readtarget, GLenum writetarget, GLintptr readoffset,
                         GLintptr writeoffset, GLsizeiptr size);
```
We could for example copy from a VERTEX_ARRAY_BUFFER buffer to a VERTEX_ELEMENT_ARRAY_BUFFER buffer by specifying those 
buffer targets as the read and write targets respectively.
If we wanted to read and write data into two different buffers that are both vertex array buffers, 
OpenGL gives us two more buffer targets called GL_COPY_READ_BUFFER and GL_COPY_WRITE_BUFFER.
```C++
float vertexData[] = { ... };
glBindBuffer(GL_COPY_READ_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```
We could've also done this by only binding the writetarget buffer to one of the new buffer target types:
```C++
float vertexData[] = { ... };
glBindBuffer(GL_ARRAY_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_ARRAY_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```
# Advanced GLSL

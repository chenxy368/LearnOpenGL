# Textures
 ## Multiple Textures
 OpenGL should have a at least a minimum of 16 texture units for you to use which you can activate using GL_TEXTURE0 to GL_TEXTURE15. They are defined in order so we could also get GL_TEXTURE8 via GL_TEXTURE0 + 8 for example, which is useful when we'd have to loop over several texture units.

Blending two textures in fragment shader
```GLSL
#version 330 core
...

uniform sampler2D texture1;
uniform sampler2D texture2;

void main()
{
    FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), 0.2);
}
```
Pass uniform
```C++
ourShader.use(); // don't forget to activate/use the shader before setting uniforms!
// either set it manually like so:
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0);
// or set it via the texture class
ourShader.setInt("texture2", 1);
```

In render loop
```C++
// Load two textures texture1 and texture2
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture1);
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, texture2);

glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0); 
 ```

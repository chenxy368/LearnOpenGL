Pass to xOffset
```C++
// In your CPP file:
// ======================
float offset = 0.5f;
ourShader.setFloat("xOffset", offset);
```
Define uniform xOffset
```GLSL
...
uniform float xOffset;
...
    gl_Position = vec4(aPos.x + xOffset, aPos.y, aPos.z, 1.0); 
    // add the xOffset to the x position of the vertex position
```

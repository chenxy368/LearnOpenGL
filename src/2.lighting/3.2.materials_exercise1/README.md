Some materials' parameter:[materials](http://devernay.free.fr/cours/opengl/materials.html)
```C++
// light properties
lightingShader.setVec3("light.ambient", 1.0f, 1.0f, 1.0f); // note that all light colors are set at full intensity
lightingShader.setVec3("light.diffuse", 1.0f, 1.0f, 1.0f);
lightingShader.setVec3("light.specular", 1.0f, 1.0f, 1.0f);

// material properties
lightingShader.setVec3("material.ambient", 0.0f, 0.1f, 0.06f);
lightingShader.setVec3("material.diffuse", 0.0f, 0.50980392f, 0.50980392f);
lightingShader.setVec3("material.specular", 0.50196078f, 0.50196078f, 0.50196078f);
lightingShader.setFloat("material.shininess", 32.0f);
```

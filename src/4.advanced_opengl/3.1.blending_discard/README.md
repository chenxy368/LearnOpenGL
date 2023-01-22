# Blending
## Introduction
Blending in OpenGL is commonly known as the technique to implement transparency within objects. 
Transparent objects can be completely transparent (letting all colors through) or partially transparent (letting colors through, but also some of its own colors).
The amount of transparency of an object is defined by its color's alpha value. 
The alpha color value is the 4th component of a color vector.
An alpha value of 0.0 would result in the object having complete transparency. 
An alpha value of 0.5 tells us the object's color consist of 50% of its own color and 50% of the colors behind the object.
![blending_transparency](https://user-images.githubusercontent.com/98029669/213894528-43eb789f-5b24-4666-b9f8-d6ae61f2530b.png)
## Discard Element
Some effects do not care about partial transparency, but either want to show something or nothing at all based on the color value of a texture.
E.g. grass, wherever there is no grass, the image shows the page's background color instead of its own.
![grass](https://user-images.githubusercontent.com/98029669/213894587-4c9d75d4-fd43-4cdc-ab40-eac334e30eb0.png)  
To load textures with alpha values(RGBA):
```C++
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
```
__Fragment Shader__ using alpha in fragment shader
```C++
void main()
{
    // FragColor = vec4(vec3(texture(texture1, TexCoords)), 1.0);
    FragColor = texture(texture1, TexCoords);
}
```
Add grass
```C++
vector<glm::vec3> vegetation;
vegetation.push_back(glm::vec3(-1.5f,  0.0f, -0.48f));
vegetation.push_back(glm::vec3( 1.5f,  0.0f,  0.51f));
vegetation.push_back(glm::vec3( 0.0f,  0.0f,  0.7f));
vegetation.push_back(glm::vec3(-0.3f,  0.0f, -2.3f));
vegetation.push_back(glm::vec3( 0.5f,  0.0f, -0.6f));
```
Grass need another VAO and also remember to bind corresponding Texture.
```C++
glBindVertexArray(vegetationVAO);
glBindTexture(GL_TEXTURE_2D, grassTexture);  
for(unsigned int i = 0; i < vegetation.size(); i++) 
{
    model = glm::mat4(1.0f);
    model = glm::translate(model, vegetation[i]);               
    shader.setMat4("model", model);
    glDrawArrays(GL_TRIANGLES, 0, 6);
}
```  
__!!!A very important issue:__  
![blending_no_discard](https://user-images.githubusercontent.com/98029669/213896689-ef6cff60-8429-4980-84f1-059f401926cb.png)  
This happens because OpenGL by default does not know what to do with alpha values, nor when to discard them. We have to manually do this ourselves. 
Luckily this is quite easy thanks to the use of shaders. 
GLSL gives us the discard command that (once called) ensures the fragment will not be further processed and thus not end up into the color buffer. 
We can check whether a fragment has an alpha value below a certain threshold and if so, discard the fragment as if it had never been processed:  
```GLSL
#version 330 core
out vec4 FragColor;

in vec2 TexCoords;

uniform sampler2D texture1;

void main()
{             
    vec4 texColor = texture(texture1, TexCoords);
    if(texColor.a < 0.1)
        discard;
    FragColor = texColor;
}
```
![blending_discard](https://user-images.githubusercontent.com/98029669/213896737-5e6fa539-13e4-4eca-8eda-061fb4772048.png)  
__!!!Texture Attribute__
OpenGL interpolates the border values with the next repeated value of the texture (because we set its wrapping parameters to GL_REPEAT by default). 
This is usually okay, but since we're using transparent values, the top of the texture image gets its transparent value interpolated with the bottom border's solid color value. 
The result is then a slightly semi-transparent colored border you may see wrapped around your textured quad.
To prevent this, set the texture wrapping method to GL_CLAMP_TO_EDGE whenever you use alpha textures that you don't want to repeat:  
```C++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);	
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE); 
```


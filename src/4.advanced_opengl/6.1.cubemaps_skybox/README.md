# Cubemaps
A cubemap is a texture that contains 6 individual 2D textures that each form one side of a cube: a textured cube.   
Cube maps have the useful property that they can be indexed/sampled using a direction vector. 
Imagine we have a 1x1x1 unit cube with the origin of a direction vector residing at its center. 
Sampling a texture value from the cube map with an orange direction vector looks a bit like this:  
![image](https://user-images.githubusercontent.com/98029669/213920222-51ec1515-fb57-4dc5-b649-eb1dc21f2bc0.png)  
If we imagine we have a cube shape that we attach such a cubemap to, this direction vector would be __similar to the (interpolated) local vertex position of the cube__. 
This way we can sample the cubemap using the cube's actual position vectors as long as the cube is centered on the origin. 
We thus consider __all vertex positions of the cube to be its texture coordinates when sampling a cubemap__. 
```C++
float skyboxVertices[] = {
    // positions          
   -1.0f,  1.0f, -1.0f,
   -1.0f, -1.0f, -1.0f,
    1.0f, -1.0f, -1.0f,
    1.0f, -1.0f, -1.0f,
    1.0f,  1.0f, -1.0f,
   -1.0f,  1.0f, -1.0f,

    ...

   -1.0f, -1.0f, -1.0f,
   -1.0f, -1.0f,  1.0f,
    1.0f, -1.0f, -1.0f,
    1.0f, -1.0f, -1.0f,
   -1.0f, -1.0f,  1.0f,
    1.0f, -1.0f,  1.0f
};
...
// skybox VAO
unsigned int skyboxVAO, skyboxVBO;
glGenVertexArrays(1, &skyboxVAO);
glGenBuffers(1, &skyboxVBO);
glBindVertexArray(skyboxVAO);
glBindBuffer(GL_ARRAY_BUFFER, skyboxVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(skyboxVertices), &skyboxVertices, GL_STATIC_DRAW);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
```
## Create Cubemaps(SkyBox)
![image](https://user-images.githubusercontent.com/98029669/213925862-9b8fafc3-3051-407b-81f7-7173ba382fde.png)  
1. Create and bind
```C++
unsigned int textureID;
glGenTextures(1, &textureID);
glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);
```
2. Load images: Enumerate: GL_TEXTURE_CUBE_MAP_POSITIVE_X, GL_TEXTURE_CUBE_MAP_NEGATIVE_X, 
GL_TEXTURE_CUBE_MAP_POSITIVE_Y, GL_TEXTURE_CUBE_MAP_NEGATIVE_Y, GL_TEXTURE_CUBE_MAP_POSITIVE_Z, GL_TEXTURE_CUBE_MAP_NEGATIVE_Z   
P.S. We set the wrapping method to GL_CLAMP_TO_EDGE since texture coordinates that are exactly between two faces may not hit an exact face (due to some hardware limitations) so by using GL_CLAMP_TO_EDGE OpenGL always returns their edge values whenever we sample between faces.  
```C++ 
    vector<std::string> faces
    {
        FileSystem::getPath("resources/textures/skybox/right.jpg"),
        FileSystem::getPath("resources/textures/skybox/left.jpg"),
        FileSystem::getPath("resources/textures/skybox/top.jpg"),
        FileSystem::getPath("resources/textures/skybox/bottom.jpg"),
        FileSystem::getPath("resources/textures/skybox/front.jpg"),
        FileSystem::getPath("resources/textures/skybox/back.jpg")
    };
    unsigned int cubemapTexture = loadCubemap(faces);

....
unsigned int loadCubemap(vector<std::string> faces)
{
    unsigned int textureID;
    glGenTextures(1, &textureID);
    glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);

    int width, height, nrChannels;
    for (unsigned int i = 0; i < faces.size(); i++)
    {
        unsigned char *data = stbi_load(faces[i].c_str(), &width, &height, &nrChannels, 0);
        if (data)
        {
            glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
            stbi_image_free(data);
        }
        else
        {
            std::cout << "Cubemap texture failed to load at path: " << faces[i] << std::endl;
            stbi_image_free(data);
        }
    }
    // Specify its wrapping and filtering methods
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);

    return textureID;
}
```
3. Vertex Shader
P.S. We want to achieve early depth testing here, which means we want to guarantee the skybox always at the farthest face. In fact, the skybox original cooradinates are very close to the camera since it is a 1X1X1 cube. We need to play some trick here.
P.S. Perspective division is performed after the vertex shader has run, dividing the gl_Position's xyz coordinates by its w component. The z component of the resulting division is equal to that vertex's depth value. To achieve early depth testing, we can set the z component of the output position equal to its w component which will result in a z component that is always equal to 1.0, because when the perspective division is applied its z component translates to w / w = 1.0. Therefore, The resulting normalized device coordinates will then always have a z value equal to 1.0: the maximum depth value.  
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;

out vec3 TexCoords;

uniform mat4 projection;
uniform mat4 view;

void main()
{
    TexCoords = aPos;
    vec4 pos = projection * view * vec4(aPos, 1.0);
    gl_Position = pos.xyww; // z component is always equal to 1.0
}  
```
4. Fragment Shader
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 TexCoords;

uniform samplerCube skybox;

void main()
{    
    FragColor = texture(skybox, TexCoords);
}
```
The fragment shader then takes these as input to sample a samplerCube.
```C++
skyboxShader.use();
skyboxShader.setInt("skybox", 0); // Later glActiveTexture(GL_TEXTURE0);
```
5. In render loop, remember activate texture  
P.S. We can remove the translation section of transformation matrices by taking the upper-left 3x3 matrix of the 4x4 matrix. We can achieve this by converting the view matrix to a 3x3 matrix (removing translation) and converting it back to a 4x4 matrix.  
P.S. We want to reder skybox last. If we render the skybox first, we're running the fragment shader for each pixel on the screen even though only a small part of the skybox will eventually be visible. In fact, fragments that could have easily been discarded using early depth testing saving us valuable bandwidth(Some trick has been played in vertex shader). We do have to change the depth function a little by setting it to GL_LEQUAL instead of the default GL_LESS, since the default is farthest face whose depth equals to skybox.
```C++
// Render scene
...
// draw skybox as last
glDepthFunc(GL_LEQUAL);  // change depth function so depth test passes when values are equal to depth buffer's content
skyboxShader.use();
view = glm::mat4(glm::mat3(camera.GetViewMatrix())); // remove translation from the view matrix
skyboxShader.setMat4("view", view);
skyboxShader.setMat4("projection", projection);
// skybox cube
glBindVertexArray(skyboxVAO);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);
glDrawArrays(GL_TRIANGLES, 0, 36);
glBindVertexArray(0);
glDepthFunc(GL_LESS); // set depth function back to default
 ```

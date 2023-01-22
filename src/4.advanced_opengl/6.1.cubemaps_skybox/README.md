# Cubemaps
A cubemap is a texture that contains 6 individual 2D textures that each form one side of a cube: a textured cube.   
Cube maps have the useful property that they can be indexed/sampled using a direction vector. 
Imagine we have a 1x1x1 unit cube with the origin of a direction vector residing at its center. 
Sampling a texture value from the cube map with an orange direction vector looks a bit like this:  
![image](https://user-images.githubusercontent.com/98029669/213920222-51ec1515-fb57-4dc5-b649-eb1dc21f2bc0.png)  
If we imagine we have a cube shape that we attach such a cubemap to, this direction vector would be __similar to the (interpolated) local vertex position of the cube__. 
This way we can sample the cubemap using the cube's actual position vectors as long as the cube is centered on the origin. 
We thus consider __all vertex positions of the cube to be its texture coordinates when sampling a cubemap__. 
## Create Cubemaps
1. Create and bind
```C++
unsigned int textureID;
glGenTextures(1, &textureID);
glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);
```
2. Load on faces
```C++
int width, height, nrChannels;
unsigned char *data;  
for(unsigned int i = 0; i < textures_faces.size(); i++)
{
    data = stbi_load(textures_faces[i].c_str(), &width, &height, &nrChannels, 0);
    glTexImage2D(
        GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 
        0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data
    );
}
```
Enumerate: GL_TEXTURE_CUBE_MAP_POSITIVE_X, GL_TEXTURE_CUBE_MAP_NEGATIVE_X, 
GL_TEXTURE_CUBE_MAP_POSITIVE_Y, GL_TEXTURE_CUBE_MAP_NEGATIVE_Y, GL_TEXTURE_CUBE_MAP_POSITIVE_Z, GL_TEXTURE_CUBE_MAP_NEGATIVE_Z  



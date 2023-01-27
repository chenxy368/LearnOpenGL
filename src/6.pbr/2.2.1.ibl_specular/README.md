# Specular IBL
Epic Games proposed a solution where they were able to pre-convolute the specular part for real time purposes, given a few compromises, 
known as the split sum approximation.

![1674840572630](https://user-images.githubusercontent.com/98029669/215154339-fe6b1ea4-672b-400d-93e3-2a26b2de4c55.png)

We were able to pre-compute the irradiance map as the integral only depended on ωi and we could move the constant diffuse albedo terms out of the integral. 
This time, the integral depends on more than just ωi as evident from the BRDF:

![1674840625946](https://user-images.githubusercontent.com/98029669/215154537-dfdcb3c4-601d-4b6b-9391-864b2328897f.png)

Epic Games' split sum approximation solves the issue by splitting the pre-computation into 2 individual parts that we can later combine to get the resulting pre-computed result we're after. 
The split sum approximation splits the specular integral into two separate integrals, __the first one do not influence by wo:__

![1674840694208](https://user-images.githubusercontent.com/98029669/215154768-145e591c-e5bc-419f-8353-91d122fe2021.png)

## Pre-filtering an HDR environment map
No wo in the first part.

![1674841033720](https://user-images.githubusercontent.com/98029669/215155912-a415cb03-5cac-46cc-a766-b42ee0ac45e9.png)

The first part (when convoluted) is known as the pre-filtered environment map which is (similar to the irradiance map) a pre-computed environment convolution map, but this time taking roughness into account. 
For increasing roughness levels, the environment map is convoluted with more scattered sample vectors, creating blurrier reflections. 
For each roughness level we convolute, we store the sequentially blurrier results in the pre-filtered map's mipmap levels.

![image](https://user-images.githubusercontent.com/98029669/215156108-06e8bfeb-d969-4eb1-a41d-2869917c017e.png)

We generate the sample vectors and their scattering amount using the normal distribution function (NDF) of the Cook-Torrance BRDF that __takes as input both a normal and view direction__. 
As we __don't know beforehand the view direction when convoluting the environment map(we only convolute the environment map once before all rendering)__, 
Epic Games makes a further approximation by __assuming the view direction (and thus the specular reflection direction) to be equal to the output sample direction ωo__.

```GLSL
vec3 N = normalize(w_o);
vec3 R = N;
vec3 V = R;
```
We do not need to care about view direction anymore while we have to admit it is really a rough approximation.

First, we need to generate a new cubemap to hold the pre-filtered environment map data. We store environment map with different roughness in different mipmap level.
To make sure we allocate enough memory for its mip levels we call glGenerateMipmap as an easy way to allocate the required amount of memory:
```C++
unsigned int prefilterMap;
glGenTextures(1, &prefilterMap);
glBindTexture(GL_TEXTURE_CUBE_MAP, prefilterMap);
for (unsigned int i = 0; i < 6; ++i)
{
    glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB16F, 128, 128, 0, GL_RGB, GL_FLOAT, nullptr);
}
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR); // tri-linear interpolation
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

glGenerateMipmap(GL_TEXTURE_CUBE_MAP);
```
### Monte Carlo integration and importance sampling
 When it comes to specular reflections, based on the roughness of a surface, the light reflects closely or roughly around a reflection vector r over a normal n, 
 but (unless the surface is extremely rough) around the reflection vector nonetheless:
 ![image](https://user-images.githubusercontent.com/98029669/215159108-78d6771f-fa1c-4cf8-9fda-94f48976880e.png)

The general shape of possible outgoing light reflections is known as the specular lobe. 
As roughness increases, the specular lobe's size increases; and the shape of the specular lobe changes on varying incoming light directions. 
The shape of the specular lobe is thus __highly dependent on the material__.

Theory of Monte Carlo integration can be finded on floder 6.pbr.

#### A low-discrepancy sequence
We can do Monte Carlo integration on something called low-discrepancy sequences which still generate random samples, but each sample is more evenly distributed.

![image](https://user-images.githubusercontent.com/98029669/215160230-dc5325b3-fb02-4518-9834-651fb538e5f9.png)

When using a low-discrepancy sequence for generating the Monte Carlo sample vectors, the process is known as Quasi-Monte Carlo integration. 
Quasi-Monte Carlo methods have a faster rate of convergence which makes them interesting for performance heavy applications.
 

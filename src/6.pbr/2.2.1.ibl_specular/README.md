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
 
The sequence we'll be using is known as the Hammersley Sequence as carefully described by Holger Dammertz. The Hammersley sequence is based on the Van Der Corput sequence which mirrors a decimal binary representation around its decimal point.

```GLSL
float RadicalInverse_VdC(uint bits) 
{
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    return float(bits) * 2.3283064365386963e-10; // / 0x100000000
}
// ----------------------------------------------------------------------------
// The GLSL Hammersley function gives us the low-discrepancy sample i of the total sample set of size N.
vec2 Hammersley(uint i, uint N)
{
    return vec2(float(i)/float(N), RadicalInverse_VdC(i));
}  
```

#### GGX Importance sampling
To build a sample vector, we need some way of orienting and biasing the sample vector towards the specular lobe of some surface roughness. 
__(Based on the roughness of a surface, we can calculate the ratio of microfacets roughly aligned to some vector h.)__
```GLSL
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness)
{
    float a = roughness*roughness;
	
    float phi = 2.0 * PI * Xi.x;
    float cosTheta = sqrt((1.0 - Xi.y) / (1.0 + (a*a - 1.0) * Xi.y));
    float sinTheta = sqrt(1.0 - cosTheta*cosTheta);
	
    // from spherical coordinates to cartesian coordinates
    vec3 H;
    H.x = cos(phi) * sinTheta;
    H.y = sin(phi) * sinTheta;
    H.z = cosTheta;
	
    // from tangent-space vector to world-space sample vector
    vec3 up        = abs(N.z) < 0.999 ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
    vec3 tangent   = normalize(cross(up, N));
    vec3 bitangent = cross(N, tangent);
	
    vec3 sampleVec = tangent * H.x + bitangent * H.y + N * H.z;
    return normalize(sampleVec);
}  
```
pre-filter convolution shader:
```GLSL
#version 330 core
out vec4 FragColor;
in vec3 localPos;

uniform samplerCube environmentMap;
uniform float roughness;

const float PI = 3.14159265359;

float RadicalInverse_VdC(uint bits);
vec2 Hammersley(uint i, uint N);
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness);
  
void main()
{		
    vec3 N = normalize(localPos);    
    vec3 R = N;
    vec3 V = R;

    const uint SAMPLE_COUNT = 1024u;
    float totalWeight = 0.0;   
    vec3 prefilteredColor = vec3(0.0);     
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H  = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(dot(N, L), 0.0);
        if(NdotL > 0.0)
        {
            prefilteredColor += texture(environmentMap, L).rgb * NdotL;
            totalWeight      += NdotL;
        }
    }
    prefilteredColor = prefilteredColor / totalWeight;

    FragColor = vec4(prefilteredColor, 1.0);
}  
```
We pre-filter the environment, based on some input roughness that varies over each mipmap level of the pre-filter cubemap (from 0.0 to 1.0), and store the result in prefilteredColor. The resulting prefilteredColor is divided by the total sample weight, where samples with less influence on the final result (for small NdotL) contribute less to the final weight.
#### Capturing pre-filter mipmap levels
```C++
prefilterShader.use();
prefilterShader.setInt("environmentMap", 0);
prefilterShader.setMat4("projection", captureProjection);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);

glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
unsigned int maxMipLevels = 5;
for (unsigned int mip = 0; mip < maxMipLevels; ++mip)
{
    // reisze framebuffer according to mip-level size.
    unsigned int mipWidth  = 128 * std::pow(0.5, mip);
    unsigned int mipHeight = 128 * std::pow(0.5, mip);
    glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, mipWidth, mipHeight);
    glViewport(0, 0, mipWidth, mipHeight);

    // the roughness is corresponding to the mipmap level
    float roughness = (float)mip / (float)(maxMipLevels - 1);
    prefilterShader.setFloat("roughness", roughness);
    for (unsigned int i = 0; i < 6; ++i)
    {
        prefilterShader.setMat4("view", captureViews[i]);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
                               GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, prefilterMap, mip);

        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        renderCube();
    }
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
The process is similar to the irradiance map convolution, but this time we scale the framebuffer's dimensions to the appropriate mipmap scale, each mip level reducing the dimensions by a scale of 2. Additionally, we specify the mip level we're rendering into in glFramebufferTexture2D's last parameter and pass the roughness we're pre-filtering for to the pre-filter shader.
### Pre-filter convolution artifacts
#### Cubemap seams at high roughness
OpenGL by default doesn't linearly interpolate across cubemap faces. Because the lower mip levels are both of a lower resolution and the pre-filter map is convoluted with a much larger sample lobe, the lack of between-cube-face filtering becomes quite apparent:

![image](https://user-images.githubusercontent.com/98029669/215243629-64b3828e-2022-49f1-a0ac-73ac1bf2adc6.png)

To solve, enable
```C++
glEnable(GL_TEXTURE_CUBE_MAP_SEAMLESS);  
```
#### Bright dots in the pre-filter convolution
Due to high frequency details and wildly varying light intensities in specular reflections, you'll start seeing dotted patterns emerge around bright areas:
![image](https://user-images.githubusercontent.com/98029669/215246337-a103ad5f-9dd1-4d1b-a3d8-e11944b9114d.png)

not directly sampling the environment map, but sampling a mip level of the environment map based on the integral's PDF and the roughness:
```GLSL
void main()
{		
    vec3 N = normalize(WorldPos);
    
    // make the simplyfying assumption that V equals R equals the normal 
    vec3 R = N;
    vec3 V = R;

    const uint SAMPLE_COUNT = 1024u;
    vec3 prefilteredColor = vec3(0.0);
    float totalWeight = 0.0;
    
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        // generates a sample vector that's biased towards the preferred alignment direction (importance sampling).
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(dot(N, L), 0.0);
        // if(NdotL > 0.0)
        // {
              // Here we directly sample the environmentMap regardless the roughness.
        //    prefilteredColor += texture(environmentMap, L).rgb * NdotL;
        //    totalWeight      += NdotL;
        // }
    
        if(NdotL > 0.0)
        {
            // sample from the environment's mip level based on roughness/pdf
            float D   = DistributionGGX(N, H, roughness);
            float NdotH = max(dot(N, H), 0.0);
            float HdotV = max(dot(H, V), 0.0);
            float pdf = D * NdotH / (4.0 * HdotV) + 0.0001; 

            float resolution = 512.0; // resolution of source cubemap (per face)
            float saTexel  = 4.0 * PI / (6.0 * resolution * resolution);
            float saSample = 1.0 / (float(SAMPLE_COUNT) * pdf + 0.0001);

            // Compute the mipLevel
            float mipLevel = roughness == 0.0 ? 0.0 : 0.5 * log2(saSample / saTexel); 
            
            prefilteredColor += textureLod(environmentMap, L, mipLevel).rgb * NdotL;
            totalWeight      += NdotL;
        }
    }

    prefilteredColor = prefilteredColor / totalWeight;

    FragColor = vec4(prefilteredColor, 1.0);
}
```

enable trilinear filtering on the environment map
```C++
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR); 
```
generate the mipmaps 
```C++
// convert HDR equirectangular environment map to cubemap equivalent
[...]
// then generate mipmaps
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);
glGenerateMipmap(GL_TEXTURE_CUBE_MAP);
```

## Pre-computing the BRDF
The second part of the split sum equation equals the BRDF part of the specular integral. If we pretend the incoming radiance is completely white for every direction __(thus L(p,x)=1.0)__ we can pre-calculate the BRDF's response given an input __roughness and an input angle between the normal n and light direction ωi, or n⋅ωi__. 

![1674887779856](https://user-images.githubusercontent.com/98029669/215250898-1bf806b2-8c04-455e-bf1c-85f1aa311c09.png)

![image](https://user-images.githubusercontent.com/98029669/215250912-cda71203-a897-4142-b2ca-08743204efe4.png)

We store the convoluted results in a 2D lookup texture (LUT) known as a BRDF integration map that we later use in our PBR lighting shader to get the final convoluted indirect specular result.

![image](https://user-images.githubusercontent.com/98029669/215250967-87771a12-919c-4fac-a910-1de056db978a.png)

![image](https://user-images.githubusercontent.com/98029669/215253756-c8a6a10a-41b3-4f6d-a3f4-cca4ebefeca7.png)

![image](https://user-images.githubusercontent.com/98029669/215255395-40d7c0cf-70bd-4dc2-9117-029435d02214.png)

```GLSL
vec2 IntegrateBRDF(float NdotV, float roughness)
{
    vec3 V;
    V.x = sqrt(1.0 - NdotV*NdotV);
    V.y = 0.0;
    V.z = NdotV;

    float A = 0.0;
    float B = 0.0;

    vec3 N = vec3(0.0, 0.0, 1.0);

    const uint SAMPLE_COUNT = 1024u;
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H  = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(L.z, 0.0);
        float NdotH = max(H.z, 0.0);
        float VdotH = max(dot(V, H), 0.0);

        if(NdotL > 0.0)
        {
            float G = GeometrySmith(N, V, L, roughness);
            float G_Vis = (G * VdotH) / (NdotH * NdotV);
            float Fc = pow(1.0 - VdotH, 5.0);

            A += (1.0 - Fc) * G_Vis;
            B += Fc * G_Vis;
        }
    }
    A /= float(SAMPLE_COUNT);
    B /= float(SAMPLE_COUNT);
    return vec2(A, B);
}
// ----------------------------------------------------------------------------
void main() 
{
    vec2 integratedBRDF = IntegrateBRDF(TexCoords.x, TexCoords.y);
    FragColor = integratedBRDF;
}
```
Notice k in G is different from before

![1674894348529](https://user-images.githubusercontent.com/98029669/215255619-a7a53dca-fa20-40df-b861-379f8856a85a.png)

```GLSL
float GeometrySchlickGGX(float NdotV, float roughness)
{
    float a = roughness;
    float k = (a * a) / 2.0;

    float nom   = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return nom / denom;
}
// ----------------------------------------------------------------------------
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2 = GeometrySchlickGGX(NdotV, roughness);
    float ggx1 = GeometrySchlickGGX(NdotL, roughness);

    return ggx1 * ggx2;
}  
```

Save to 2D texture
```C++
unsigned int brdfLUTTexture;
glGenTextures(1, &brdfLUTTexture);

// pre-allocate enough memory for the LUT texture.
glBindTexture(GL_TEXTURE_2D, brdfLUTTexture);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RG16F, 512, 512, 0, GL_RG, GL_FLOAT, 0);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR); 
```

Note that we use a 16-bit precision floating format as recommended by Epic Games. Be sure to set the wrapping mode to GL_CLAMP_TO_EDGE to prevent edge sampling artifacts.

Render to FBO
```C++
glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, 512, 512);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, brdfLUTTexture, 0);

glViewport(0, 0, 512, 512);
brdfShader.use();
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
RenderQuad();

glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```

## Completing the IBL reflectance
Prefilter light + BRDF intergration
```GLSL
uniform samplerCube prefilterMap;
uniform sampler2D   brdfLUT;  
```
From roughness to mipmap level
```GLSL

void main()
{
    [...]
    vec3 R = reflect(-V, N);   

    const float MAX_REFLECTION_LOD = 4.0;
    vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;    
    [...]
}
```

Combine two parts of specular
```GLSL
vec3 F        = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);
```

Combine with diffuse
```GLSL
vec3 F = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);

vec3 kS = F;
vec3 kD = 1.0 - kS;
kD *= 1.0 - metallic;     

vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;

const float MAX_REFLECTION_LOD = 4.0;
vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;   
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);

// Note that we don't multiply specular by kS as we already have a Fresnel multiplication in there.
vec3 ambient = (kD * diffuse + specular) * ao; 
```



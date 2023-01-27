# PBR Theory
## The Microfacet Model
All the PBR techniques are based on the theory of microfacets. 
The theory describes that any surface at a microscopic scale can be described by tiny little perfectly reflective mirrors called microfacets.   
![image](https://user-images.githubusercontent.com/98029669/214962988-5690ed2e-5ec3-4776-9f2f-95fd8a5cd1c7.png)  
No surface is completely smooth on a microscopic level, but seeing as these microfacets are small enough that we can't make a distinction between them on a per-pixel basis, we statistically approximate the surface's microfacet roughness given a roughness parameter. 
Based on the roughness of a surface, we can calculate the ratio of microfacets roughly aligned to some vector h.
This vector h is the halfway vector that sits halfway between the light l and view v vector.  
![1674771562669](https://user-images.githubusercontent.com/98029669/214963649-1c6b3ce0-3c09-4d56-84f0-25a74937fcb7.png)  
The more the microfacets are aligned to the halfway vector, the sharper and stronger the specular reflection.  
![image](https://user-images.githubusercontent.com/98029669/214963736-4115f68d-94a6-430d-b0f3-5c6dfda5fc6b.png)  

## Energy Conservation
The microfacet approximation employs a form of energy conservation: outgoing light energy should never exceed the incoming light energy (excluding emissive surfaces).  
![image](https://user-images.githubusercontent.com/98029669/214964087-15b10a31-45c5-4656-9d46-e2cd6a8cca90.png)  
An additional subtlety when it comes to reflection and refraction are surfaces that are metallic. 
Metallic surfaces react different to light compared to non-metallic surfaces (also known as dielectrics). 
Metallic surfaces follow the same principles of reflection and refraction, but all refracted light gets directly absorbed without scattering. 
This means metallic surfaces only leave reflected or specular light; metallic surfaces show no diffuse colors.
Because of this apparent distinction between metals and dielectrics, they're both treated differently in the PBR pipeline which we'll delve into further down the chapter.   
This distinction between reflected and refracted light brings us to another observation regarding energy preservation: they're mutually exclusive. 
Whatever light energy gets reflected will no longer be absorbed by the material itself.   
```GLSL
float kS = calculateSpecularComponent(...); // reflection/specular fraction
float kD = 1.0 - kS;                        // refraction/diffuse  fraction
```

## The Reflectance Equation
![1674772030340](https://user-images.githubusercontent.com/98029669/214964896-749c35d2-a0b1-4373-98f0-d03f6d746758.png)  
The result of small discrete steps of the reflectance equation over the hemisphere Ω and averaging their results over the step size. 
This is known as the Riemann sum that we can roughly visualize in code as follows:
```GLSL
int steps = 100;
float sum = 0.0f;
vec3 P    = ...;
vec3 Wo   = ...;
vec3 N    = ...;
float dW  = 1.0f / steps;
for(int i = 0; i < steps; ++i) 
{
    vec3 Wi = getNextIncomingLightDir(i);
    sum += Fr(P, Wi, Wo) * L(P, Wi) * dot(N, Wi) * dW;
}
```
## BRDF
Fr() model in this case, Cook-Torrance BRDF:  
![1674772209309](https://user-images.githubusercontent.com/98029669/214965404-4ea15a57-34ac-46d5-9a87-c91d76f25033.png)  
The left side of the BRDF states the diffuse part of the equation denoted here as flambert.
This is known as Lambertian diffuse similar to what we used for diffuse shading, which is a constant factor denoted as:  
![1674772286671](https://user-images.githubusercontent.com/98029669/214965599-00294651-2309-4c07-a03a-da814ed5accf.png)  
With c being the albedo or surface color (think of the diffuse surface texture). 
The divide by pi is there to normalize the diffuse light as the earlier denoted integral that contains the BRDF is scaled by π.  

![1674772516393(1)](https://user-images.githubusercontent.com/98029669/214966166-5ff31a77-50c2-44a0-ae65-0acc2242754a.png)

Each of the D, F and G symbols represent a type of function that approximates a specific part of the surface's reflective properties. These are defined as the normal Distribution function, the Fresnel equation and the Geometry function:

__Normal distribution function:__ approximates the amount the surface's microfacets are aligned to the halfway vector, influenced by the roughness of the surface; this is the primary function approximating the microfacets.

__Geometry function:__ describes the self-shadowing property of the microfacets. When a surface is relatively rough, the surface's microfacets can overshadow other microfacets reducing the light the surface reflects.

__Fresnel equation:__ The Fresnel equation describes the ratio of surface reflection at different surface angles.

We're going to pick the same functions used by Epic Game's Unreal Engine 4 which are the Trowbridge-Reitz GGX for D, the Fresnel-Schlick approximation for F, and the Smith's Schlick-GGX for G.
 
### Normal distribution function
The normal distribution function D statistically approximates the relative surface area of microfacets exactly aligned to the (halfway) vector h.  
![1674785635032](https://user-images.githubusercontent.com/98029669/214995812-5020752d-17d9-4bb7-a4f1-d35a6ba18114.png)  
Here h is the halfway vector to measure against the surface's microfacets, with a being a measure of the surface's roughness. If we take h
as the halfway vector between the surface normal and light direction over varying roughness parameters we get the following visual result:
![image](https://user-images.githubusercontent.com/98029669/214995927-9b61403e-803f-48d0-a82c-ee7d4fd32a8a.png)
```GLSL
float DistributionGGX(vec3 N, vec3 H, float a)
{
    float a2     = a*a;
    float NdotH  = max(dot(N, H), 0.0);
    float NdotH2 = NdotH*NdotH;
	
    float nom    = a2;
    float denom  = (NdotH2 * (a2 - 1.0) + 1.0);
    denom        = PI * denom * denom;
	
    return nom / denom;
}
```
### Geometry function
The geometry function statistically approximates the relative surface area where its micro surface-details overshadow each other, causing light rays to be occluded.

![image](https://user-images.githubusercontent.com/98029669/214996201-3dbd7cb7-9cd7-429f-81b3-a1678353959a.png)

Similar to the NDF, the Geometry function takes a material's roughness parameter as input with rougher surfaces having a higher probability of overshadowing microfacets. The geometry function we will use is a combination of the GGX and Schlick-Beckmann approximation known as Schlick-GGX:

![1674785894431](https://user-images.githubusercontent.com/98029669/214996232-b35ec653-c42d-421e-92ea-c43b5e7ee1ba.png)

Here k is a remapping of α based on whether we're using the geometry function for either direct lighting or IBL lighting:
![1674785907612](https://user-images.githubusercontent.com/98029669/214996257-51dbb53a-73f1-444c-a168-e852ebb13651.png)

To effectively approximate the geometry we need to take account of both the view direction(v) (geometry obstruction) and the light direction vector(l) (geometry shadowing). We can take both into account using Smith's method:

![1674786581175](https://user-images.githubusercontent.com/98029669/214997563-3dbfdf20-6500-4716-b219-2984437f9b89.png)

![image](https://user-images.githubusercontent.com/98029669/214997623-a4381768-df61-4b95-9c14-ff2262ea3c1e.png)
```GLSL
float GeometrySchlickGGX(float NdotV, float k)
{
    float nom   = NdotV;
    float denom = NdotV * (1.0 - k) + k;
	
    return nom / denom;
}
  
float GeometrySmith(vec3 N, vec3 V, vec3 L, float k)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx1 = GeometrySchlickGGX(NdotV, k);
    float ggx2 = GeometrySchlickGGX(NdotL, k);
	
    return ggx1 * ggx2;
}
```
### Fresnel equation
![1674786700345](https://user-images.githubusercontent.com/98029669/214997815-f894e989-12bf-4f05-ad57-63cfce2c2e24.png)
![1674786754242](https://user-images.githubusercontent.com/98029669/214997892-ae5c5f35-80d8-44aa-bdde-835b77865052.png)

What is interesting to observe here is that for all __dielectric__ surfaces the base reflectivity __never gets above 0.17__ which is the exception rather than the rule, while for __conductors__ the base reflectivity __starts much higher and (mostly) varies between 0.5 and 1.0__. Furthermore, for __conductors (or metallic surfaces)__ the base reflectivity is tinted. This is __why F0 is presented as an RGB triplet__ (reflectivity at normal incidence can vary per wavelength); this is something we only see at metallic surfaces.

These specific attributes of metallic surfaces compared to dielectric surfaces __gave rise to something called the metallic workflow__. In the metallic workflow we author surface materials with an extra parameter known as metalness that describes __whether a surface is either a metallic or a non-metallic surface__.

```GLSL
vec3 F0 = vec3(0.04);
F0      = mix(F0, surfaceColor.rgb, metalness);
```
```GLSL
vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}
```
With cosTheta being the dot product result between the surface's normal n and the halfway h (or view v) direction.
## Cook-Torrance reflectance equation
![1674787080269](https://user-images.githubusercontent.com/98029669/214998475-62ac8021-e076-488a-9554-f08ab920c2d0.png)
This equation is not fully mathematically correct however. You may remember that the Fresnel term F represents the ratio of light that gets reflected on a surface. This is effectively our ratio ks, meaning the specular (BRDF) part of the reflectance equation implicitly contains the reflectance ratio ks. Given this, our final final reflectance equation becomes:

![1674787105461](https://user-images.githubusercontent.com/98029669/214998533-db0cfbd1-261d-4137-87c2-21f7fe335d91.png)
## Authoring PBR materials
![image](https://user-images.githubusercontent.com/98029669/214999032-8d7347be-7b20-4c94-8694-276994c5d7b8.png)

__Albedo(color, F0 in Fresnel if metallic):__ the albedo texture specifies for each texel the color of the surface, or the base reflectivity if that texel is metallic. This is largely similar to what we've been using before as a diffuse texture, but all lighting information is extracted from the texture. Diffuse textures often have slight shadows or darkened crevices inside the image which is something you don't want in an albedo texture; it should only contain the color (or refracted absorption coefficients) of the surface.

Normal: the normal map texture is exactly as we've been using before in the normal mapping chapter. The normal map allows us to specify, per fragment, a unique normal to give the illusion that a surface is bumpier than its flat counterpart.

Metallic(Metal or not): the metallic map specifies per texel whether a texel is either metallic or it isn't. Based on how the PBR engine is set up, artists can author metalness as either grayscale values or as binary black or white.

Roughness(a in normal distribution and geometry functions): the roughness map specifies how rough a surface is on a per texel basis. The sampled roughness value of the roughness influences the statistical microfacet orientations of the surface. A rougher surface gets wider and blurrier reflections, while a smooth surface gets focused and clear reflections. Some PBR engines expect a smoothness map instead of a roughness map which some artists find more intuitive. These values are then translated (1.0 - smoothness) to roughness the moment they're sampled.

AO: the ambient occlusion or AO map specifies an __extra shadowing factor__ of the surface and __potentially surrounding geometry__. If we have a brick surface for instance, the albedo texture should have no shadowing information inside the brick's crevices. The AO map however does specify these darkened edges as it's more difficult for light to escape. __Taking ambient occlusion in account at the end of the lighting stage can significantly boost the visual quality of your scene__. The ambient occlusion map of a mesh/surface is either manually generated, or pre-calculated in 3D modeling programs.

# PBR Lighting
![image](https://user-images.githubusercontent.com/98029669/215002268-3c1bf09d-8ab5-4bd8-8e11-cec9be82b7a5.png)
To put this in more practical terms: in the case of a direct point light the radiance function L measures the light color, attenuated over its distance to p and scaled by n⋅wi, but only over the single light ray wi that hits p which equals the light's direction vector from p. In code this translates to:
```GLSL
vec3  lightColor  = vec3(23.47, 21.31, 20.79);
vec3  wi          = normalize(lightPos - fragPos);
float cosTheta    = max(dot(N, Wi), 0.0);
float attenuation = calculateAttenuation(fragPos, lightPos);
float radiance    = lightColor * attenuation * cosTheta;
```
## A PBR surface model
In fragment shader:
```GLSL
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
in vec3 WorldPos;
in vec3 Normal;
  
uniform vec3 camPos;
  
uniform vec3  albedo;
uniform float metallic;
uniform float roughness;
uniform float ao;
```
## Direct Light
In this chapter's example demo we have a total of 4 point lights that together represent the scene's irradiance.
```GLSL
vec3 Lo = vec3(0.0);
for(int i = 0; i < 4; ++i) 
{
    vec3 L = normalize(lightPositions[i] - WorldPos);
    vec3 H = normalize(V + L);
  
    float distance    = length(lightPositions[i] - WorldPos);
    float attenuation = 1.0 / (distance * distance);
    vec3 radiance     = lightColors[i] * attenuation; 
    [...]  
 }
 ```
 Then, for each light we want to calculate the full Cook-Torrance specular BRDF term:  
 ![1674789733143](https://user-images.githubusercontent.com/98029669/215003160-33f785d8-b959-4200-8b10-a7d154140f0d.png)
 
F(Calculate the ratio between specular and diffuse reflection):
 ```GLSL
vec3 F0 = vec3(0.04); 
F0      = mix(F0, albedo, metallic);
vec3 F  = fresnelSchlick(max(dot(H, V), 0.0), F0);
 ...
 vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}  
```

D and G:
```GLSL
float DistributionGGX(vec3 N, vec3 H, float roughness)
{
    float a      = roughness*roughness;
    float a2     = a*a;
    float NdotH  = max(dot(N, H), 0.0);
    float NdotH2 = NdotH*NdotH;

    float nom   = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;

    return nom / denom;
}

float GeometrySchlickGGX(float NdotV, float roughness)
{
    float r = (roughness + 1.0);
    float k = (r*r) / 8.0;

    float nom   = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return nom / denom;
}
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2  = GeometrySchlickGGX(NdotV, roughness);
    float ggx1  = GeometrySchlickGGX(NdotL, roughness);

    return ggx1 * ggx2;
}
```

Cook-Torrance BRDF:
```GLSL
float NDF = DistributionGGX(N, H, roughness);       
float G   = GeometrySmith(N, V, L, roughness);  

vec3 nominator    = NDF * G * F;
float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.001; 
vec3 specular     = nominator / denominator;  
```
This is the reflection term, we also need to work on refraction term whose proportion is Kd = 1-Ks = 1-F
```GLSL
vec3 kS = F;
vec3 kD = vec3(1.0) - kS;

kD *= 1.0 - metallic; // Metal does not refract
```
```GLSL
const float PI = 3.14159265359;

float NdotL = max(dot(N, L), 0.0);        
Lo += (kD * albedo / PI + specular) * radiance * NdotL;
```

Add an (improvised) ambient term to the direct lighting result Lo
```GLSL
vec3 ambient = vec3(0.03) * albedo * ao;
vec3 color   = ambient + Lo;  
```
## Linear and HDR rendering
```GLSL
color = color / (color + vec3(1.0)); // To LDR
color = pow(color, vec3(1.0/2.2));  // Gamma Correction
```
## Full Fragment Shader
```GLSL
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
in vec3 WorldPos;
in vec3 Normal;

// material parameters
uniform vec3 albedo;
uniform float metallic;
uniform float roughness;
uniform float ao;

// lights(4 lights)
uniform vec3 lightPositions[4];
uniform vec3 lightColors[4];

uniform vec3 camPos;

const float PI = 3.14159265359;
// ----------------------------------------------------------------------------
// Normal Distribution
float DistributionGGX(vec3 N, vec3 H, float roughness)
{
    float a = roughness*roughness;
    float a2 = a*a;
    float NdotH = max(dot(N, H), 0.0);
    float NdotH2 = NdotH*NdotH;

    float nom   = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;

    return nom / denom;
}
// ----------------------------------------------------------------------------
// Geometry
float GeometrySchlickGGX(float NdotV, float roughness)
{
    float r = (roughness + 1.0);
    float k = (r*r) / 8.0;

    float nom   = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return nom / denom;
}
// ----------------------------------------------------------------------------
// Combine two sub geometry
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2 = GeometrySchlickGGX(NdotV, roughness);
    float ggx1 = GeometrySchlickGGX(NdotL, roughness);

    return ggx1 * ggx2;
}
// ----------------------------------------------------------------------------
// Fresnel to get proportion of reflection
vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
}
// ----------------------------------------------------------------------------
void main()
{	
    // In world space
    vec3 N = normalize(Normal);
    vec3 V = normalize(camPos - WorldPos);

    // calculate reflectance at normal incidence; if dia-electric (like plastic) use F0 
    // of 0.04 and if it's a metal, use the albedo color as F0 (metallic workflow)    
    vec3 F0 = vec3(0.04); 
    F0 = mix(F0, albedo, metallic);

    // reflectance equation, just loop over four direct light
    vec3 Lo = vec3(0.0);
    for(int i = 0; i < 4; ++i) 
    {
        // calculate per-light radiance
        vec3 L = normalize(lightPositions[i] - WorldPos);
        vec3 H = normalize(V + L);
        float distance = length(lightPositions[i] - WorldPos);
        float attenuation = 1.0 / (distance * distance);
        vec3 radiance = lightColors[i] * attenuation;

        // Cook-Torrance BRDF
        float NDF = DistributionGGX(N, H, roughness);   
        float G   = GeometrySmith(N, V, L, roughness);      
        vec3 F    = fresnelSchlick(clamp(dot(H, V), 0.0, 1.0), F0);
           
        vec3 numerator    = NDF * G * F; 
        float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.0001; // + 0.0001 to prevent divide by zero
        vec3 specular = numerator / denominator;
        
        // kS is equal to Fresnel, reflection
        vec3 kS = F;
        // for energy conservation, the diffuse and specular light can't
        // be above 1.0 (unless the surface emits light); to preserve this
        // relationship the diffuse component (kD) should equal 1.0 - kS.
        vec3 kD = vec3(1.0) - kS;
        // multiply kD by the inverse metalness such that only non-metals 
        // have diffuse lighting, or a linear blend if partly metal (pure metals
        // have no diffuse light).
        kD *= 1.0 - metallic;	  

        // scale light by NdotL
        float NdotL = max(dot(N, L), 0.0);        

        // add to outgoing radiance Lo
        Lo += (kD * albedo / PI + specular) * radiance * NdotL;  // note that we already multiplied the BRDF by the Fresnel (kS) so we won't multiply by kS again
    }   
    
    // ambient lighting (note that the next IBL tutorial will replace 
    // this ambient lighting with environment lighting).
    vec3 ambient = vec3(0.03) * albedo * ao;

    vec3 color = ambient + Lo;

    // HDR tonemapping
    color = color / (color + vec3(1.0));
    // gamma correct
    color = pow(color, vec3(1.0/2.2)); 

    FragColor = vec4(color, 1.0);
}
```

# PBR Theory
All the PBR techniques are based on the theory of microfacets. 
The theory describes that any surface at a microscopic scale can be described by tiny little perfectly reflective mirrors called microfacets.   
![image](https://user-images.githubusercontent.com/98029669/214962988-5690ed2e-5ec3-4776-9f2f-95fd8a5cd1c7.png)  
No surface is completely smooth on a microscopic level, but seeing as these microfacets are small enough that we can't make a distinction between them on a per-pixel basis, we statistically approximate the surface's microfacet roughness given a roughness parameter. 
Based on the roughness of a surface, we can calculate the ratio of microfacets roughly aligned to some vector h.
This vector h is the halfway vector that sits halfway between the light l and view v vector.  
![1674771562669](https://user-images.githubusercontent.com/98029669/214963649-1c6b3ce0-3c09-4d56-84f0-25a74937fcb7.png)  
The more the microfacets are aligned to the halfway vector, the sharper and stronger the specular reflection.  
![image](https://user-images.githubusercontent.com/98029669/214963736-4115f68d-94a6-430d-b0f3-5c6dfda5fc6b.png)  
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

BRDF:
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
Fr() model in this case, Cook-Torrance BRDF:  
![1674772209309](https://user-images.githubusercontent.com/98029669/214965404-4ea15a57-34ac-46d5-9a87-c91d76f25033.png)  
The left side of the BRDF states the diffuse part of the equation denoted here as flambert.
This is known as Lambertian diffuse similar to what we used for diffuse shading, which is a constant factor denoted as:  
![1674772286671](https://user-images.githubusercontent.com/98029669/214965599-00294651-2309-4c07-a03a-da814ed5accf.png)
With c being the albedo or surface color (think of the diffuse surface texture). 
The divide by pi is there to normalize the diffuse light as the earlier denoted integral that contains the BRDF is scaled by π.  
![1674772516393(1)](https://user-images.githubusercontent.com/98029669/214966166-5ff31a77-50c2-44a0-ae65-0acc2242754a.png)

 

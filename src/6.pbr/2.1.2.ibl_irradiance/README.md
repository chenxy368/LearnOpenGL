# IBL Diffuse Irradiance
As the ambient light comes from all directions within the hemisphere oriented around the normal N, there's no single halfway vector to determine the Fresnel response. 
To still simulate Fresnel, we calculate the Fresnel from the angle between the normal and view vector.
However, earlier we used the micro-surface halfway vector, influenced by the roughness of the surface, as input to the Fresnel equation. 
As we currently don't take roughness into account, the surface's reflective ratio will always end up relatively high. 

We can alleviate the issue by injecting a roughness term in the Fresnel-Schlick equation as described by Sébastien Lagarde:
```GLSL
vec3 fresnelSchlickRoughness(float cosTheta, vec3 F0, float roughness)
{
    return F0 + (max(vec3(1.0 - roughness), F0) - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
} 
```

```GLSL
vec3 kS = fresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness); 
vec3 kD = 1.0 - kS;
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
vec3 ambient    = (kD * diffuse) * ao; 
```

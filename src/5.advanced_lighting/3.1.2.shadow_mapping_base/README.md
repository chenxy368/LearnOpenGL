# Shadow Mapping
## Use Depth Map to Render Shadow
1. Vertex Shader:
We do the light-space transformation in the vertex shader(T(P))
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace; // Position in light space which is the same as in shadow map
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat4 lightSpaceMatrix;

void main()
{    
    vs_out.FragPos = vec3(model * vec4(aPos, 1.0));
    vs_out.Normal = transpose(inverse(mat3(model))) * aNormal;
    vs_out.TexCoords = aTexCoords;
    vs_out.FragPosLightSpace = lightSpaceMatrix * vec4(vs_out.FragPos, 1.0);
    gl_Position = projection * view * vec4(vs_out.FragPos, 1.0);
}
```
2. Fragment Shader:
The code to check if a fragment is in shadow is (quite obviously) executed in the fragment shader.
```GLSL
#version 330 core
out vec4 FragColor;

in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace;
} fs_in;

uniform sampler2D diffuseTexture;
uniform sampler2D shadowMap;

uniform vec3 lightPos;
uniform vec3 viewPos;

// Compute a coefficient of shadow here
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // perform perspective divide
    // As the clip-space FragPosLightSpace is not passed to the fragment shader through gl_Position, we have to do this perspective divide ourselves
    // This returns the fragment's light-space position in the range [-1,1].
    // P.S. When using an orthographic projection matrix the w component of a vertex remains untouched so this step is actually quite meaningless. 
    // However, it is necessary when using perspective projection so keeping this line ensures it works with both projection matrices.
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    
    
    // transform to [0,1] range
    // Because the depth from the depth map is in the range [0,1] and we also want to use projCoords to sample from the depth map, we transform the NDC coordinates to the range [0,1]
    projCoords = projCoords * 0.5 + 0.5;
    
    
    // We can sample the depth map as the resulting [0,1] coordinates from projCoords directly correspond to the transformed NDC coordinates from the first render pass. 
    // This gives us the closest depth from the light's point of view
    // get closest depth value from light's perspective (using [0,1] range fragPosLight as coords)
    float closestDepth = texture(shadowMap, projCoords.xy).r; 
    
    
    // To get the current depth at this fragment we simply retrieve the projected vector's z coordinate which equals the depth of this fragment from the light's perspective.
    // get depth of current fragment from light's perspective
    float currentDepth = projCoords.z;
    
    
    // The actual comparison is then simply a check whether currentDepth is higher than closestDepth and if so, the fragment is in shadow
    // check whether current frag pos is in shadow
    float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;

    return shadow;
}

void main()
{           
    vec3 color = texture(diffuseTexture, fs_in.TexCoords).rgb;
    vec3 normal = normalize(fs_in.Normal);
    vec3 lightColor = vec3(1.0);
    // Ambient
    vec3 ambient = 0.15 * color;
    // Diffuse
    vec3 lightDir = normalize(lightPos - fs_in.FragPos);
    float diff = max(dot(lightDir, normal), 0.0);
    vec3 diffuse = diff * lightColor;
    // Specular
    vec3 viewDir = normalize(viewPos - fs_in.FragPos);
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = 0.0;
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    spec = pow(max(dot(normal, halfwayDir), 0.0), 64.0);
    vec3 specular = spec * lightColor;    
    // Compute Shadow(If in shadow = 1, else = 0)
    float shadow = ShadowCalculation(fs_in.FragPosLightSpace);       
    vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;    

    FragColor = vec4(lighting, 1.0f);
}
```

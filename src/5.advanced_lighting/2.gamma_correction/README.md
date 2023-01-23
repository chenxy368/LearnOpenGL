# Gamma Correction
# Introduction
In the old days of digital imaging most monitors were cathode-ray tube (CRT) monitors. 

For different monitors, they have their own gamma value. Based on the formula: brightness = voltage^gamma, the curve between brightness and voltage is shown in the below graph.

![image](https://user-images.githubusercontent.com/98029669/214074548-0eec2695-52f1-4332-8c54-ff5219f6578c.png)

Ideally, if the gamma equals to 1, the relationship between brightness and voltage is linear. However, it's almost impossible. In the image, with a 2.2 gamma, the 
output can be generally darker than anticipation. On the contrary with a 0.45 gamme, it turns out to be brighter. If we apply a 0.45 gamma correction on a 2.2 gamma
monitor, we can correct the brightness, which is how gamma correction works.

There is another interesting fact. Human's percept of brightness does not hold a linear relationship with the change of physical brightness.  
![image](https://user-images.githubusercontent.com/98029669/214075958-5946e3b9-b30f-45aa-a2f6-d593bf10bcb2.png)  
However, when we're talking about the physical brightness of light e.g. amount of photons leaving a light source, 
the bottom scale actually displays the correct brightness. If we want to let human feel twice bright, we need to multiply 2 to the original lighting brightness, which 
means from 0.1 to 0.2 or 0.4 to 0.8 etc.  

Because the human eyes prefer to see brightness colors according to the top scale, monitors (still today) use a power relationship for displaying output colors 
so that the original physical brightness colors are mapped to the non-linear brightness colors in the top scale.

But when it comes to rendering graphics there is one issue: __all the color and brightness options we configure__(when design) in our applications are based on 
what we perceive from the monitor and thus all the options are actually non-linear brightness/color options. (Our working platform treat every thing as physical brightness
which is weird based on our perception of monitors)

For instance, take a light's color vector (0.5, 0.0, 0.0) which represents a semi-dark red light. 
If we would double this light in linear space it would become (1.0, 0.0, 0.0) as you can see in the graph. 
However, the original color gets displayed on the monitor as (0.218, 0.0, 0.0) as you can see from the graph. 
Here's where the issues start to rise: __once we double the dark-red light__ in linear space, it actually becomes __more than 4.5 times as bright__ on the monitor!

## Gamma Correction
The idea of gamma correction is to apply the inverse of the monitor's gamma to the final output color before displaying to the monitor.  
![image](https://user-images.githubusercontent.com/98029669/214074548-0eec2695-52f1-4332-8c54-ff5219f6578c.png)  
Let's give another example. Say we again have the dark-red color (0.5,0.0,0.0). 
Before displaying this color to the monitor we first apply the gamma correction curve to the color value. 
Linear colors displayed by a monitor are roughly scaled to a power of 2.2 so the inverse requires scaling the colors by a power of 1/2.2.
The gamma-corrected dark-red color thus becomes (0.5,0.0,0.0)^(1/2.2) = (0.5,0.0,0.0) ^ (0.45) = (0.73,0.0,0.0). 
The corrected colors are then fed to the monitor and as a result the color is displayed as (0.73,0.0,0.0) ^ 2.2=(0.5,0.0,0.0).
(All in all, twice light in configuration, twice brightness we can feel.)

P.S. A gamma value of 2.2 is a default gamma value that roughly estimates the average gamma of most displays. 
The color space as a result of this gamma of 2.2 is called the sRGB color space (not 100% exact, but close). 

## OpenGL's built-in sRGB framebuffer 
Enable built-in gamma correction
```C++
glEnable(GL_FRAMEBUFFER_SRGB);
```

## Manually correct in shader
```GLSL
void main()
{
    // do super fancy lighting 
    [...]
    // apply gamma correction
    float gamma = 2.2;
    fragColor.rgb = pow(fragColor.rgb, vec3(1.0/gamma));
}
```

## SRGB Texture
It is possible that texture artist do a gamma correction because they work in SRGB space.  
![image](https://user-images.githubusercontent.com/98029669/214081551-38e01da2-80ca-4dc2-b6eb-be414d36c5e3.png)  
Unfortunately, twice gamma correction are performed. To recover, we can do it manaully. 
```GLSL
float gamma = 2.2;
vec3 diffuseColor = pow(texture(diffuse, texCoords).rgb, vec3(gamma));
```
Or let OpenGL do it for us. Set texture with GL_SRGB or GL_SRGB_ALPHA format.
```C++
glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image);
```

## Attenuation
Physically, the correct lighting attenuation formula is
```GLSL
float attenuation = 1.0 / (distance * distance);
```
In practical, the light attenuate too fast with it. So we use this formula in previous cases.
```GLSL
float distance    = length(light.position - FragPos);
float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));
```
You can also use.
```GLSL
float attenuation = 1.0 / distance;
```

The linear equivalent gives more plausible results compared to its quadratic variant without gamma correction, 
but when we enable gamma correction the linear attenuation looks too weak and the physically correct quadratic attenuation suddenly gives the better results. 
![image](https://user-images.githubusercontent.com/98029669/214083265-95f47d8a-3f44-4f4e-bf78-4616623d9e3f.png)

If we were to use this function without gamma correction, the attenuation function effectively becomes: (1.0/distance^2)^2.2 when displayed on a monitor. 
This creates a much larger attenuation from what we originally anticipated. 
This also explains why the linear equivalent makes much more sense without gamma correction as this effectively 
becomes (1.0/distance)^2.2 = 1.0/distance^2.2 which resembles its physical equivalent a lot more.

P.S. The second one is also good but we need to find apropriate parameter.

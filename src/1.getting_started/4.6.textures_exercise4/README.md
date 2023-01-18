Use a uniform variable as the mix function's third parameter to vary the amount the two textures are visible. 
Use the up and down arrow keys to change how much the container or the smiley face is visible.  
Fragment shader has a uniform about mixValue:
```GLSL
#version 330 core
out vec4 FragColor;

in vec3 ourColor;
in vec2 TexCoord;

uniform float mixValue;

// texture samplers
uniform sampler2D texture1;
uniform sampler2D texture2;

void main()
{
	// linearly interpolate between both textures
	FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), mixValue);
}
```
Change callback function to tune the mixValue
```C++
float mixValue = 0.2f;
...
    ourShader.setFloat("mixValue", mixValue);
...
void processInput(GLFWwindow *window)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);

    if (glfwGetKey(window, GLFW_KEY_UP) == GLFW_PRESS)
    {
        mixValue += 0.001f; // change this value accordingly (might be too slow or too fast based on system hardware)
        if(mixValue >= 1.0f)
            mixValue = 1.0f;
    }
    if (glfwGetKey(window, GLFW_KEY_DOWN) == GLFW_PRESS)
    {
        mixValue -= 0.001f; // change this value accordingly (might be too slow or too fast based on system hardware)
        if (mixValue <= 0.0f)
            mixValue = 0.0f;
    }
}
```

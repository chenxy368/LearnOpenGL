# Creating a window
## Include headfile
```shell
#include <glad/glad.h>
#include <GLFW/glfw3.h>
```

## Callback functions
```shell
void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow *window);

...

void processInput(GLFWwindow *window)
{
    // Press ESC to quit
    if(glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}
void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
    glViewport(0, 0, width, height);
}
```
![glviewport](https://user-images.githubusercontent.com/98029669/212851169-756c7ffb-08c3-4904-8559-b3fcd93637c5.png)

## Initialize glfw
```shell
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

// For Mac OS X
#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif
```

## Initialize window
```shell
// Width, Height, Title, ..., ...
GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
// Check failure
if (window == NULL)
{
    std::cout << "Failed to create GLFW window" << std::endl;
    glfwTerminate();
    return -1;
}
    
// This function makes the OpenGL or OpenGL ES context of the specified window current on the calling thread. 
// A context must only be made current on a single thread at a time and each thread can have only a single current context at a time.
glfwMakeContextCurrent(window);

// Call the callback function when the window size changes
glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);
```

## Load all OpenGL function pointers
```shell
// GLAD manage all OpenGL function pointers
if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
{
    std::cout << "Failed to initialize GLAD" << std::endl;
    return -1;
}
```

## Render loop
```shell
while (!glfwWindowShouldClose(window))
{
    // Call callback function to check inputs
    processInput(window);

    // glfw: swap buffers
    glfwSwapBuffers(window);
    // glfw: poll IO events (keys pressed/released, mouse moved etc.)
    glfwPollEvents();
}
```

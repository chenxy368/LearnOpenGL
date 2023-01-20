# learnopengl.com code repository
Contains code samples for all chapters of Learn OpenGL and [https://learnopengl.com](https://learnopengl.com). 

## My notes
Chapter 1: [Chapter 1](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started)  
1.1 hellow window: create a window [1.1.1(Basic)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/1.1.hello_window) [1.1.2(Remember clear buffer)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/1.2.hello_window_clear)  
1.2 hellow triangle: draw a triangle [1.2.1(Basic)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/2.1.hello_triangle) 
[1.2.2(EBO)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/2.2.hello_triangle_indexed) 
[1.2.3(Exercise1)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/2.3.hello_triangle_exercise1) 
[1.2.4(Exercise2)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/2.4.hello_triangle_exercise2)
[1.2.5(Exercise3)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/2.5.hello_triangle_exercise3)  
1.3 shaders: learn basic shaders [1.3.1(Uniform)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.1.shaders_uniform)
[1.3.2(Add one more vertex attribute and modify shaders)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.2.shaders_interpolation) 
[1.3.3(A class for loading and compile shaders)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.3.shaders_class) 
[1.3.4(Exercise1)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.4.shaders_exercise1)
[1.3.5(Exercise2)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.5.shaders_exercise2)
[1.3.6(Exercise3)](https://github.com/chenxy368/LearnOpenGL/tree/master/src/1.getting_started/3.6.shaders_exercise3)

## Windows building
All relevant libraries are found in /libs and all DLLs found in /dlls (pre-)compiled for Windows. 
The CMake script knows where to find the libraries so just run CMake script and generate project of choice.

Keep in mind the supplied libraries were generated with a specific compiler version which may or may not work on your system (generating a large batch of link errors). In that case it's advised to build the libraries yourself from the source.

## Linux building
First make sure you have CMake, Git, and GCC by typing as root (sudo) `apt-get install g++ cmake git` and then get the required packages:
Using root (sudo) and type `apt-get install libsoil-dev libglm-dev libassimp-dev libglew-dev libglfw3-dev libxinerama-dev libxcursor-dev  libxi-dev libfreetype-dev libgl1-mesa-dev xorg-dev` .
Next, run CMake (preferably CMake-gui). The source directory is LearnOpenGL and specify the build directory as LearnOpenGL/build. Creating the build directory within LearnOpenGL is important for linking to the resource files (it also will be ignored by Git). Hit configure and specify your compiler files (Unix Makefiles are recommended), resolve any missing directories or libraries, and then hit generate. Navigate to the build directory (`cd LearnOpenGL/build`) and type `make` in the terminal. This should generate the executables in the respective chapter folders.

Note that CodeBlocks or other IDEs may have issues running the programs due to problems finding the shader and resource files, however it should still be able to generate the exectuables. To work around this problem it is possible to set an environment variable to tell the tutorials where the resource files can be found. The environment variable is named LOGL_ROOT_PATH and may be set to the path to the root of the LearnOpenGL directory tree. For example:

    `export LOGL_ROOT_PATH=/home/user/tutorials/LearnOpenGL`

Running `ls $LOGL_ROOT_PATH` should list, among other things, this README file and the resources direcory.

## Mac OS X building
Building on Mac OS X is fairly simple:
```
brew install cmake assimp glm glfw freetype
cmake -S . -B build
cmake --build build -j$(sysctl -n hw.logicalcpu)
```
## Create Xcode project on Mac platform
Thanks [@caochao](https://github.com/caochao):
After cloning the repo, go to the root path of the repo, and run the command below:
```
mkdir xcode
cd xcode
cmake -G Xcode ..
```

## Glitter
Polytonic created a project called [Glitter](https://github.com/Polytonic/Glitter) that is a dead-simple boilerplate for OpenGL. 
Everything you need to run a single LearnOpenGL Project (including all libraries) and just that; nothing more. 
Perfect if you want to follow along with the chapters, without the hassle of having to manually compile and link all third party libraries!

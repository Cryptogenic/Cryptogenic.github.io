---
title:  "Writing a Rhythm (GH) Game Engine"
date:   2018-08-02 13:34:00
categories: []
tags: []
---
I've always been an avid player of rhythm games, more specifically, Guitar Hero. While the game is fairly simple in concept, in practice it can be quite challenging, and there's always areas you can improve on. There are always more difficult charts to play, even for players who play on expert difficulty. Unfortunately, around 2010, the franchise was cancelled. With the economy being in the bad state that it was combined with corporate greed, the franchise wasn't making the money it needed to survive (as you can probably tell - I like to pretend Guitar Hero Live doesn't exist).

Then suddenly, in 2017, a clone project started gaining popularity, named "Clone Hero". Clone Hero was essentially Guitar Hero remade with the Unity 3D game engine, using the UI from the last game of the franchise, "Warriors of Rock". What made Clone Hero so awesome was it's support for custom songs - importing songs charted by other people was incredibly easy compared to Guitar Hero III for PC.

The Clone Hero project inspired me to start my own Guitar Hero engine - however unlike Clone Hero, it wouldn't rely on Unity, but the engine would be completely custom. There were a few reasons I had for wanting to create my own engine for this, being:

1. I've wanted to learn about 3D graphics and game engine design for a while
2. Not only would it allow me to learn and gain experience working with low-level game design, but it'd also be a fun project
3. I'd have complete access to source code and such - making it easier to port to other platforms such as the PS4 (at some point I hope)
4. I'd have the ability to make it an open-source, community based project

**Note: I'm not an experienced game developer, as mentioned above, a lot of the reason I wanted to do this project was as a learning experience. Some things I have below may have inefficiencies.**



## Engine Architecture

I decided to use OpenGL 3.3 with C++ to start out with, using the GLFW and GLEW libraries for window management and API respectively. Right away I faced some challenges in designing the engine. I'll detail a few of those challenges below - I may go into more of these challenges in future blog posts :)

The first challenge was trying to figure out how to structure such a massive project. When designing something like a game engine, you want to try to plan out the project's architecture before just jumping into writing code - or your project will likely end up an unmaintainable mess.

To do this we first need to think about what a game engine does. When we think about breaking down a 3D game into it's components, there's a few important things that we need to have.

1. Models. Models are basically an array of vertices for an object (i.e. a cube model will contain 8 vertices).
2. Shaders. There are a few different types of shaders, but the ones I'm mainly talking about are vertex shaders and fragment shaders. A "shader" is basically another name for a "program". A vertex shader runs the given program on each vertex, and a fragment shader runs the given program on each sample. In 4x anti-aliasing, there will be 4 samples per pixel.
3. Scenes. When I refer to scene, I refer to the general environment that the player sees. For a Guitar Hero example, the menu may be one scene, and playing a song in-game would be another scene.

Combining that knowledge with some resources I found from other very helpful websites (such as [IN2GPU](http://in2gpu.com/)), I finally figured out how I wanted to structure my project.

We'll want a model manager - because models are going to be a big part of the engine. The model manager will keep an internal list of all the active models, allow us to easily add new models, and it will be responsible for drawing all the active models.

A shader manager is also going to be another important manager. It will export an interface for us to be able to create shader instances at run-time, and obtain a handle for them to apply to models. 

The scene manager manages the shader and model managers - therefore each scene will have it's own separate model and shader managers. This allows us to isolate the models/shaders to their own scenes. For example, we don't need the fretboard model for the menu scene - only the gameplay scene. The scene that's currently active will use it's model manager to render the screen via GLFW. 

Here's a quick graphic depicting the current layout of the engine.

![](https://i.imgur.com/U5DPxdT.png)

The next challenge was setting up OpenGL with Visual Studio. Setting up the dependencies manually kind of sucks, and it's easy to run into stupid errors by making one tiny mistake. Luckily, I found an amazing article by in2gpu on how to setup Visual Studio using NuGet, a package manager that gets shipped with Visual Studio. Using NuGet is super easy, all I had to do was use the Package Manager Console (Tools -> NuGet Package Manager -> Package Manager Console) and use the following command to install the package:

```
Install-Package nupengl.core
```



## A (small) rant on Visual Studio

Here I'll go into a bit of a rant on Visual Studio - I love it and hate it. Working with Visual Studio is nice - when it actually works. I ran into a lot of stupid, stupid, stupid issues which I'll detail a few of here. Partially because I want to rant about it a bit, partially because my found solutions may benefit others who encounter these issues.

The class generator sucks. For multi-folder projects, don't even bother. Seriously. Right clicking on a folder and using the "Create Class" option will create the class in the main folder rather than the folder you clicked on. Moving these files to the folder I intended seemed to break Intellisense - syntax highlighting was failing and errors weren't getting caught. I had to close and re-open the project to get Intellisense back.

I spent about an hour or two trying to debug a linker error saying that the symbols weren't found for one of my classes that I had in the solution. For some reason, the object (.obj) file for the class wasn't getting built even though it was included in the solution. Rebuilding and cleaning the solution didn't help. The only way to fix this was to remove the source file from the solution and re-add it. Why? Who knows, hopefully this helps others who encounter this issue though.

Over all, MSVC errors/exceptions suck. Instead of getting a "null pointer dereference", you get this thrown in your face:

```
Exception thrown: read access violation.
**std::_Tree_comp_alloc<std::_Tmap_traits<std::basic_string<char,std::char_traits<char>,std::allocator<char> >,Rendering::IGameObject *,std::less<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >,std::allocator<std::pair<std::basic_string<char,std::char_traits<char>,std::allocator<char> > const ,Rendering::IGameObject *> >,0> >::_Myhead**(...) returned 0x4.
```

Looking at this error you'd probably assume the issue was the map of strings to game objects. Rather, it was the object that contained the map, as I forgot to initialize the object properly. A silly error on my part - but this error message was worse than useless.



## Shaders vs. Models Implementation

The shader manager is easier to implement from an architecture stand-point compared to the model manager. The shader manager only needed to provide an interface to add new shaders to a list (given a path to a vertex and fragment shader file as well as a name for the shader), and fetch the handle of existing ones (as well as some cleanup in the destructor to free the resources - but I'll skip that for brevity).

```cpp
map<string, GLuint> ShaderManager::shaders;

void ShaderManager::createShader(const string &name, const string &vertexShaderPath, const string &fragmentShaderPath)
{
	int result = 0;

	string vertexShaderCode = readShader(vertexShaderPath);
	string fragmentShaderCode = readShader(fragmentShaderPath);

	GLuint vertexShader = createShaderInternal(GL_VERTEX_SHADER, vertexShaderCode, string("vertex shader"));
	GLuint fragmentShader = createShaderInternal(GL_FRAGMENT_SHADER, fragmentShaderCode, string("fragment shader"));

	GLuint program = glCreateProgram();

	glAttachShader(program, vertexShader);
	glAttachShader(program, fragmentShader);

	glLinkProgram(program);
	glGetProgramiv(program, GL_LINK_STATUS, &result);

	if (result == GL_FALSE)
	{
		// ...
	}

	shaders[name] = program;
}

const GLuint ShaderManager::getShader(const string &name)
{
	if (shaders.find(name) != shaders.end())
	{
		return shaders.at(name);
	}

	string err = "Could not find shader '" + name + "'!";
	Core::Error(err, Core::ERROR_WARNING);
	return -1;
}

```

The model manager however was not as straight-forward. Different models can have different attributes - and it'd be nice to have a class for common model types (for example, a cube). Later on I'll add other models for things like the notes for songs, however I'm not that far along yet :)

There are a few things that all models will want to do though. For example, all models should have an easy method to set the shader, or have proper clean-up in the destructor. I therefore decided to make a parent class called "Model", and had other classes, such as "Cube", that inherited from that class.

```cpp
// The "model" base class
class Model
{
public:
	// Model() will get overwritten by the inherited class, but ~Model() won't
	Model();
	~Model();

	// draw() and update() get overwritten by the inherited class
	virtual void draw(glm::mat4 view, glm::mat4 projection) = 0;
	virtual void update() = 0;

	void setShader(GLuint shader);

	GLuint getVAO() const;
	const std::vector<GLuint> &getVBOS() const;

protected:
	std::vector<GLuint> VBOS;
	std::vector<GLuint> CBOS;

	GLuint VAO;
	GLuint shader;
};

// The "cube" class
class Cube : public Model
{
public:
	Cube();
	~Cube();

	// Overwriting update() and draw() methods from the Model class
	virtual void update();
	virtual void draw(glm::mat4 view, glm::mat4 projection);
};
```

I decided to handle the construction of the vertex array objects (VAO) and vertex buffer objects (VBO)'s inside the class constructor. For those a bit unfamiliar with graphics lingo - VAO's and VBO's basically hold the vertices that a model has.

```cpp
Cube::Cube()
{
	GLuint vao;
	glGenVertexArrays(1, &vao);
	glBindVertexArray(vao);

	static const GLfloat g_vertex_buffer_data[] = {
		// ...
	};

	GLuint vbo;
	glGenBuffers(1, &vbo);
	glBindBuffer(GL_ARRAY_BUFFER, vbo);
	glBufferData(GL_ARRAY_BUFFER, sizeof(g_vertex_buffer_data), g_vertex_buffer_data, GL_STATIC_DRAW);

	this->VAO = vao;
	this->VBOS.push_back(vbo);
}
```

Since the destructor wasn't really specific to any type of model, I left it as the generic one defined in `Model.cpp`.

```cpp
Model::~Model()
{
	// Delete Vertex Array Objects and Vertex Buffer Objects
	glDeleteVertexArrays(1, &VAO);
	glDeleteBuffers(VBOS.size(), &VBOS[0]);

	VBOS.clear();
}
```



## Drawing Models

In the above section I created the vertex arrays necessary for drawing a model - but that code doesn't actually *tell* OpenGL to draw the model - the class contains a separate method for that called `draw()`. Each object's `draw()` method is called by the scene manager in a loop. Something to keep in mind in the future is that this method should be quick. Any inefficiencies in the draw code can cause serious performance issues if there's many models that are using that inefficient model's draw code at the same time.

The draw method though is fairly simple. We simply apply the shader and setup a few API calls to send our array off to OpenGL to get drawn.

```cpp
void Cube::draw(/* */)
	// ...

	/*
		We want to optimize performance by minimizing GL calls, so we'll create models in a special way that allows us to cut down our usage of the glUseProgram() call. By creating objects that will use the same shader in sequence, we can use just one call to glUseProgram().
	*/
	if (!Managers::SceneManager::checkShaderInUse(this->shader))
	{
		glUseProgram(this->shader);
		Managers::SceneManager::setShaderInUse(this->shader);
	}

	// ...

	glEnableVertexAttribArray(0);
	glBindBuffer(GL_ARRAY_BUFFER, this->VBOS[0]);
	glVertexAttribPointer(
		0,                  // attribute
		3,                  // size
		GL_FLOAT,           // type
		GL_FALSE,           // normalized
		0,                  // stride
		(void*)0            // array buffer offset
	);

	// ...

	// Draw the triangles
	glDrawArrays(GL_TRIANGLES, 0, 3*12); // 2 triangles per face (2x6 = 12), 3 vertices per triangle (3x12)

	glDisableVertexAttribArray(0);
```

As you can see, I have a small optimization in here already. OpenGL calls can be expensive, so minimizing the amount of calls made by something like a model's draw function which is called constantly can be a useful optimization. If I only call `glUseProgram()` when I actually need to tell OpenGL to change the shader it's rendering with, then we'll have a minimal amount of API calls. The way I'm planning on doing this is by essentially creating objects that are going to have the same shader in sequence. This way, the `glUseProgram()` call will be skipped since the previous model that was being drawn already used that shader.

Each scene manager will have it's own static member declaring the shader that's "in use", and will provide methods to check if `glUseProgram()` needs to be called.

```cpp
class SceneManager
{
public:
	SceneManager();
	~SceneManager();

	// Generic callbacks
	void draw();

	// ...

	static bool checkShaderInUse(GLuint shader);
	static void setShaderInUse(GLuint shader);
private:
	static GLuint shaderInUse;

	ShaderManager *shaderManager;
	ModelManager *modelManager;

	// ...
};
```



## Final Thoughts

As of the writing of this article, I currently have the following:

![](https://i.imgur.com/JPoiFeN.png)

As you can see, nothing very special yet. My next task is to work on the camera, as well as adding multiple models into the scene and moving them relative to the camera. This involves some fairly interesting mathematics involving Matrices, which I hope to go into in a future blog post.

The engine isn't currently open source as there isn't much to it yet, however I hope to make it open source at a later date.

This isn't perfect - and I don't expect it to be. As mentioned earlier in the post, I'm far from a professional game developer. The nice thing is though, I don't (currently) think that game engine inefficiencies will be incredibly bad for a simpler game like Guitar Hero compared to an FPS. Throughout this project I'll do my best to write the engine as efficient as possible, but I know that it won't be as it's my first time writing anything like this.

That being said, if you're an experienced game developer and have any tips, I'd really appreciate it if you could shoot me a tweet or an email, both of which can be found via my blog's front page :)

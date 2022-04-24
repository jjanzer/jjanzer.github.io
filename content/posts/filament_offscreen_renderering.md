---
title: "Rendering Offscreen with Filament using Render Targets"
date: 2021-10-06T22:13:11-06:00
draft: false
tags: ['3d']
---

I was making a small application for viewing 3D asets and wanted to use [Google's Filament](https://github.com/google/filament) renderer. It's not too hard to get started with this, but I was already using my own OpenGL context and didn't want to yield full control over to Filament. Instead I wanted to render the model on an offscreen buffer and then display it on my main window. Surprisingly this is quite difficult to figure out how and the documentation and examples don't cover this case well. I did find some helpful discussion but they didn't deliver the full result I was looking for.

In short, the method below describes how to use Filament to draw into a render target and then render that same target on your own. The technologies I'm using are `OpenGL`, `SDL`, `Dear ImGui`, and `Filament`. I hope this is helpful to anyone else doing something similar as I was a bit stumped until I figured it out.

*Note* currently rendering this way is only supported when using OpenGL.


## Step 1 - Initialize OpenGL Contexts

For the most part you will create all of your SDL window handling as typical, of importance is obtaining the OpenGL context such as

~~~ C++
SDL_Window* window = NULL;
// ... setup your gl attributes and flags ...
SDL_GLContext gl_context = SDL_GL_CreateContext(window);

// Ensure current context is null
SDL_GL_MakeCurrent(window, nullptr);

// Create Filament engine using context
Engine* engine = Engine::create(Engine::Backend::OPENGL,nullptr,gl_context);

// Now that engine is created we can make the context current
SDL_GL_MakeCurrent(window, gl_context);
~~~

Notice that we ensure the current context is null before initializing filament's engine then set it after. This is very important or the Engine::create call will fail.

## Step 2 - Setup OpenGL Buffers

Now we need to create the buffers for rendering

~~~ C++
// square size of our render target texture
const uint32_t rt_dim = 512;
// rgba
const uint32_t rt_channels = 4;

// Allocate pixel data
GLubyte* pixels = new GLubyte[rt_dim*rt_dim * rt_channels];

// set pixel data for green and alpha channels to 100% (255) using 4 bytes per pixel
// you can skip this step if you don't want to flood fill the base texture
// I set it to green for debugging
for(int i=0;i<rt_dim*rt_dim;i++)
{
	int slot = i*rt_channels;
	for(int j=0;j<rt_channels;j++){
		pixels[slot+j] = 0;
		//green or alpha
		if(j == 1 || j == 3){
			pixels[slot+j] = 255;
		}
	}
}

// make a texture
GLuint tex_gl;
glGenTextures(1, &tex_gl);
glBindTexture(GL_TEXTURE_2D, tex_gl);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
#if defined(GL_UNPACK_ROW_LENGTH) && !defined(__EMSCRIPTEN__)
glPixelStorei(GL_UNPACK_ROW_LENGTH, 0);
#endif
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, rt_dim, rt_dim, 0, GL_RGBA, GL_UNSIGNED_BYTE, pixels);

// We don't need this array anymore so we can safely release it now
delete[] pixels;
~~~

## Step 3 - Setup Filament Buffers and Render Targets

Now we need to tell Filament to use our OpenGL buffer instead of it's own, we can do that when creating a texture by using `import`.

~~~ C++
// Create filament textures and render targets (note the color buffer has the import call)
auto rt_color = filament::Texture::Builder()
	.width(rt_dim)
	.height(rt_dim)
	.levels(1)
	.usage(filament::Texture::Usage::COLOR_ATTACHMENT | filament::Texture::Usage::SAMPLEABLE)
	.format(filament::Texture::InternalFormat::RGBA8)
	.import(tex_gl)
	.build(*engine);
auto rt_depth = filament::Texture::Builder()
	.width(rt_dim)
	.height(rt_dim)
	.levels(1)
	.usage(filament::Texture::Usage::DEPTH_ATTACHMENT)
	.format(filament::Texture::InternalFormat::DEPTH24)
	.build(*engine);
auto rt = filament::RenderTarget::Builder()
	.texture(RenderTarget::AttachmentPoint::COLOR, rt_color)
	.texture(RenderTarget::AttachmentPoint::DEPTH, rt_depth)
	.build(*engine);

// Make a specific viewport just for our render target
Viewport rt_vp({0, 0, rt_dim, rt_dim});
view_rt->setRenderTarget(rt);
view_rt->setViewport(rt_vp)
~~~

##  Step 4 - Render Loop

Now we just need to setup a render pass in our draw loop, this can be done with SDL or ImGui.

~~~ C++
while(run)
{
	// ... snipped check for quit/escape/etc ...

	// Start the Dear ImGui frame
	ImGui_ImplOpenGL3_NewFrame();
	ImGui_ImplSDL2_NewFrame();
	ImGui::NewFrame();

	//Render in Filament and wait for the results (we could do this async if desired)
	if (renderer->beginFrame(swapChain_rt)) {
		renderer->render(view_rt);
		renderer->endFrame();
		engine->flushAndWait();
	}

	// ... snipped handle other imgui/sdl drawing ...

	// Draw our texture using ImGui
	ImGui::Image((void*)(intptr_t)tex_gl, ImVec2(rt_dim, rt_dim));

	// handle close of ImGui render and window buffer swapping
	ImGui::Render();
	glViewport(0, 0, (int)io.DisplaySize.x, (int)io.DisplaySize.y);
	glClearColor(clear_color.x * clear_color.w, clear_color.y * clear_color.w, clear_color.z * clear_color.w, clear_color.w);
	glClear(GL_COLOR_BUFFER_BIT);
	ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
	SDL_GL_SwapWindow(window);
}

// don't forget to cleanup our texture
glDeleteTextures(1,&tex_gl);
~~~

If you're successful you'll end up with something similar to this image where I am rendering a triangle in Filament but displaying the texture via Dear ImGui.

![Rendering a Triangle to a Render Target](https://i.imgur.com/M7dgdGf.png)


## Reading Buffer Back To CPU

If we want we can read the buffer back into cpu land like so:

~~~ C++

std::vector<uint8_t> raw_buffer;
uint32_t pixels_w = rt_dim; //w;
uint32_t pixels_h = rt_dim; //h;
raw_buffer.resize(pixels_w*pixels_h * 4);
filament::backend::PixelBufferDescriptor bd(raw_buffer.data(), raw_buffer.size(), backend::PixelDataFormat::RGBA, backend::PixelDataType::UBYTE);

//after rendering to the target we can now read it back
renderer->readPixels(rt,0,0,pixels_w,pixels_h,std::move(bd));

//write it to disk
std::string path = "./out.raw";
std::ofstream stream(path, std::ios::out | std::ios::binary);
if(!stream.good())
{
	std::cerr << "Failed to open: " << path << std::endl;
}
else {
	stream.write((char*)&raw_buffer[0], raw_buffer.size());
	stream.close();
}
~~~

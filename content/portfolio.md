---
title: "Jesse Janzer's Portfolio"
draft: false
---


<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Caveat:wght@400..700&display=swap" rel="stylesheet">

<style type="text/css">

	.portfolio-intro {
		margin: 0 0 0 10px;
	}

	.portfolio-item {
		margin: 3em 0;
	}

	.portfolio-item-title, .portfolio-item-description, .portfolio-item-tech {
		margin: 0 0 0 10px;
	}

	.portfolio-item-title {
		font-size: 1.25em;
	}
	.portfolio-item-description {
		color: #555;
	}
	.portfolio-item-tech {
		margin-top: 1em;
		color: #777;
		font-style: italic;
	}

	.portfolio-item-images {
		display: grid;
		grid-template-columns: 33% 33% 33%;
		gap: 10px;
	}

	.portfolio-item-images img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		z-index: -1;
		position: relative;
	}
	.portfolio-item-images-image {
		/*
		display: inline-block;
		vertical-align: middle;
		width: 40vh;
		height: 40vh;
		margin: 10px;
		*/
		border: solid 10px rgba(255,255,255,1);
		border-bottom-width: 80px;
		box-shadow: 3px 3px 6px 0 rgba(0, 0, 0, 0.2),
			inset 3px 3px 6px 0 rgba(0, 0, 0, 0.2);
		object-fit: cover;
	}

	.portfolio-item-images-image-text {
		font-family: "Caveat", cursive;
		font-weight: 400;
		font-style: normal;
		font-size: 14pt;
		text-align: center;
		text-wrap: balance;
		margin-top: 10px;
	}
</style>

<div class="portfolio">
	<div class="portfolio-intro">I enjoy working with mixed technology and different types of tooling. Below contains a few of these projects that I can talk about publicly. You can see additional work on my <a href="https://www.youtube.com/channel/UCVBljercTqd_fWrKzN1mGFQ">YouTube Channel</a> or my <a href="https://github.com/jjanzer">GitHub</a>.</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Distortion Correction and Relaxer</div>
		<div class="portfolio-item-description">Wrote a geometry and UV relaxer in C++ that was successfully incorporated into the real-time avatar system at Daz/Tafi. This was used to correct for distortion from the projection system used to refit follower assets like clothing when morphed on an avatar. The images below are slowed down to show the iteration steps for visualization purposes.</div>
		<div class="portfolio-item-tech">C++, C#</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/bnG7dOo.gif"/><div class="portfolio-item-images-image-text">Smooths curvature of the morph while preserving volume along with correcting UVs.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/Uxpc8Zx.gif"/><div class="portfolio-item-images-image-text">Example of a gas mask when morphed from a human to alien face.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/Ec7EZn3.gif"/><div class="portfolio-item-images-image-text">Heavy distortion when refitting to a very different character being corrected.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Tetrahedronal Improvements to Projection</div>
		<div class="portfolio-item-description">Improved the results of mesh distortion when projecting by using tetrahedronal mapping instead of the previous methods. Greatly improves results with rotations and twist movements.</div>
		<div class="portfolio-item-tech">C++, C#</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/EsCVGmw.gif"/><div class="portfolio-item-images-image-text">Demonstrating the improvements to rotation quality.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/Cu1A7SF.gif"/><div class="portfolio-item-images-image-text">Toggling the tetrahedronal mapping system on-off to visualize twist movements.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/sBfAYHY.gif"/><div class="portfolio-item-images-image-text">Reconstruction of the tetrahedronal mapping to visualize how it works.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Real-time Avatar System</div>
		<div class="portfolio-item-description">Created a high performance, real-time, engine agnostic avatar system called <b>Astra</b>. Supported real-time projection of follower assets that dynamically reformed clothing, hair, and accessories to unseen shapes allowing characters to morph and adapt. The system was split into two parts, an engine agnostic dynamic library in C++, and a per-engine, Unity and Unreal, plugin. Assets would stream down from a network on demand and be loaded procedurally into the engine at run-time. Aggressively optimized for performance in categories such as latency, cpu and battery usage, rendering, memory, disk usage, and others. Supported all major gpu compression and optimal disk and stream compressions. Full makeup, cloth layering system, fitment, decimation, and smart asset replacement. Was used to power the Tafi VRChat Avatar Creator. The library was cross-compiled and ran on ARM and x86 on Windows, macOS, Linux, iOS, tvOS, and Android.</div>
		<div class="portfolio-item-tech">C++, C#, OpenGL, Unity, Unreal, Shaders, Go, Google Cloud, e-commerce, Linux, macOS, Windows, iOS, Android, tvOS</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://imgur.com/ZWLII89.gif"/><div class="portfolio-item-images-image-text">Examples of character combinations you could make.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/sh35hpq.gif" style="object-fit: contain;"/><div class="portfolio-item-images-image-text">Smart clothing content.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/GQFUwFP.gif"/><div class="portfolio-item-images-image-text">Poke-through system.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/IMMGUZ0.gif"/><div class="portfolio-item-images-image-text">Real-time cloth and material morphing.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/sdVk9f5.gif"/><div class="portfolio-item-images-image-text">Included a full avatar creator kit that worked on mobile.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/yXnjzsJ.gif"/><div class="portfolio-item-images-image-text">Powered the Tafi VRChat Avatar System.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Custom Ray Tracer</div>
		<div class="portfolio-item-description">Developed a headless custom ray tracer using C++. This was then used to calculate occlusion masks for hiding surfaces under other surfaces and detecting collisions of geometry.</div>
		<div class="portfolio-item-tech">C++, OpenGL, eigenvector, GPGPU, SIMD, intrinsics</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://imgur.com/393T63W.gif"/><div class="portfolio-item-images-image-text">Without normal map.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/aW0Udqc.gif"/><div class="portfolio-item-images-image-text">Producing a depth map.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/rkoqy8t.gif"/><div class="portfolio-item-images-image-text">Visualizing normals.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Raw Pixel Streaming From Desktop to Web</div>
		<div class="portfolio-item-description">Created a C++ plugin that captured the existing OpenGL viewport buffer inside a 3D software application, Daz Studio, and sent it to a server where it was transformed into a low latency feed with MPEG-TS. The feed was then shown in a remote browser. This was successfully used to build a prototype that ran Daz Studio on an array of machines and let the user control it remotely from a browser.</div>
		<div class="portfolio-item-tech">C++, Python, JavaScript, OpenGL, Qt, WebSockets, FFmpeg</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/Vz0MJj6.gif"/><div class="portfolio-item-images-image-text">Captured OpenGL viewport is streamed across the network to a browser.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Real-time Eye Shader</div>
		<div class="portfolio-item-description">Custom and controllable shader for rendering realistic eyes with modifiable parameters.</div>
		<div class="portfolio-item-tech">C#, HLSL, GLSL, Unity</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/VGXDyrv.gif"/><div class="portfolio-item-images-image-text">Pupil dilation controls.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/UDWs2Po.gif"/><div class="portfolio-item-images-image-text">Turntable showing anatomically accurate geometry and accurate refraction through the cornea and iris.</div></div>
			<div class="portfolio-item-images-image"><img src="https://i.imgur.com/2IXAhkf.gif"/><div class="portfolio-item-images-image-text">Iris shape controlled with a cookie.</div></div>
		</div>
	</div>
	<div class="portfolio-item">
		<div class="portfolio-item-title">Photography Website</div>
		<div class="portfolio-item-description">Handwritten highly dynamic database backed website that handles over 1000 active pages for a high traffic photography portal. Aggressively optimized for SEO and consistently ranks in the top 3 in their space for over 15 years. The tooling allows the photographer to self manage all aspects of the website including adding new domains. Created the site template and designs along with all custom code. Templates could nest other templates and exposed as easy to use GUI interface to the user.</div>
		<div class="portfolio-item-tech">MySQL, PHP, HTML, CSS, JavaScript, SCSS, Bootstrap, Webpack, AWS</div>
		<div class="portfolio-item-images">
			<div class="portfolio-item-images-image"><img src="https://imgur.com/4xIZfMY.gif"/><div class="portfolio-item-images-image-text">Handwritten dynamic and best fit block arranging system.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/cY5AAX9.png"/><div class="portfolio-item-images-image-text">Allows multiple domains and templating to be handled by the owner.</div></div>
			<div class="portfolio-item-images-image"><img src="https://imgur.com/q93Vw4j.png"/><div class="portfolio-item-images-image-text">Page template system allowing the photographer to create new pages and get access to custom templated elements.</div></div>
			<!--<div class="portfolio-item-images-image"><img src="https://imgur.com/2L0KSOY.png"/><div class="portfolio-item-images-image-text">Dynamic elements support nested structures and sub-linking of other templates.</div></div>-->
		</div>
	</div>
</div>
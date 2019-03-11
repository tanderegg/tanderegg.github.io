---
layout: post
title: Building a Platformer on Amethyst - Part 1
---

# Introduction

Like many developers, I got my start out of an interest in game development.  I first began to understand what goes into making a game during a computer class where we used the [Logo](https://en.wikipedia.org/wiki/Logo_(programming_language)) language, and from there began to try to make my own Zelda-like game.

Fast forward a few decades, and I'm now a professional developer, but not in the game industry.  I've kept up my interest in game programming partly for fun, but also as a way to motivate me to learn new technologies and languages.  As such, I've never really made a full game, but I've developed a number of game engines first using C, then C++, and more recently Rust.

Rust, for those who aren't familiar, is a new programming language designed from the ground up to address some of the major deficencies in languages like C/C++, namely memory safety.  This turns out to be a very important aspect of game programming, as games tend to tax a computer more than any other type of software, and attempt to eek as much efficiency as possible from a computer's RAM and CPU cycles.  This leads to fairly low-level programming and memory management, which in turn increases the risk of memory corruption, stack overflows, and null pointer exceptions.

Rust aims to be just as performant as C/C++, but with memory safety baked in to the compiler, you can experiment and "fail fast" while developing without fear of hidden bugs wreaking havoc later.  Well, that's the theory at most, let's see how it goes!

# Amethyst and Rust for game development

[Amethyst](https://amethyst.rs) is a game development framework written in Rust that is based on the Enitity, Component, and Systems (ECS) model of game development.  For more background on that, check out [@kyren's closing keynote for Rustconf 2018](https://kyren.github.io/2018/09/14/rustconf-talk.html).  Her experience is much like mine has been, trying to work with C++ OOM models for game development.  ECS and Rust go hand and hand, and since I've spent a fair amount of time writing (and-rewriting) game engines, and less so actual games, I thought I start with Amethyst and work directly on developing a game within that framework.

# Part 1: Drawing things

## Amethyst bootstrap and folder structure

To get started, first install Rust using [Rustup](https://rustup.rs/) if you haven't already.  Then, install Amethyst's CLI tools with cargo: `cargo install amethyst-tools`.  Finally, type `amethyst new quartz-knight` to create a new folder called "quartz-knight" with a few skeleton files in it.

The folder structure should look like this:

```
quartz-knight ->
  resources ->
    display_config.ron
  src ->
    main.rs
  Cargo.toml
```

At this point, you should be able to run your game already, first `cd quartz-knight` then `cargo run`, and after a fair amount of compiling you should get a blank screen, woo hoo!

## Set the stage with a background image

OK, let's start filling in that blank screen.  First, let's pretend we are targeting a 720p resolution (say, Nintendo Switch in hand-held mode), and update `display_config.ron` like so:

```
(
  title: "Quartz Knight",
  dimensions: Some((1280, 720)),
  max_dimensions: None,
  min_dimensions: None,
  fullscreen: false,
  multisampling: 1,
  visibility: true,
  vsync: true,
)
```

This will make the window title look nicer, and set the window size to 1280x720.  If you `cargo run` again, you should see a bigger window.

Next, create a new folder called `assets` in your project root, then download [this file](https://github.com/tanderegg/amethyst-platformer/blob/part_1/quartz_knight/assets/background.png) to it.  That will be our background image to get started with.

To load it, we need to start writing some code.  First, let's rename the default game state struct to something better than `Example`, and add some startup code to the impl:

```
struct PlayState;

impl SimpleState for PlayState {
    fn on_start(&mut self, data: StateData<'_, GameData<'_, '_>>) {
        let world = data.world;
        let texture_handle = load_texture(world, "background.png");
        let _image = init_image(world, &texture_handle);
        init_camera(world);
    }
}
```

In Amethyst, `SimpleState` is a good trait to get started with, as it implements the basic state management plumbing for you.  Here we are adding a few calls to helper functions that will load the PNG file as a texture, create an image Entity using that texture, and initializing the camera.

### load_texture

```
fn load_texture(world: &mut World, png_path: &str) -> TextureHandle {
    let loader = world.read_resource::<Loader>();
    let texture_storage = world.read_resource::<AssetStorage<Texture>>();
    loader.load(
        png_path,
        PngFormat,
        TextureMetadata::srgb_scale(),
        (),
        &texture_storage,
    )
}
```

The `world` object in Amethyst is the central gateway to all entities and resources used by your game, so to load the texture we first ask it for a resource loder.  A Loader takes a resource of a given format, in this case PngFormat provided by Amethyst, and a storage, then does the loading for you.  Here we are using Amethyst's AssetStorage to store a Texture, and we provide it with the needed TextureMetadata, a default provided by Amethyst for standard color images.

### init_image

```
fn init_image(world: &mut World, texture: &TextureHandle) -> Entity {
    let mut transform = Transform::default();
    transform.set_x(0.0);
    transform.set_y(0.0);
    transform.set_scale(0.6667, 0.667, 0.0);

    world
        .create_entity()
        .with(transform)
        .with(texture.clone())
        .build()
}
```

Time to build our first entity!  Recall that in ECS, and entity is little more than an ID, and is made up of a number of Components associated with that ID.  For now, our simple background image is made up of just two components, a texture and a transform.  In Amethyst, and game programming in general, an entity's position, orientation, and scale are all contained in a 4x4 matrix.  We don't have to deal with the details of that though, we can just set our X and Y coordinates to 0, and scale the size to match our resolution.  Since the image is 1080p, we scale by 2/3 to get to 720p.

### init_camera

```
fn init_camera(world: &mut World) {
    let mut transform = Transform::default();
    transform.set_z(1.0);
    world
        .create_entity()
        .with(Camera::from(Projection::orthographic(
            -640.0, 640.0, -360.0, 360.0
        )))
        .with(transform)
        .build();
}
```

Our camera, like most everything else, is an entity, this time with a Transform component as before, and a Camera component.  Since we are in 2D land, we can just use an orthographic camera without perspective (so Z can be anything, 1.0 works fine), and we can set the viewport size to match our resolution.  This results in our coordinate grid aligning with pixels, i.e moving an entity's X coordinate + 1 will move that entity to the right one pixel.

### Main

Lastly, to draw our background on the screen we need to update `main.rs` as follows:

```
amethyst::start_logger(Default::default());

    // 1. Setup paths
    let app_root = application_root_dir();
    let assets_path = format!("{}/assets", app_root);

    // 2. Setup rendering
    let display_config = DisplayConfig::load(format!("{}/resources/display_config.ron", app_root));
    let pipe = Pipeline::build().with_stage(
        Stage::with_backbuffer()
            .clear_target([0., 0., 0., 1.0], 1.0)
            .with_pass(DrawFlat2D::new())
    );

    // 3. Build our Game Data
    let game_data =
        GameDataBuilder::default()
            .with_bundle(
                TransformBundle::new()
            )?
            .with_bundle(
                RenderBundle::new(pipe, Some(display_config))
            )?;

    // Build and run
    let mut game = Application::new(assets_path, PlayState, game_data)?;
    game.run();
    Ok(())
```

Let's break this down step by step:

#### 1. Setup the Asset Loader

Here, we are just telling the Amethyst `Application` where the root of our assets folder is.  This is passed to `Application::new` at the end of `main`.  Without this, the Amethyst Loader class used in `load_texture` would not know where to look for the files, and we would have to build absolute paths instead.

#### 2. Setup rendering

Next, we need to setup the render pipeline.  Most of this code was provided by the bootstraper, but we're making one change: swap out DisplayFlat for DisplayFlat2D::new(), which is a convenience that provides basically the same functionality: a default 2D render pass.

#### 3. Build Game Data

Lastly, we just need to add a System that handles our transforms.  Right now it doesn't do much, so just include a default TransformBundle::new() which will register the Transform Component with the ECS engine.

OK, now `cargo run` and you should see your screen, but this time with a nice pretty / scary background image!

![background screenshot](https://tanderegg.github.io/images/quartz_knight_part1a.png)

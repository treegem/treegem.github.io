---
layout: post
title: "Hello, (Gaming) World!"
excerpt: "Equipped with the basics, I wanted to create a small game as a finger exercise on my own. In programming
languages, it's a common classic to output \"Hello, World!\" as a starting point. I think creating Pong is a suitable
equivalent, like a \"Hello, Gaming World!\""
image: "/assets/images/2023-03/dodge_preview.gif"
card: "summary"
toc: true
---

Equipped with the basics, I wanted to create a small game as a finger exercise on my own. In programming languages, it's
a common classic to output "Hello, World!" as a starting point. I think
creating [Pong](https://de.wikipedia.org/wiki/Pong) is a suitable equivalent, like a "Hello, Gaming World!". The code of
the first
version can be found [in my repository](https://github.com/treegem/SweetPong/tree/hello_gaming_world). Just
as [the current version](https://github.com/treegem/SweetPong).

<div class="centered-container">
  <video width="600" loop autoplay muted>
    <source src="{{site.baseurl}}/assets/videos/2023-03/pong_vanilla_gameplay.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <figcaption>This is the first version of my attempt of creating a Pong game on my own: Battle Pong</figcaption>
</div>

For my initial attempt, my goal was to create a playable version of Pong with a minimum feature set of:

- a human player can play against a computer enemy
- a minimal menu
- a few game sounds
- score would be tracked
- game ends when a certain score was reached, with the option to start a new game

## Main Scene

The main scene's current structure uses a simple Node to contain the world, UI, sounds, ball, and both the player and
enemy paddles. This Node serves as a central container for all the game elements. With the exception of
the `SoundService`, all other child nodes themselves serve as containers for additional nodes. However, as these were
saved as [Packed Scenes](https://docs.godotengine.org/en/stable/classes/class_packedscene.html) and added here as child
nodes, they function as black boxes within the context of the main node. By concealing the internal contents of Packed
Scenes when adding them as child nodes, it encourages giving the Packed Scenes a clear and well-defined API that can be
accessed externally. This approach greatly facilitates the development of a modular software design.

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-03/main_scene_structure.png">
        <figcaption>Main scene node structure. </figcaption>
    </figure>   
</div>

## The Paddles

During development, I started with the `PlayerPaddle` because it was already clear to me which nodes it had to be
composed of:

- `CharacterBody2D` for the built-in move_and_collide method
- `Sprite2D` to make the paddle visible
- `CollisionShape2D` to detect collisions with the ceiling/floor and the ball automatically.

To standardize the movement of both `PlayerPaddle` and `EnemyPaddle`, I created a parent class called Paddle that
inherits from `CharacterBody2D`, and manages the movement based on an input vector. Initially, the controls felt very
clunky to me. After a brief research, I found that adding acceleration instead of immediate maximum speed feels much
more natural. The complete code for the Paddle parent class is:

    extends CharacterBody2D
    
    class_name Paddle
    
    const MAX_SPEED = 700
    const ACCELERATION = 4000
    
    func move(input: Vector2, delta: float):
        velocity = velocity.move_toward(input * MAX_SPEED, ACCELERATION * delta)
        move_and_collide(velocity * delta)

The move function requires a two-dimensional `input` vector and a `delta`, which represents the time since the last
physics
frame. Multiplying them ensures that approximately the same distance is covered per unit time in case of frame drops.
For the `PlayerPaddle`, extending the `Paddle` class, the two parameters are determined as follows:

    func _physics_process(delta):
        var input_vector: Vector2
    
        if Input.is_action_pressed("ui_up"):
            input_vector = Vector2.UP
        elif Input.is_action_pressed("ui_down"):
            input_vector = Vector2.DOWN
        else:
            input_vector = Vector2.ZERO
        
        move(input_vector, delta)

The `EnemyPaddle`, which also extends `Paddle`, adjusts its position based on a target: the `Ball`. The further the ball
is above or below the paddle, the faster the paddle moves in that direction. When the distance between the centers of
the paddle and the ball is 50 pixels or more, the input vector is at its maximum.

    func _physics_process(delta):
        var distance = target.position.y -position.y
        var speed = clampf(distance / 50, -1, 1)
        move(Vector2.DOWN * speed, delta)

The `EnemyPaddle` provides its parent node with the ability to set a target via API.

    var target: Node2D

    func set_target(newTarget: Node2D):
        target = newTarget

This offers the advantage that the `EnemyPaddle` and the `Ball` do not need to know anything about each other. Instead,
the `Main` node links the two after their own instantiation.

    func _ready():
	    $EnemyPaddle.set_target($Ball)

## The Ball

The node structure of the ball is identical to that of the paddles: `CharacterBody2D`, `Sprite2D`,
and `CollisionShape2D` for the same reasons. However, here I use a special feature of the `move_and_collide` method that
I have only used so far for positioning along a predefined vector:

    var collision: KinematicCollision2D  = move_and_collide(direction * speed * delta)

The function returns a `KinematicCollision2D` if a collision occurs in this frame, otherwise null. In the case of a
collision with the ceiling or floor, the y-component of the ball must be inverted. In the case of a collision with the
paddles, the x-component must be inverted. However, instead of having to do this manually, the `KinematicCollision2D`
offers a much more elegant solution:

    if collision:
		direction = direction.bounce(collision.get_normal())

I can simply calculate the direction after a reflection off the collision object using the `bounce` method. Even the
required normal is supplied in the `KinematicCollision2D` object. This allows
for [physically correct scattering](https://en.wikipedia.org/wiki/Specular_reflection) even with complex shapes, without
any additional effort.

However, using this approach, the ball could not be controlled, and could only be reflected at a predetermined angle. If
two perfect players were playing, the trajectory of the ball would be determined from the start to infinity. As I am not
a big fan of [determinism](https://en.wikipedia.org/wiki/Determinism), I have added a bit of chaos:

    func adjust_direction_y_based_on_hit_area(paddle: Paddle):
	    direction.y += (position.y - paddle.position.y) / 200

    func _physics_process(delta):
        (...)
        var collider = collision.get_collider()
	    	if collider is Paddle:
		    	adjust_direction_y_based_on_hit_area(collider)

The further above the center of the paddle the ball hits, the stronger it will be accelerated upwards, and vice versa if
it hits below the center of the paddle. This not only allows one to potentially confuse the opponent (if they were to
think), but also allows one to accelerate the ball and win the game. The magnitude of the x-component of the ball's
velocity remains constant, but by positioning the paddle appropriately, one can increase the magnitude of the
y-component of the ball's velocity at each collision, and eventually win against the AI.

Ich bin mir nicht sicher, ob der Code für die Beschleunigung hier an der richtigen Stelle ist. Der Ball muss für diese
Interaktion wissen, dass Paddel existieren und das erscheint mir reichlich unsauber. Vermutlich wäre es sauberer, ein
Signal auszusenden, das den `collider` als Payload hat und die `Main` node verknüpft dann erst `Paddle` und `Ball`,
indem sie den collider type prüft und dann ein `accelerate` oder ähnlich auf dem `Ball` aufruft. Aber wie gesagt, ich
bin mir hier nicht sicher und struggle sowieso regelmäßig, wie hier eine vernünftige Architektur aussehen könnte.

I'm not sure if the acceleration code is in the right place here. The ball needs to know that paddles exist for this
interaction, which seems rather messy to me. It would probably be cleaner to emit a signal that has the `collider` as a
payload, and then have the `Main` node link `Paddle` and `Ball` by checking the collider type and calling `accelerate`
or something similar on the `Ball`. But as I said, I'm not sure here and I regularly struggle with how to design a
reasonable architecture here.

The ball also detects and signals when it hits one of the paddles, and even resolves it depending on which paddle it
hits:

    func _physics_process(delta):
        (...)
        if collider is Paddle:
            (...)
            if collider is PlayerPaddle:
                emit_signal("hit_player")
            elif collider is EnemyPaddle:
                emit_signal("hit_enemy")

## Sounds

These signals are linked to the `SoundService` by the `Main` node to play alternating "Ping" and "Pong" sounds.
The `SoundService` is a single `AudioStreamPlayer2D` node, where I load and play the appropriate signal depending on the
sound.

    func play_sound(sound: String):
        stream = load(sound)
        play()

Practically, a green arrow appears next to the method linked with the signal in the editor.

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-03/play_sound.png">
    </figure>   
</div>

Clicking on this arrow opens a window that documents the connection in a very understandable way:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-03/signal_connection.png">
    </figure>   
</div>

Overall, there are 4 sounds in the game: "Ping", "Pong", a sad melody when the player loses, and a happy melody when
they win. For the first release, all the sounds are [my voice](https://www.youtube.com/@sweetgeorgiebrown6143) recorded
with [Audacity](https://www.audacityteam.org/). I just wanted to test how to place sounds in the right positions, and I
find that it's straightforward to do in Godot, as seen above. The only thing that makes me a bit uneasy is that I feel
like the sounds are played with a slight delay, and this was already the case before I had to load each sound before
playing it. I will need to investigate this further to see how to improve it.

## The World

The `World` scene is kept quite simple with a green background, a `CollisionShape2D` at the top and bottom (representing
the ceiling and the floor), as well as to the left and right (representing the player's goal and the enemy's goal).

<div class="centered-container">
    <figure>
        <img width="600px" src="{{site.baseurl}}/assets/images/2023-03/world.png">
        <figcaption>World scene as seen in the 2D editor.</figcaption>
    </figure>   
</div>

There are no big surprises with the nodes:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-03/world_nodes.png">
        <figcaption>Node structure of the the World scene.</figcaption>
    </figure>   
</div>

The background is implemented with a `ColorRect` node, which can easily be extended to cover the entire visible area and
colored in Godot.

The ceiling and floor are `StaticBody2D`. Using a `StaticBody2D`, the `Ball` can automatically collide with them using
its `move_and_collide` method.

The goals are `Area2D` that can be penetrated by the `Ball`. However, they register when other bodies enter their area
and emit a `body_entered(body)` signal, which also transmits the information of the penetrated body. This signal is used
in the `Main` node to register a scored goal, update the corresponding score label, and start a new round.

It's clear that I was inconsistent with regard to abstraction level. The ceiling and floor are themselves packed scenes,
in which `StaticBody2D` and `CollisionShape2D` are combined. With the goals, I added both as parent and child nodes
visibly to the World. As I mentioned, I am unsure about architecture in Godot. However, I believe that the packed scene
is more appropriate. The ceiling and floor serve the exact same purpose, as do the two goals. So, if functionality or
additional child nodes are ever added, they will almost certainly be added to both objects. The DRY principle therefore
demands encapsulation into a packed scene.

## The UI

The UI is straightforward. There are two labels to track the points of the player and the enemy, a menu, and a mostly
invisible countdown:

<div class="centered-container">
    <figure>
        <img width="600px" src="{{site.baseurl}}/assets/images/2023-03/ui_scene.png">
        <figcaption>The UI of the game.</figcaption>
    </figure>   
</div>

The corresponding node structure is:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-03/ui_nodes.png">
        <figcaption>The node structure of the UI scene.</figcaption>
    </figure>   
</div>

Das meistens unsichtbare `CountdownLabel` besitzt einen `Timer`, der bei jeder neuen Runde gestartet wird. Das Label
zeigt die verbleibende Zeit an und das `timeout` Signal wird in der `Main` node mit dem `Ball` verknüpft und startet
ihn.

The `Menu` is again a packed scene and consists of a semi-transparent `ColorRect`, a `Label` for the game title, and
a `Button`. When the button is pressed, the entire menu disappears, the score is reset (if a round has already been
played to the end), and the countdown, and thus a new round, is started.

## Takeaways

The most important takeaway for me is that I enjoy using Godot, and it provides me with the exact solution I was looking
for in my initial projects. It seems powerful enough to not limit my ideas for a long time. At the same time, the entry
is pleasant with pre-built nodes that allow for collisions and movement with very little code and effort, as well as
signals that can link any objects together, and packed scenes that make modular, maintainable, and expandable structure
very easy.

I didn't take longer for the project than to write the blog post about it (which makes me doubt the idea of blogging).
The main obstacle was a proper structure. As you may have noticed in the text above, I became aware of some
inconsistencies while writing. And especially when it came to figuring out how to modularize things, I was often
uncertain.

What helped me the most here was the realization of defining the API for a packed scene in its main node. The main node
then communicates with the child nodes, and someone implementing the packed scene doesn't need to know anything about
the structure of the packed scene. The child nodes throw signals upwards. Either these can be finally processed directly
in the main node, or the main node receives the signal and throws a corresponding signal that can be processed by the
implementer of the packed scene. This loosely reminds me of the structure of the only frontend framework I
know: [Vue](https://vuejs.org/).

## Problems

Refactoring can be a tedious task, especially in Godot, as it is not a fully-fledged IDE
like [Intellij](https://www.jetbrains.com/de-de/idea/) or other JetBrains IDEs. It lacks automated refactoring tools to
extract a code block into a method or rename variables. Moreover, although you can convert an existing branch of a node
to a new packed scene, all signal connections with objects outside the new packed scene are lost, requiring manual
reconnection. This can be time-consuming and discourages cleaning up code. While
a [Ryder Plugin](https://plugins.jetbrains.com/plugin/20123-gdscript) exists for Godot scripting, it does not work with
the latest Ryder version at the time of writing.

Another hurdle to enjoy cleaning up is that I am not yet sure how unit or integration tests could work. A brief research
has confused me more than enlightened me, as it only dealt with C++ code. The lack of any tests takes away my confidence
in making changes while still guaranteeing some functionality. Before starting a new, larger project, I definitely need
to invest more research in this area.

## What's on the horizon

I am not finished with the game yet and plan to put a lot more work into it.

### Release

This game in its current state is nothing more than a mere exercise. However, my plan is to turn it into a real game
that is fun to play for at least 15 minutes and to release it in some form. The easiest option is
probably [itch.io](https://itch.io/), a platform for indie games where games can be downloaded for free or for a fee.
The next step would be to see if a Steam release is realistic and to get to know that world. Finally, I would like to
revise the controls to release the game on the Google Play Store. The goal is simply to learn all the relevant options
for me.

The main idea and reason why the game is called Battle Pong, is the implementation of power-ups. Players should be able
to collect various weapons to shoot the enemy paddle, making it easier to score points. Additionally, I am considering
making the enemy paddle faster, making it necessary to successfully shoot it in order to score. Implementing power-ups
will be the next concrete step in the game's development.

### Visuals

Overall, I have absolutely no idea about graphic design. However, the menu is particularly ugly and uninspired. I
definitely want to spice it up a bit. Instead of a simple font, we might need a logo. And instead of a semi-transparent
overlay, we could display something dynamic.

This would also be a direct upgrade for the background. I would like to have it not static anymore. For example, a
slowly changing starry sky would be great.

Lastly, the paddles and ball look very clumsy. I don't have any ideas yet, but I definitely want to revamp them before
releasing the game.

### Sound

Here, I actually already have an idea. A loop of a few basic chords in a key as continuous background music. And when
the paddles hit, they don't play my voice anymore, but a tone from the pentatonic scale of the corresponding key. I
imagine that you could then play something like an un-rhythmic solo with the paddles.

I've never really produced music before, but I've been wanting to do it for a while now :)

## Special Thanks

*Special thanks go to [sschellhoff](https://github.com/sschellhoff) for input concerning the structure of the game code.
He actually even had more ideas, that I did not include yet. But before his comments, the main scene was a mess.* 

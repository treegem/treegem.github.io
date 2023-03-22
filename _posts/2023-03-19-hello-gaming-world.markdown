---
layout: post
title:  "Hello, (Gaming) World!"
excerpt: "Equipped with the basics, I wanted to create a small game as a finger exercise on my own. In programming
languages, it's a common classic to output \"Hello, World!\" as a starting point. I think creating Pong is a suitable
equivalent, like a \"Hello, Gaming World!\""
image: "/assets/images/2023-03/dodge_preview.gif"
card: "summary"
---

Equipped with the basics, I wanted to create a small game as a finger exercise on my own. In programming languages, it's
a common classic to output "Hello, World!" as a starting point. I think
creating [Pong](https://de.wikipedia.org/wiki/Pong) is a suitable equivalent, like a "Hello, Gaming World!". The code of
the first
version can be found [in my repository](https://github.com/treegem/SweetPong).

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

# Main Scene

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

# The Paddles

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

# The Ball

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

# Sounds

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

Overall, there are 4 sounds in the game: "Ping", "Pong", a sad melody when the player loses, and a happy melody when they win. For the first release, all the sounds are [my voice](https://www.youtube.com/@sweetgeorgiebrown6143) recorded with [Audacity](https://www.audacityteam.org/). I just wanted to test how to place sounds in the right positions, and I find that it's straightforward to do in Godot, as seen above. The only thing that makes me a bit uneasy is that I feel like the sounds are played with a slight delay, and this was already the case before I had to load each sound before playing it. I will need to investigate this further to see how to improve it.

# The World

Die Welt ist ziemlich simpel gehalten.

# The UI

# Outlook
Sound, Hintergrund, PowerUps, Release

# Problems: Unit Testing, Refactoring, Die Stelle wo der Ball beschleunigt wird

*P.S.: Thanks to [sschellhoff](https://github.com/sschellhoff) for input concerning the structure of the game code. He
actually even had more ideas, that I did not include yet. But before his comments, the main scene was a mess.* 

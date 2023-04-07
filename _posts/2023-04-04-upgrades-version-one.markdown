---
layout: post
title: "Powering Up Pong"
excerpt: "Pong is awesome, Pong captivates everyone, and you can never get enough of Pong. Nevertheless, I thought it would be worth taking it up a notch. Upgrades are needed, and here I describe the prototype for the upgrade system."
image: "/assets/images/MustacheMan.png"
toc: true
---

Pong is awesome, Pong captivates everyone, and you can never get enough of Pong. Nevertheless, I thought it would be
worth taking it up a notch. Upgrades are needed, and here I describe the prototype for the upgrade system. The code of
the prototype can be found [in my repository](https://github.com/treegem/SweetPong/tree/upgrade_prototype). Just
as [the current version](https://github.com/treegem/SweetPong).

## Upgrade

For the initial version, there is only one type of upgrade: slow bullets. The upgrade spawns randomly somewhere between
the paddles, must be collected with the ball, and then activated. Upon activation, a bullet is launched horizontally
from the player towards the enemy. If it hits the enemy, it is temporarily slowed down significantly.

<div class="centered-container">
  <video width="600" loop autoplay muted>
    <source src="{{site.baseurl}}/assets/videos/2023-04/upgrade_fired.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <figcaption>The ball collects an upgrade for the player, who can then fire a slow bullet at the enemy to slow it down.</figcaption>
</div>

The node structure is simple: an `Area2D` with a Sprite and a CollisionShape. `Area2D` has a very practical
signal: `body_entered(body: Node2D)`. This signal is triggered when another body enters the `Area2D`, such as the ball.
That's why the body_entered signal is connected to the `Area2D` itself:

    func _on_body_entered(body):
        if body is Ball:
            if body.lastContact.is_in_group("upgrade_collector"):
                body.lastContact.receive_upgrade()
                emit_signal("was_collected")
                queue_free()

The ball has been modified so that it knows its last contact, allowing differentiation between the `PlayerPaddle` and
the `EnemyPaddle`. As soon as the ball enters the upgrade, the `receive_upgrade` method of the last touched paddle is
triggered. Subsequently, the `was_collected` signal is triggered, and finally, the upgrade is destroyed
using `queue_free`. I must admit, to my shame, that the signal is a relic from the development process and has no impact
anymore. By now, it is replaced by the active triggering of `receive_upgrade`.

## PlayerPaddle

First, the `PlayerPaddle` had to be added to the `upgrade_collector` group, so that the logic described earlier would
also be triggered for the `PlayerPaddle`. This can be easily accomplished on the right side of the editor:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-04/player_paddle_group.png">
        <figcaption>The PlayerPaddle is added to the group upgrade_collector.</figcaption>
    </figure>   
</div>

### Upgrade Collection

Here, I am using the group as a kind of interface, unfortunately without a formal contract. Therefore, I cannot rely on
every node in the `upgrade_collector` group offering a `receive_upgrade` method. I have to be disciplined and ensure it
manually. Here is the code that was added to the `PlayerPaddle` for collecting (and losing) upgrades:

    var upgraded: bool = false

    func receive_upgrade():
        upgraded = true
        glow()
    
    func remove_upgrade():
        upgraded = false
        dim()

There is a new flag, `upgraded`, which indicates whether the paddle currently has an upgrade available or not. This is
controlled by the `receive_upgrade` and `remove_upgrade` methods. The `remove_upgrade` method is only needed if a new
game is started while still having an upgrade. In this case, both the player and the enemy should start without upgrades
again.

### Visuals

However, you can also see that the two methods `glow` and `dim` are being called. These control a new node that I have
added to the `PlayerPaddle`: `PointLight2D`. This particular node has the following description in the documentation:

    Casts light in a 2D environment. 
    This light's shape is defined by a (usually grayscale) texture.

As a texture, I created a radially linear decreasing hemisphere in [Inkscape](https://inkscape.org/de/):

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-04/light_source.png">
        <figcaption>The texture for eh PointLight2D node: a radially decaying hemisphere.</figcaption>
    </figure>   
</div>

Die zugehörigen `glow` und `dim` Methoden sind auch denkbar einfach gehalten. Sie schalten die Lichtquelle einfach nur
an und wieder aus:

    func glow():
        $PointLight2D.enabled = true
    
    func dim():
        $PointLight2D.enabled = false

After playing around with this node, I decided to make the color reddish via the Inspector and to compress the sphere
strongly in the x-dimension. The hemisphere does not start directly on the surface but has moved a few pixels into the
paddle. Overall, this creates a very nice glowing effect on the paddle's surface, suggesting that an upgrade can now be
used:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-04/paddle_glowing_non_glowing.png">
        <figcaption>Comparison of paddle without and with upgrade. Left: Paddle without upgrade. The PointLight2D node is disabled. Right: Paddle with upgrade. The PointLight2D node is enabled and lets the paddle surface glow DRAMATICALLY.</figcaption>
    </figure>   
</div>

### Upgrade Usage

Well, such an upgrade also needs to be triggered. For this purpose, I added the following two methods:

    func _unhandled_input(event):
        if Input.is_action_pressed("ui_accept"):
            if upgraded:
                trigger_upgrade()

    func trigger_upgrade():
        emit_signal("upgrade_triggered", self)
        upgraded = false
        dim()

The [`_unhandled_input`](https://docs.godotengine.org/en/stable/classes/class_node.html#class-node-method-unhandled-input)
method manages inputs, as the name suggests. Some inputs
are [already processed in the `_physics_process`]({{site.url}}/2023/03/19/hello-gaming-world.html#the-paddles) method.
However, this mainly involves inputs that need to be processed regularly in every physics frame to control movement. For
sporadically occurring inputs, the `_unhandled_input` method is a more natural choice.

In my case, pressing the spacebar while in the upgraded state calls the `trigger_upgrade` method. This removes the
upgrade and turns off the light source. It might sound a bit dull, as it doesn't seem like an obvious advantage in the
game. BUT: a signal is emitted, which is then processed in the `UpgradeService`, and actually triggers a game-changing
upgrade there.

Even at the time of writing this article, I regret the decision, as I simply find it more natural for the paddle itself
to be responsible for what triggering an upgrade means. Additionally, signals can quickly become confusing
and [make the code maintenance quite difficult]({{site.url}}/2023/03/19/hello-gaming-world.html#problems). I think in a
next iteration, some of the code will move back into the `PlayerPaddle`.

## UpgradeService

The `UpgradeService` is a simple `Node` with a script and a `Timer` as child node. The `Timer` is started either at the
beginning of a new round or when triggering the last upgrade. When the `Timer` runs out, a new `Upgrade` is generated
somewhere between the paddles:

    var upgrade: PackedScene = preload("res://upgrade.tscn")

    func create_upgrade():
        var newUpgrade: Upgrade = upgrade.instantiate()
        newUpgrade.position.x = randi_range(upgrade_position_x_min + upgrade_radius, upgrade_position_x_max - upgrade_radius)
        var max_y: int = get_viewport().get_visible_rect().size.y
        newUpgrade.position.y = randi_range(upgrade_position_y_min + upgrade_radius, upgrade_position_y_max - upgrade_radius)
        add_visible_child(newUpgrade)
    
    
    func add_visible_child(node: Node):
        node.z_index = 1
        add_child(node)

I have made the minimum and maximum values for the x and y positions accessible in the Inspector using `@export`:

    @export var upgrade_position_x_min: int
    @export var upgrade_position_x_max: int
    @export var upgrade_position_y_min: int
    @export var upgrade_position_y_max: int

Unfortunately, I didn't have a clever solution for this, so I created a node and moved it all the way to the
left/right/top/bottom, read the current position there, and then manually entered these values in the Inspector.

At first, I fell for the misconception that I had to create the `add_visible_child` method in the `Main` node and could
only access it through a signal. The reason for this was that the nodes created here were initially not visible.
Visibility is determined, among other factors, by the position of the nodes relative to each other in the node tree. So,
depending on whether the `UpgradeService` was above or below, for example, the `World` node, its child nodes could be
seen or not. Fortunately, this position is only considered after the `z_index`. By default, all nodes get
the `z_index = 0`. Therefore, I set the `z_index = 1` here, making all generated child nodes visible.

As mentioned earlier, the `UpgradeService` is also responsible for generating the `SlowBullet` when a `PlayerPaddle`
triggers its collected upgrade. This is a sad remnant of my trial and error. This should definitely be moved to the
script of the `PlayerPaddle` itself:

    var slowBulletScene: PackedScene = preload("res://slow_bullet.tscn")

    func _on_player_paddle_upgrade_triggered(paddle: Paddle):
        var slowBullet = slowBulletScene.instantiate()
        slowBullet.position = paddle.position + Vector2(-paddle.width() / 2,0)
        add_visible_child(slowBullet)
        reset_timer()

## SlowBullet

The `SlowBullet` is an `Area2D` node in order to be able to use the `body_entered` signal. Currently, only horizontal
movement to the left is implemented. I have implemented an accelerated movement here to make aiming a bit more
challenging:

    func _physics_process(delta):
        SPEED += delta * ACCELERATION
        ACCELERATION *= 1.1
        position.x -= SPEED * delta
    
    
    func _on_body_entered(body):
        if body.is_in_group("hittable"):
            body.get_hit(self)
            queue_free()

Initially, I had problems with the `body_entered` signal not being triggered regularly. The issue is that the bullet
position only takes discrete values. If the bullet's speed is high enough, it's possible that it is completely to the
right of the `EnemyPaddle` in one frame and already completely to the left in the next calculated frame. That's why I
increased the physics frame rate from 60 frames per second to 120 frames per second in the editor settings. Since only
nodes from the `hittable` group can be hit, the `EnemyPaddle` had to be added to this group, of course.

## EnemyPaddle

The `EnemyPaddle` was added to both the `hittable` and `upgrade_collector` groups. The former allows it to be hit by
the `SlowBullet`, and the latter enables it to collect an upgrade as well. However, currently, when collecting the
upgrade, only a signal is sent to destroy the upgrade. The `EnemyPaddle` does not gain any advantage:

    func receive_upgrade():
        emit_signal("upgrade_collected")

I made the `EnemyPaddle` fundamentally faster by introducing a `speed_modifier` to make it significantly more difficult
to win against it without upgrades:

    const MAX_SPEED_MODIFIER: float = 2
    var speed_modifier: float = MAX_SPEED_MODIFIER
    
    func _physics_process(delta):
        var distance = target.position.y -position.y
        var speed = clampf(distance / 100, -1, 1)
        move(Vector2.DOWN * speed * speed_modifier, delta)

When hit by a `SlowBullet`, the `EnemyPaddle` is temporarily slowed down. The slowdown should continuously be reduced to
zero. For this, I use a Godot-specific
feature: [Tweens](https://docs.godotengine.org/en/stable/classes/class_tween.html).
They allow for continuous interpolation of arbitrary properties from a start value to an end value. Various
interpolation types are provided for this purpose. There
are [nice visualizations](https://monnef.gitlab.io/transition_laboratory_of_godot/transition_laboratory_of_godot.html)
of the different types on the internet. I chose an exponential interpolation here because it changes the property very
slowly at the beginning and then increasingly faster. This felt particularly sticky:

    func get_hit(bullet: Node):
        var tween = get_tree().create_tween()
        speed_modifier = MAX_SPEED_MODIFIER / 20
        tween.tween_property(self, "speed_modifier", MAX_SPEED_MODIFIER, 4).set_trans(Tween.TRANS_EXPO)

## Planned upgrade rework

At the beginning of this post, you can see a short section of what playing with upgrades looks like. Unfortunately, it's absolutely no fun. Zero. Nada. Not at all. The effect is too weak and one can still easily win without them once you understand ball control through the paddle's hit area. That's why I'll definitely make some changes in the next step.

### New Upgrade Types

Upgrades should become a much more central part of the game (although Pong is so awesome that everyone freaks out when they get to play Pong). For this, they need to be almost constantly present. I think a first simple step could be to introduce new upgrade types, such as enlarging one's own paddle, shrinking the respective other paddle, or possibly having a one-time safety net behind one's own paddle. These would also be triggered automatically upon collection and would allow the next upgrade timer to start immediately.

### Overhaul the SlowBullet upgrade

Additionally, the SlowBullets are far too rarely usable for their weak effect. At the same time, I don't want to drastically increase the effect, as this could lead to an automatic win with good aiming when using a SlowBullet. That's why I plan to change the SlowBullet upgrade so that it leads to regular shooting with it from the first upgrade. For example, every 5 seconds, and each subsequent SlowBullet upgrade increases the frequency up to a maximum.

### Empower the EnemyPaddle

Since all the described upgrades can now be used automatically, there's nothing to prevent them from working for the EnemyPaddle as well. This makes it more difficult to win without your own upgrades and hopefully encourages players to strive to collect more upgrades than their opponent.

## Improving the SoundService

In an earlier post, I complained about the Ping and Pong sounds seemingly being played with a delay and not directly upon contact between the ball and the paddle. I have found a simple solution for this: preload `em all. 

    var ping_sound: Resource = preload("res://sounds/ping.mp3")
    var pong_sound: Resource = preload("res://sounds/pong.mp3")
    var win_sound: Resource = preload("res://sounds/win.mp3")
    var lose_sound: Resource = preload("res://sounds/lose.mp3")
    
    func play_player_sound():
        play_sound(ping_sound)
    
    func play_enemy_sound():
        play_sound(pong_sound)
    
    func play_lose_sound():
        play_sound(lose_sound)
    
    func play_win_sound():
        play_sound(win_sound)
    
    func play_sound(sound: Resource):
        stream = sound
        play()

As a result, all required sounds are now loaded into memory at the beginning of the game and only need to be played. With the previous solution, the sound file was loaded at the time of playback, then set as a stream, and only then could it be played. I think it's better to be more conservative than I have been. It would be enough to preload the Ping and Pong sounds. The Win and Lose sounds could still be loaded only when needed.

## What grinds my gears

In conclusion, I just wanted to complain once more about the lack of a way to define the required methods of a group somewhere, like with an interface. Because, from my understanding so far, groups serve the rough purpose of an interface. At the same time, they allow you to do everything wrong. In my project, this is not an issue, but I hope to find a better solution for larger projects by the time I get to them. 

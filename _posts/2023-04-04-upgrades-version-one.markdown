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

The node structure is simple: an `Area2D` with a Sprite and a CollisionShape. `Area2D` has a very practical signal: `body_entered(body: Node2D)`. This signal is triggered when another body enters the `Area2D`, such as the ball. That's why the body_entered signal is connected to the `Area2D` itself:

    func _on_body_entered(body):
        if body is Ball:
            if body.lastContact.is_in_group("upgrade_collector"):
                body.lastContact.receive_upgrade()
                emit_signal("was_collected")
                queue_free()

The ball has been modified so that it knows its last contact, allowing differentiation between the `PlayerPaddle` and the `EnemyPaddle`. As soon as the ball enters the upgrade, the `receive_upgrade` method of the last touched paddle is triggered. Subsequently, the `was_collected` signal is triggered, and finally, the upgrade is destroyed using `queue_free`. I must admit, to my shame, that the signal is a relic from the development process and has no impact anymore. By now, it is replaced by the active triggering of `receive_upgrade`.

## PlayerPaddle

First, the `PlayerPaddle` had to be added to the `upgrade_collector` group, so that the logic described earlier would also be triggered for the `PlayerPaddle`. This can be easily accomplished on the right side of the editor:

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-04/player_paddle_group.png">
        <figcaption>The PlayerPaddle is added to the group upgrade_collector.</figcaption>
    </figure>   
</div>

### Upgrade Collection

Here, I am using the group as a kind of interface, unfortunately without a formal contract. Therefore, I cannot rely on every node in the `upgrade_collector` group offering a `receive_upgrade` method. I have to be disciplined and ensure it manually. Here is the code that was added to the `PlayerPaddle` for collecting (and losing) upgrades:

    var upgraded: bool = false

    func receive_upgrade():
        upgraded = true
        glow()
    
    func remove_upgrade():
        upgraded = false
        dim()

There is a new flag, `upgraded`, which indicates whether the paddle currently has an upgrade available or not. This is controlled by the `receive_upgrade` and `remove_upgrade` methods. The `remove_upgrade` method is only needed if a new game is started while still having an upgrade. In this case, both the player and the enemy should start without upgrades again.

### Visuals

However, you can also see that the two methods `glow` and `dim` are being called. These control a new node that I have added to the `PlayerPaddle`: `PointLight2D`. This particular node has the following description in the documentation:

    Casts light in a 2D environment. 
    This light's shape is defined by a (usually grayscale) texture.

As a texture, I created a radially linear decreasing hemisphere in [Inkscape](https://inkscape.org/de/):

<div class="centered-container">
    <figure>
        <img src="{{site.baseurl}}/assets/images/2023-04/light_source.png">
        <figcaption>The texture for eh PointLight2D node: a radially decaying hemisphere.</figcaption>
    </figure>   
</div>

Die zugehörigen `glow` und `dim` Methoden sind auch denkbar einfach gehalten. Sie schalten die Lichtquelle einfach nur an und wieder aus:

    func glow():
        $PointLight2D.enabled = true
    
    func dim():
        $PointLight2D.enabled = false

After playing around with this node, I decided to make the color reddish via the Inspector and to compress the sphere strongly in the x-dimension. The hemisphere does not start directly on the surface but has moved a few pixels into the paddle. Overall, this creates a very nice glowing effect on the paddle's surface, suggesting that an upgrade can now be used:

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

The [`_unhandled_input`](https://docs.godotengine.org/en/stable/classes/class_node.html#class-node-method-unhandled-input) method manages inputs, as the name suggests. Some inputs are [already processed in the `_physics_process`]({{site.url}}/2023/03/19/hello-gaming-world.html#the-paddles) method. However, this mainly involves inputs that need to be processed regularly in every physics frame to control movement. For sporadically occurring inputs, the `_unhandled_input` method is a more natural choice.

In my case, pressing the spacebar while in the upgraded state calls the `trigger_upgrade` method. This removes the upgrade and turns off the light source. It might sound a bit dull, as it doesn't seem like an obvious advantage in the game. BUT: a signal is emitted, which is then processed in the `UpgradeService`, and actually triggers a game-changing upgrade there.

Even at the time of writing this article, I regret the decision, as I simply find it more natural for the paddle itself to be responsible for what triggering an upgrade means. Additionally, signals can quickly become confusing and [make the code maintenance quite difficult]({{site.url}}/2023/03/19/hello-gaming-world.html#problems). I think in a next iteration, some of the code will move back into the `PlayerPaddle`.

## Upgrade Service

Upgrades überarbeiten

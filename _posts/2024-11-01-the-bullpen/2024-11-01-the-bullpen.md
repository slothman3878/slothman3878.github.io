---
layout: post
title:  "The Bullpen"
date:   2024-11-01 22:00:00 -0700 
categories: rust bevy game-dev
math: true
---

This was meant to be a simple project: a quick and easy way for me to familiarize myself with Rust and the Bevy engine. My first mistake was underestimating the scope of the project. In hindsight, I should have recognized that developing a well-defined and complete baseball game within the span of a few months would not be feasible for me.

![screen capture of old project](/static/images/old-project.gif)

The Bullpen is built from the scraps of my former, overly ambitious undertaking. I'm prioritized just the core motivation of building a baseball game that focuses on or attempts to emulate the technical mechanics involved.

## Baseball Flight Simulator

The core piece of this whole endeavor: simulating movement of a ball in flight given it's spin and seam orientation. The simulator I've implemented is based off of the [UMBA baseball flight calculator](https://github.com/AndRoo88/Baseball-Flight-Calculator) by [baseballero](https://baseballaero.com). The [physics of baseball from the University of Illinois](https://baseball.physics.illinois.edu) was also of great help. The simulator calculates the drag, magnus, and seam-shifted-wake forces.

$$F_{drag} = \frac{1}{2} \space \rho \space v^2 \space C_D \space A$$

$$F_{spin} = \frac{1}{2} \space C_L \space \rho \space A \space (\omega \times v)$$

$$F_{ssw} = L_{\text{seams}} \space C_{ssw} \space v^2$$

$$F^{(t)} = F_{drag}^{(t)} + F_{spin}^{(t)} + F_{ssw}^{(t)}$$

This is not a precise simulation. From what I understand, the constants are empirically derived. Also, much to my dismay, the simulation is in imperial units, and conversion back to metric will cause some precision loss. More importantly, the simulator applies the appropriate forces necessary to move the ball to it's predicted position in the succeeding frame.

```rs
// option 3
pub(crate) fn update_state_and_get_acceleration(
    &mut self,
    config: &BaseballPluginConfig,
    translation: DVec3,
    velocity: DVec3,
    delta_t: f64,
) -> DVec3 {
    self.translation = translation;
    self.v = velocity;

    self.update_state(config, delta_t);
    let distance = self.translation - translation;

    (self.v * self.v - velocity * velocity) / (2. * distance)
}

...

force.force = a.from_baseball_coord_to_bevy().as_vec3() * MASS;
```

For now this is fine. A side spin creates horizontal break, a top spin creates vertical break, and two seam fastballs have slight vertical drop compared to the four seam counterpart. Verisimilitude is all I wish to achieve for now.

Two versions of the simulator exists. One is built on top of [bevy-rapier](https://crates.io/crates/bevy-rapier-baseball-flight), the other on top of [bevy-avian](https://crates.io/crates/bevy-avian-baseball-flight). Fundamentally they do the same things, but the Bullpen uses the rapier version over the avian version mostly because Avian's sphere colliders do odd things when they spin. I suspect it has something to do with the way avian handles friction between two colliders. I will investigate further when I have time.

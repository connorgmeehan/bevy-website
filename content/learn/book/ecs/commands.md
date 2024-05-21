+++
title = "Manipulating Entities With Commands"
insert_anchor_links = "right"
[extra]
weight = 4
status = 'hidden'
+++

**Commands** allow you to make mutations to your game's world while keeping systems running in parallel.  They work by deferring changes until they
are safe to apply, e.g. when no other systems are running.

By using Commands you will be able to spawn and despawn entities, insert/remove components,
and insert/remove resources while still taking advantage of Bevy's parrallelism.

This chapter will cover:
1. [How to use commands](#1-how-to-use-commands)
2. [How to control when a command is applied](#2-how-to-order-commands)
3. [Best practices and gotchas](#3-best-practices-and-gotchas)
4. [How commands work internally](#4-how-commands-work-internally).

## 1. How to use commands

You can use Commands by using the [`Commands`] [`SystemParam`] as an argument in your system.

```rust
// This system leaves a trail of entities behind the player.
fn leave_trail_system(
    mut commands: Commands,
    player: Query<&Transform, With<Player>>,
) {
    let player_transform = player.single();

    commands.spawn(SpatialBundle {
        transform: player_transform.clone(),
        ..Default::default()
    })
}
```

## 2. How to order commands

- Commands are automatically applied at the end of the schedule.
    - Done via the [`apply_deferred`] system.
- If a system is ordered with `before()`/`after()`/`chain()` the command will be applied after the system is run.
    - See [system ordering] for more details
- Can also manually run [`apply_deferred`] systems although it's preferred to use system ordering to do the same thing.

## 3. Best practices and gotchas

## 4. How commands work internally

- [`Commands`] system parameter borrows the [`CommandQue`] from the world.
- When we interact with the [`Commands`] struct, we're pushing new [`Command`] structs to the [`CommandQue`]
- [`Commands`] use the [`Deferred`] struct to defer their changes until [`apply_deferred`] is called, which happens automatically after a system with commands is run.
- When [`apply_deferred`] is called, each command in the [`CommandQue`] is applied to the [`World`] and the [`CommandQue`] is emptied.

- Two systems that both use [`Commands`] cannot run in parallel.

```rust

#[derive(Component)]
pub struct Box;

#[derive(Component)]
pub struct Velocity(pub Vec3);


fn spawn_box(mut commands: Commands) {
    commands.spawn(...);
}

fn apply_gravity(boxes: Query<(&mut Transform, &mut Velocity)>) {
    for (mut transform, mut velocity) {
        velocity.0.y -= 9.18;
        transform.translation += velocity.0;
    }
}

fn main() {
    let mut app = App::new();
    app.add_systems(PostUpdate, (spawn_box, move_boxes));
    app.run();
}
```

<svg viewBox="0 0 400 120" xmlns="http://www.w3.org/2000/svg">
    <defs>
        <marker id="arrow" viewBox="0 0 7 7" refX="2.5" refY="2.5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
          <path d="M 0 0 L 5 2.5 L 0 5 z" fill="#999"></path>
        </marker>
    </defs>
    <style>
        .text {
          font: 8px sans-serif;
        }
        .meta {
          font: 6px sans-serif;
        }
    </style>
    <!-- CommandQue -->
    <rect x="10" y="15" width="380" height="10" rx="2" fill="transparent" stroke="lightblue"></rect>
    <text x="20" y="23" class="text" fill="lightblue">PostUpdate Schedule</text>
    <!-- System 1 -->
    <rect x="30" y="40" width="145" height="10" rx="2" fill="transparent" stroke="white"></rect>
    <text x="35" class="text" fill="white" y="48">spawn_box (System with Commands)</text>
    <path stroke="#999" fill="transparent" marker-end="url(#arrow)" d="M 20 30 L 20 45 L 30 45"></path>
    <path d="M 175 45 L 180 45 L 180 30" stroke="#999" fill="transparent" marker-end="url(#arrow)"></path>
    <!-- No commands system -->
    <rect x="30" y="60" rx="2" fill="transparent" stroke="white" height="10" width="200"></rect>
    <text class="text" fill="white" y="68" x="35">apply_gravity (System)</text>
    <path stroke="#999" fill="transparent" marker-end="url(#arrow)" d="M 20 30 L 20 65 L 30 65"></path>
    <path d="M 230 65 L 235 65 L 235 30" stroke="#999" fill="transparent" marker-end="url(#arrow)"></path>
    <!-- No more systems running -->
    <line x1="260" x2="260" y1="30" y2="100" stroke="#999" stroke-dasharray="5,5"></line>
    <!-- Flush Commands -->
    <text class="meta" fill="lightblue" x="305" y="60">Commands are applied at the</text>
    <text class="meta" fill="lightblue" x="305" y="65">end of the schedule, when no</text>
    <text class="meta" fill="lightblue" x="305" y="70">other systems are running.</text>
    <rect y="80" width="85" rx="2" fill="transparent" stroke="lightblue" height="10" x="300"></rect>
    <text class="text" fill="lightblue" x="305" y="88">apply_deferred</text>
    <path stroke="#999" fill="transparent" marker-end="url(#arrow)" d="M 290 30 L 290 85 L 300 85"></path>
    <!-- Legend -->
    <rect x="10" y="92" width="5" height="5" fill="white"/>
    <text x="17" y="97" class="meta" fill="white">Your System Logic</text>
    <rect x="10" y="102" width="5" height="5" fill="lightblue"/>
    <text x="17" y="107" class="meta" fill="lightblue">Bevy Internals</text>
    <!-- Annotation -->
    <text y="110" text-anchor="middle" class="text" fill="#999" x="200">Time</text>
    <line x1="10" y1="115" x2="390" y2="115" stroke="#999" marker-end="url(#arrow)" />
    <text x="22" class="meta" fill="lightblue" y="35">Borrows CommandQue from World</text>
    <text x="122" class="meta" fill="lightblue" y="35">Returns CommandQue to World</text>
</svg>

## Further reading


<!--
    1. Have a clear topic, and give a high-level overview of the subtopics it is going to cover and how they fit together.
    2. Be broken down into several sections / pages to focus on detailed topics.
        These should have simple, minimal examples explaining how that functionality works.
    3. Link to appropriate sections of quick start guides that demonstrate the ideas being taught in a more coherent way.
-->

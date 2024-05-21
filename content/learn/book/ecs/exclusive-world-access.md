+++
title = "Exclusive World Access"
insert_anchor_links = "right"
[extra]
weight = 8
status = 'hidden'
+++

{% todo() %}
- ⌛ Show how to work with a raw world
- ✅ Discuss and demonstrate custom commands
- ✅ Discuss and demonstrate exclusive systems
- ❌ Discuss and demonstrate `NonSend` resources
- ❌ Discuss and demonstrate `SystemParams`?
{% end %}

In Bevy you'll typically interact with your game's world via [`systems`] and [`SystemParams`], which are performant but sometimes limiting.
When you need more immediate or dynamic control of the world you will have to use **Exclusive World Access**.

Exclusive World Access is a term to describe updating your game directly and immediately via a mutable reference to the game [`World`], as opposed to a 
[`Commands`] or [`Query`] parameter on a system.  Because Exclusive World Access borrows the entire world we cannot take advantage of Bevy's Parallelism,
however it also has a number of unique capabilities:

- Immediately spawn and despawn entities, insert and remove components, and insert and remove resources.  
- Access components, entities and systems programatically.
- Use reflection to serialize / deserialize entities.
- Programatically run other systems or schedules.

See [here](#interacting-with-the-mut-world) for a general overview on how to use the [`World`](https://docs.rs/bevy/latest/bevy/prelude/struct.World.html) API.

Additionally, there are instances where you will have to learn how to use the [`World`](https://docs.rs/bevy/latest/bevy/prelude/struct.World.html) to make use of 
some more advanced patterns or features of Bevy.

- [Extending the Commands struct with Custom Commands](#custom-commands)
- [Creating custom SystemParams](#custom-systemparams) TODO
- [Accessing !Send structs using Non Send Resources](#non-send-resources) TODO

## Exclusive Systems 

An exclusive system is any system that receives `&mut World` as an argument.  It's "exclusive" because it borrows the entire world and,
until this system ends, no other system can run.  It's best to limit your use of exclusive systems as it prevents your application from parrallelising.

```rust
fn spawn_and_despawn(world: &mut World) {
    // Entity is immediately spawned in world.
    let id = world.spawn(Name::from("HelloWorld!")).id();

    // Entity is immediately despawned from world.
    world.entity_mut(id).despawn();
}
```

> You'll note that the API is very similar to the [`Commands`](./commands.md) api.  Most functions on the [`Commands`](./commands.md) 
> struct have a [`World`](https://docs.rs/bevy/latest/bevy/prelude/struct.World.html) counter part.

The data that normal Bevy systems access is static, defined by their [SystemParam](https://docs.rs/bevy/0.11.0/bevy/ecs/system/trait.SystemParam.html#implementors) 
types.  Exclusive systems are able to dynamically access anything on any Component or Resource. This is useful for runtime
Reflection, debugging, or using complex control flow to run different systems at different times.

```rust
fn dump_entities(world: &mut World) {
    // Get list of all entities in scene
    let all_entities: Vec<Entity> = world.query::<Entity>().iter(world).collect();

    for entity in all_entities {
        println!("Entity {entity:?} components:");

        // Returns a list of metadata of every component on entity
        let info = world.inspect_entity(e)
            .iter()
            .map(|info| info.name());

        // Log all the component names
        for name in info {
            println!(" - {name}")
        }
    }
}
```

## Custom Commands

It's possible to create your own custom [`Command`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Command.html) to help keep your
systems tidy and promote code re-use.  In-fact, this is how optional crates, such as [`bevy_hierarchy`](https://docs.rs/bevy/latest/bevy/hierarchy/index.html),
extend the [`Commands`](https://docs.rs/bevy/latest/bevy/ecs/prelude/struct.Commands.html) API with methods such as [`despawn_recursive`](https://docs.rs/bevy/latest/bevy/prelude/trait.DespawnRecursiveExt.html#tymethod.despawn_recursive).

The optimal use-case for creating a custom commands is for when you need to access a resource or entity in order to perform a one-off action.
For example, you might want to spawn an entity and register it in a Resource, or spawn an entity using a mesh/material stored in a resource to make use of GPU instancing.

To build and use a custom command you simply need to call `commands.add(MyCustomCommand)` where `MyCustomCommand` implements 
[`Command`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Command.html).

### Example

Lets say we have a game where we want to spawn enemies into the world.  Every 5 enemies spawned should be an extra big and evil enemy type.  We'll need to store this state
within a [`Resource`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Resource.html).

```rust
/// Component storing enemy type.
#[derive(Component)]
pub enum Enemy {
    Normal,
    BigBad,
};

// Stores data related to spawning enemies
#[derive(Resource)]
pub struct EnemySpawnState {
    // Number of enemies spawned
    spawned_count: usize,
    // The frequency we spawn a big bad enemy.  In this case 5
    big_bad_frequency: usize,
}
```

First we need a struct to represent our Command.  I want to be able to control the position the enemy spawns so I've included that in my command struct.
```rust
// Store input data for the command here.
pub struct SpawnEnemyCommand {
    pub position: Vec2,
};
```

Then we'll implement the [`Command`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Command.html) trait on it.  The `apply()` method is run when it is safe to do so (no other systems running).  Here we track some state in the `EnemySpawnState` resource and use it to control our spawning logic.

```rust
impl Command for SpawnEnemyCommand {
    // Apply 
    fn apply(self, world: &mut World) {
        let mut state = world.resource_mut::<EnemySpawnState>();
        state.spawned_count += 1;

        let should_spawn_big_bad = state.spawned_count % state.big_bad_frequency == 0;
        let enemy_tag = if should_spawn_big_bad {
            Enemy::BigBad
        } else {
            Enemy::Normal
        };

        world.spawn((
            enemy_tag,
            Transform {
                translation: Vec3::new(self.position.x, self.position.y, 0),
                ..Default::default()
            }
        ));
    }
}
```

Now we are able to push this command onto the commands que when the user presses the `Space` button.

```rust
fn spawn_enemy_on_space_press(ev_reader: EventReader<KeyboardInput>, mut commands: Commands) {
    for ev in ev_reader.read() {
        if matches!(KeyCode::SPACE, ev.key_code) && matches!(ButtonState::Pressed, ev.state) {
            commands.add(SpawnEnemyCommand { position: Vec2::new(0., 0.) });
        }
    }
}
```

### Extending the `Commands` struct.

It's possible to add your own extensions to the [`Commands`](https://docs.rs/bevy/latest/bevy/ecs/system/struct.Commands.html) struct to make accessing your custom command
This is how crates like [`bevy_hierarchy`](https://docs.rs/bevy_hierarchy/latest/bevy_hierarchy/) add methods like 
[`despawn_recursive`](https://docs.rs/bevy/latest/bevy/hierarchy/trait.DespawnRecursiveExt.html) to the Commands struct.

First we'll define our Commands extension as a trait.  Then we'll implement it for the different Command types.

```rust
pub trait SpawnEnemyExt {
    fn spawn_enemy(&mut self, position: Vec2);
}

impl SpawnEnemyExt for Commands {
    fn spawn_enemy(&mut self, position: Vec2) {
        self.add(SpawnEnemyCommand { position });
    }
}
```

Now we can use our new custom command with a similar API to the Bevy builtin commands.

```rust
fn spawn_enemy_on_space_press(ev_reader: EventReader<KeyboardInput>, mut commands: Commands) {
    for ev in ev_reader.read() {
        if matches!(KeyCode::SPACE, ev.key_code) && matches!(ButtonState::Pressed, ev.state) {
            commands.spawn_enemy(Vec2::new(0., 0.));
        }
    }
}
```

### Closure Commands

Sometimes you need to perform a one-off command but dont want to go through the process of defining a new command struct.  You can
parse a closure that receives a `&mut World` and it will be executed as a command.  This is possible because Bevy automatically implements
[`From<F: FnOnce(&mut World) + Send + 'static > for Command`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Command.html#impl-Command-for-F).

```rust
fn closure_commands(mut commands: Commands) {
    // A closure that receives a &mut World argument is also a Command.
    commands.add(|world: &mut World| {
        // Do whatever you want with the `World` here.
    })
}
```

## Non send resources

TODO

## Custom SystemParams
https://docs.rs/bevy_ecs/latest/bevy_ecs/system/trait.SystemParam.html#implementors

## Interacting with the `&mut World`

### Querying

You can perform one off queries into the world using the [`query`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.query) and [`query_filtered`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.query_filtered) methods.

```rust
fn exclusive_sys(world: &mut World) {
    let transforms_and_visiblity = world.query::<(&Transform, &Visiblity)>();

    let transforms_with_visiblity = world.query_filtered::<&Transform, With<Visiblity>();
}
```

### Accessing resources

You can access resouces using the [`resource`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.resource), [`get_resource`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.get_resource), [`resource_mut`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.resource_mut), and [`get_resource_mut`](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.get_resource_mut) methods.

```rust
#[derive(Resource)]
struct MyResource {
    field: f32,
}

fn exclusive_sys(world: &mut World) {
    // Get resource, panic if not exist
    let my_resource = world.resource::<MyResource>();
    println!("MyResource's field is {}", my_resource.field)

    // Get resource if it exists.
    if let Some(my_resource) = world.get_resource::<MyResource>() {
        println!("MyResource's field is {}", my_resource.field)
    }

    // Mutably get resource resource, panic if not exist
    let mut my_resource = world.resource_mut::<MyResource>();
    my_resource.field = 1;

    // Mutably get resource if it exists.
    if let Some(mut my_resource) = world.get_resource_mut::<MyResource>() {
        my_resource.field = 1;
    }
}
```

Sometimes you might want to borrow [`Resource`](https://docs.rs/bevy/latest/bevy/ecs/system/trait.Resource.html) without losing mutable
access to the world. In these instances you can use [`resource_scope`](https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.World.html#method.resource_scope)..

```rust
#[derive(Component)]
struct MyComponent(pub f32);

#[derive(Resource)]
struct MyResource {
    field: f32,
}

fn exclusive_sys(world: &mut World) {
    world.resource_scope::<MyResource, ()>(|world, my_resource| {
        // Still able to access World mutably.
        world.spawn(MyComponent(my_resource.field));
    });
}
```

### Sending and receiving Events

Events are stores in the [`Events<T>`](https://docs.rs/bevy_ecs/latest/bevy_ecs/event/struct.Events.html) resource and therefore can be accessed via the Resource access methods.

> The [`Events::drain(&mut self)`](https://docs.rs/bevy/latest/bevy/ecs/event/struct.Events.html#method.drain) method is not the same as
[`EventReader::read(&mut self)`](https://docs.rs/bevy/latest/bevy/ecs/event/struct.EventReader.html#method.read).  `drain` will delete the events from the resource once they've been iterated.
> This will prevent other systems from dispatching these events (Verify?)

```rust
pub struct MyEvent(pub usize);

fn exclusive_sys(world: &mut World) {
    // You will need to borrow the resource mutably in order to drain or send events.
    let mut events = world.resource_mut::<Events<MyEvent>>();
    for ev in events.drain() {
        println!("Received event: {ev:?}");
    }

    for i in 0..10 {
        events.send(MyEvent(i));
    }
}
```

### Accessing Assets

Similarly to events, assets are stored in the [`Assets<T>`](https://docs.rs/bevy_ecs/latest/bevy_ecs/event/struct.Events.html) resource and therefore can be accessed 
via the Resource access methods.

```rust
fn exclusive_sys(world: &mut World) {
    let mut color_materials = world.resource_mut::<Assets<ColorMaterial>>();

    let handle: Handle<ColorMaterial> = color_materials.add(Color::RED);
}
```

### Running Systems programatically

TODO

### Accessing SystemParams without a System

TODO

### Accessing component, resource metadata for reflection

TODO

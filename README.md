# Lootbox Legends

*App icon generated by DALL-E 3.*

This repo contains the code for **Lecture 13: AR & RealityKit**.

In this repo, we'll be building a ~~free-to-play interactive experiential game~~ an AR app that's entirely placing and opening loot boxes.

You'll need a physical device to run this app, as it needs to use the camera and motion sensors.

> [!IMPORTANT]
> To run the app on a physical device, you may need to change the bundle identifier in the project settings to something unique.
> 
> You can do this by adding a bunch of random letters/numbers to the bundle identifier, like:
> 
> `edu.upenn.seas.cis1951.Lootbox-Legends-SOMETHING-RANDOM-HERE`

Get started by cloning the repo and opening the project in Xcode, then follow along with these steps:

## Step 1: Add loot boxes

We'll start by setting up a "template" entity that we can clone to create loot boxes in the scene. Go ahead and add a property to `LootboxViewModel` to store this entity:

```swift
let lootboxTemplate: Entity
```

Now, let's set up the template entity when the view model initializes. We've already included a 3D model called `lootbox.usdz`, which we'll load into a new entity in the `init()` initializer:

```swift
init() {
    lootboxTemplate = try! Entity.load(named: "lootbox")
}
```

We'll want to clone this entity and place it in the scene whenever the user taps **Tap to add loot box**. To do so, add this code to the `addLootbox()` method:

```swift
func addLootbox() {
    guard let anchor, let arView else {
        return
    }

    let lootbox = lootboxTemplate.clone(recursive: true)
    anchor.addChild(lootbox)
}
```

Run the app, aim your phone at a table, and tap the button to add a loot box to the scene - you should see a loot box appear!

## Step 2: Place the loot box relative to the camera

Our app is looking good so far, but we're only placing loot boxes in a fixed location that the user can't control. Let's change that by placing the loot box in the center of the view, based on whereever the camera is.

To do this, we'll need to translate the center of the view in 2D space to a 3D point in the scene. There are many ways to do this (such as `unproject()` or `ray()`), but we'll go with a hit-testing method. Namely, we'll ask ARKit to cast a ray from the center of the screen and see what it hits.

Hit testing requires an entity with a `CollisionComponent`, but we've already added one to the `AnchorEntity` representing the table. All that's left for you to do is to do the hit test from the `addLootbox()` method. Add this just before you clone the loot box entity:

```swift
let hits = arView.hitTest(arView.center, query: .nearest)
guard let hit = hits.first else {
    showUnableToPlaceMessage = true
    return
}
```

Here, we ask the scene for the nearest object at the cneter of the screen. If we can't find anything, we'll ask our view to display an error message. If we do find something, we'll convert the hit position from the scene's coordinate space to the anchor's coordinate space:

```swift
let position = anchor.convert(position: hit.position, from: nil)
```

Then, we'll update our loot box creation code to place the loot box at this position:

```swift
lootbox.position = position
```

Run the app again - assuming you're aiming at a table, you should now be able to place loot boxes wherever you tap on the screen!

## Step 3: Add physics

Let's make our loot boxes a little more interesting with physics! Whenever we add a loot box, we'll drop it from above and let it bounce around a bit.

To add physics to an entity, we need two components: a `CollisionComponent` and a `PhysicsBodyComponent`. As you've already seen, the former lets entities detect collisions, while the latter lets entities respond to physics.

We've already added these two to the table `AnchorEntity`, but we haven't yet added them to the loot box. Go ahead and update `init()` to add them:

```swift
lootboxTemplate = try! Entity.load(named: "lootbox")
lootboxTemplate.components.set(CollisionComponent(shapes: [.generateBox(width: 0.2, height: 0.13, depth: 0.1)]))

var physicsBodyComponent = PhysicsBodyComponent()
physicsBodyComponent.massProperties.mass = 0.5
physicsBodyComponent.mode = .dynamic
lootboxTemplate.components.set(physicsBodyComponent)
```

Next, whenever we add a loot box, we'll place it a little above the target position and let gravity do the rest. Update the `addLootbox()` method to add a small offset to the position:

```swift
lootbox.position = position + [0, 0.5, 0]
```

Run the app once more - loot boxes should now drop from above and bounce on the table! As a bonus from adding the `CollisionComponent`, you'll also now be able to place loot boxes on top of each other.

## Step 4: Create a LootboxComponent

Now, it's time to make our loot boxes interactive. To do this, we'll need to store some lootbox-specific data along with each entity, like:
* The number of taps it's received
* The number needed to open the box
* The last time its tap count was updated

This calls for a custom component. Go ahead and add a Swift file under the RealityKit folder called `LootboxComponent.swift`, then create a `LootboxComponent` that conforms to RealityKit's `Component` protocol:

```swift
import Foundation
import RealityKit

struct LootboxComponent: Component {
    var tapsReceived: Int = 0 {
        didSet {
            lastUpdate = Date()
        }
    }
    
    let requiredTaps: Int
    
    var lastUpdate: Date?
}
```

We'll need to tell RealityKit about this new component. Head to `LootboxLegends.swift`, then update the app's initializer to register the new component:

```swift
init() {
    LootboxComponent.registerComponent()
}
```

Finally, let's add this component to our loot box template. Update `init()` in `LootboxViewModel.swift` like this:

```swift
lootboxTemplate.components.set(LootboxComponent(requiredTaps: 5))
```

## Step 5: Handle user input

To actually interact with the loot boxes, we'll need to detect when the user taps on them. This is a little harder than a 2D environment -- yet again, you need to somehow convert a 2D tap to a 3D position in the scene. Luckily, we can use hit testing again to find entities at the tap location, then identify the nearest one with a `LootboxComponent`. Start by adding this code to `handleTap()`:

```swift
guard let arView else {
    return
}

let hits = arView.hitTest(position, query: .all)
guard let hit = hits.first(where: { $0.entity.components.has(LootboxComponent.self) }) else {
    return
}
```

Once we've found an entity, we'll need to update its `LootboxComponent` to increment the tap count. Add this code after the hit test:

```swift
hit.entity.components[LootboxComponent.self]!.tapsReceived += 1
```

Finally, we'll check if the loot box has been tapped enough times to open. If it has, we'll remove the loot box from the scene, and we'll tell the view model to present a random item. Add this code after updating the tap count:

```swift
let lootboxComponent: LootboxComponent = hit.entity.components[LootboxComponent.self]!
if lootboxComponent.tapsReceived >= lootboxComponent.requiredTaps {
    hit.entity.removeFromParent()
    currentItem = LootboxItem.items.randomElement()
}
```

Now, when you run the app, you should be able to tap repeatedly on loot boxes to open them and reveal a random item!

## Step 6: Add finishing touches with a LootboxSystem

Our app is almost complete, but there's a few more things we can do to make it more polished. Let's make it so that each loot box expands when tapped, and gradually resets if left alone.

For things like this, it's useful to run code on every frame of the app. We can do this by creating a `System`. Create a new file under the RealityKit folder called `LootboxSystem.swift`, then add a class that conforms to RealityKit's `System` protocol:

```swift
import Foundation
import RealityKit

class LootboxSystem: System {
    static let tapDecayThreshold: TimeInterval = 0.25
    static let scaleFactor: Float = 0.2
    
    required init(scene: Scene) {}
    
    func update(context: SceneUpdateContext) {
        // TODO
    }
}
```

In our `update()` method, we'll start by iterating over all entities with a `LootboxComponent`:

```swift
let query = EntityQuery(where: .has(LootboxComponent.self))
for entity in context.scene.performQuery(query) {
    var lootboxComponent: LootboxComponent = entity.components[LootboxComponent.self]!
}
```

Next, we'll check if we should decrease the loot box's tap count due to inactivity. In the `for` loop, add this code:

```swift
// Check if we need to decay any taps
if let date = lootboxComponent.lastUpdate, date.timeIntervalSinceNow < -Self.tapDecayThreshold, lootboxComponent.tapsReceived > 0 {
    lootboxComponent.tapsReceived -= 1
    entity.components.set(lootboxComponent)
}
```

> [!NOTE]
> We need to call `.set` again because `LootboxComponent` is a struct, so our changes haven't been reflected in the entity yet.

Finally, we'll update the loot box's scale based on the number of taps it's received. Add this code after the decay check:

```swift
// Scale entities according to how many times they've been tapped
entity.scale = SIMD3(repeating: 1 + Self.scaleFactor * Float(lootboxComponent.tapsReceived))
```

There's only one thing left to do: register the system in `LootboxLegends.swift`:

```swift
init() {
    LootboxComponent.registerComponent()
    LootboxSystem.registerComponent()
}
```

Run the app one last time - you should now see loot boxes expand when tapped, and shrink back down if left alone!

## Step 7: Fill in a purpose string

As a final step, don't forget to fill in a camera purpose string in your project's settings! While the app has been running so far, it will be rejected from the App Store if you don't.
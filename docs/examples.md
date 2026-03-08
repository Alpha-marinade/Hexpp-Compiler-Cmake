# Examples

This document contains some practical examples of programs, including what the resulting spell does.

## Example List

1. Mine Block
2. Mine 3x3x3
3. Cast Spell
4. Block-Preserving Explosion
5. Fill Lake
6. Mine Area
7. Teleport Waypoints

## 1. Mine Block

```
void main() {
    // Get the target block by raycasting from self
    let target = block_raycast(self());

    // Check to see if the block is null
    if (target) {
        mine(target);
    }
}
```
This spell is a simple block breaking spell. It will break the block the caster is looking at, but it also will check if there is a block, preventing a mishap in the case of an invalid target.

## 2. Mine 3x3x3
```
void main() {
    // Get the block being looked at
    let target = block_raycast(self());

    // Check if the block is null, and end spell if so
    if (target == null) {
        return;
    }

    // Get the center of the 3x3x3 area using the normal
    let center = target - block_normal_raycast(self());

    // Loop over x from -1 to 1
    let x = -1;
    while (x < 2) {
        // Loop over y from -1 to 1
        let y = -1;
        while (y < 2) {
            // Loop over z from -1 to 1
            let z = -1;
            while (z < 2) {
                // Mine block at the current x, y, z relative to center
                mine(center + vec(x, y, z));
                ++z;
            }
            ++y;
        }
        ++x;
    }

    // Create a light at the center of the 3x3x3
    create_light(center);
}
```
This spell breaks a 3x3x3 of blocks centered on the block behind the block the caster is looking at. It leaves a light in the center of the 3x3x3 so that any tunnels dug with it will be lit.

## 3. Cast Spell
```
void main() {
    // Check if the off-hand can be read from
    if (can_read()) {
        // Execute iota in off-hand
        execute(read());
    }
}
```
This spell is pretty basic, but is important. This one is meant to be imbued into a trinket, and will let you use the trinket to cast the spell in an iota storage in your off-hand. For something this simple, you'll probably want to make the spell without the compiler, though. This version also uses quite a few more patterns than needed. Since the execute is the last thing it does, we don't need to worry about the spell failing after it if we use "execute_unsafe" instead. This means that we could reduce the pattern count by doing so.

you may also notice the check to see if the off-hand can be read in the first place. This prevents a mishap from happening if you use the spell with something other than iota storage.

## 4. Block-Preserving Explosion
```
void main() {
    // Init target variable
    let target = null;

    // See if the caster is looking at an entity
    let entity = entity_raycast(self());

    // If they are, use that as the target
    if (entity) {
        target = entity.pos();
    }
    // If not, then use the block they're looking at
    else {
        target = block_raycast(self());

        // If also not looking at a block, end the spell
        if (target == null) {
            return;
        }

        // Add normal so that we're getting the pos in front of the target
        target += block_normal_raycast(self());
    }

    // Create water to preserve blocks
    create_water(target);

    // Cause explosion at the target with a power of 4
    create_explosion(target, 4);

    // Destroy water
    create_block(target);
    mine(target);
}
```
This spell will explode the target entity if there is one, or just cause an explosion on the ground if not. It includes checks to make sure there is a target. It will also preserve the blocks around the explosion, causing the explosion to leave the ground un-touched, but to still damage entities.

## 5. Fill Lake
```
// Init global list to hold the found blocks
let blocks = [];

void main() {
    // Get the block to start filling from
    let start_block = block_raycast(self());

    // Null check
    if (start_block == null) {
        return;
    }

    // Find all blocks connected to the start that aren't above
    get_blocks(start_block + block_normal_raycast(self()));

    // Fill blocks with water
    let i = blocks.size();
    while (i > 0) {
        --i;
        create_water(blocks[i]);
    }
}

// Get all air blocks below or sideways from the given block and add them to blocks list if not already there
void get_blocks(center) {
    // Add the given block to the blocks list
    blocks = blocks.with(center);

    // Hard-coded list of directions to check for new blocks
    let dirs = [vec(1, 0, 0), vec(-1, 0, 0), vec(0, 0, 1), vec(0, 0, -1), vec(0, -1, 0)];

    // Loop over all directions, backwards because it's more pattern efficient
    let i = dirs.size();
    while (i > 0) {
        --i;
        // Check to see if the block in the current direction isn't in the list and is air, and if so recurse
        check_block(center + dirs[i]);
    }
}

// Check to see if the passed block is 
void check_block(block_to_check) {
    // Check if the block is in the list
    if (blocks.find(block_to_check) == -1) {
        // Use a raycast to see if the block is air
        let ray_result = block_raycast(block_to_check, vec(1, 0, 0));

        // If it is air, recursively add adjacent blocks
        if (ray_result != block_to_check) {
            get_blocks(block_to_check);
        }
    }
}
```
This is a complex program that will fill a hole with water. It takes the block the player is looking at, and fills all blocks beside or below it with water, stopping at any blocks it runs across. Due to the scale of this spell, it's slow to run and may cause a server-side lag spike. Additionally, it has a tendancy to over-fill the stack if used on too large an area. There are a few ways you could mitigate that issue, such as doing a breadth-first algorithm (instead of depth-first) or by doing the recursion layer-by-layer. This would allow the stack to be more clear by going backwards in recursion more often, limiting the recursion depth.

## 6. Mine Area
```
// Save var to hold the first corner position for the break spell
save pos = null;

void main()
{
    // If the first corner is null, the spell should try to grab the first corner
    if (pos == null)
    {
        // If the raycast is null, it will just keep using this branch each time the spell is cast, so nulls are accounted for
        pos = block_raycast(self());

        // Print position to caster so they can see what they grabbed
        print(pos);
    }
    else
    {
        // Grab second position
        let pos2 = block_raycast(self());

        // Print position to caster so they can see what they grabbed
        print(pos2);

        // Null check
        if (pos2 == null)
        {
            return;
        }

        // Loop over x value, from least to greatest
        let x = min(pos.x(), pos2.x());
        while (x <= max(pos.x(), pos2.x()))
        {
            // Loop over y value, from least to greatest
            let y = min(pos.y(), pos2.y());
            while (y <= max(pos.y(), pos2.y()))
            {
                // Loop over z value, from least to greatest
                let z = min(pos.z(), pos2.z());
                while (z <= max(pos.z(), pos2.z()))
                {
                    // Mine current block
                    mine(vec(x, y, z));
                    ++z;
                }
                ++y;
            }
            ++x;
        }

        // Reset save var so the spell can be cast again
        pos = null;
    }
}
```
This is a spell using a save variable. When cast the first time, it will grab the location of the block being looked at and store it in a save variable. When cast the second time, it will grab the location of the block being looked at, and break all blocks in the axis-aligned cuboid defined by it and the previously saved location. Basically, it will break all the blocks in the cube whose corners you were looking at during the first and second casting. After breaking the blocks, it resets itself so it can be cast again.

## 7. Teleport Waypoints
```
// Variable storing the list of teleport positions. Empty to begin with
save tele_positions = [];

// Variable storing the current index of the position list that is selected
save tele_index = 0;

void main() {
    // Check if the caster is crouching
    if (self().height() <= 1.5) {
        // Check if the caster is looking straight down
        if (self().forward() == vec_down()) {
            // If they are, then add the current location as a new teleport point
            tele_positions = tele_positions.with(self().pos());
            print (self().pos());
        }
        else {
            // If not, then cycle to a new index for teleporting
            tele_index++;

            // Wrap index
            if (tele_index >= tele_positions.size()) {
                tele_index = 0;
            }

            // Print new index and destination
            print (tele_index);
            print (tele_positions[tele_index]);
        }
    }
    else {
        // Check to make sure there's a location to teleport to
        if (tele_positions.size() > 0) {
            // Teleport to the position at the current index
            teleport_to(self(), tele_positions[tele_index]);
        }
        else {
            // Print null to indicate a failed operation
            print(null);
        }
    }
}
```
This is a spell that makes use of both save variables and a great spell. In order for this spell to work with Hexagon's building, you'll need to specify teleport's pattern in Hexagon's config file. How to do so is described in the README. This spell allows the saving of multiple teleportation points and lets you teleport to them or add to them at will. By crouching and looking straight down while casting, a new teleport point can be added to the end of the list. By crouching while casting and NOT looking straight down, the selected teleport point will be cycled through. If cast while not crouching, the caster will be teleported to the selected point. This spell does not include any way to remove locations, nor a way to cycle through the list backwards, although such things could be added.
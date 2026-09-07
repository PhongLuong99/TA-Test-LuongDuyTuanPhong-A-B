# Technical Breakdown - Interactive Spell Casting System

## 1. Niagara Systems

### NS_SpellCharge
- Emitter type: GPU Sprite
- Location module: Sphere Location (radius: 50-80 units)
- User Parameters: ChargeIntensity (Float), ChargeColor (Linear Color), Scale Mesh, Scale Sprite
- Color logic: Scratch Pad Lerp Module

### NS_SpellCast
- Emitter type: Ribbon (GPU Sim)
- Trail follows 
- Custom HLSL: distortion trong Material Custom Node (UV panning + noise)
- Particle budget: <500 active

### NS_SpellImpact
- Burst emission tại thời điểm collision (Spawn Burst Instantaneous)
- Mesh Renderer: sphere primitive, scale theo random range
- Kill particles: Location-based kill volume, lifetime 2s
- Event binding: Custom Niagara Event → Blueprint Screen Shake

## 2. Blueprint Architecture

### BP_SpellCaster
- BP_Projectile and BP_Spellcaster
- Add Physics Impulse in Projectile
- Hit box collision
- Input Action bindings: BP_SpellCaster (Started/Ongoing/Completed)

## 3. Level Sequencer

- Camera rig: 
- Export settings: resolution, frame rate, codec dùng cho .mp4
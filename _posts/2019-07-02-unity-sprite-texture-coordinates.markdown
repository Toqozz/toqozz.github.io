---
layout: post
title: "Finding sprite UV/texture coordinates in Unity"
date: 2019-07-02
categories:
---

# Finding sprite UV/texture coordinates in Unity

![Eyedrop demo.](/assets/2019_eyedropper.png)

If you want to do any kind of texture manipulation in games, you'll need some form of texture coordinates.  If you're in 3D, you can obtain UV coordinates of a particular point on a mesh via [raycasting](https://docs.unity3d.com/ScriptReference/RaycastHit.html), but there's no easy way to achieve the same thing for sprite renders.

&nbsp;
Luckily, we can calculate these coordinates ourselves.
The approach here is not very complicated, however, there are many edge cases, and the Unity documentation doesn't exactly make things simple to arrive at a good solution on your own.

# Code
Attach this to each sprite:
```cs
[RequireComponent(typeof(SpriteRenderer))]
public class CoordinateMap : MonoBehaviour {
    private Sprite sprite;

    private void Start() {
        sprite = GetComponent<SpriteRenderer>().sprite;
    }

    public Vector2 TextureSpaceCoord(Vector3 worldPos) {
        float ppu = sprite.pixelsPerUnit;
        
        // Local position on the sprite in pixels.
        Vector2 localPos = transform.InverseTransformPoint(worldPos) * ppu;
        
        // When the sprite is part of an atlas, the rect defines its offset on the texture.
        // When the sprite is not part of an atlas, the rect is the same as the texture (x = 0, y = 0, width = tex.width, ...)
        var texSpacePivot = new Vector2(sprite.rect.x, sprite.rect.y) + sprite.pivot;
        Vector2 texSpaceCoord = texSpacePivot + localPos;

        return texSpaceCoord;
    }
    
    public Vector2 TextureSpaceUV(Vector3 worldPos) {
        Texture2D tex = sprite.texture;
        Vector2 texSpaceCoord = TextureSpaceCoord(worldPos);
        
        // Pixels to UV(0-1) conversion.
        Vector2 uvs = texSpaceCoord;
        uvs.x /= tex.width;
        uvs.y /= tex.height;


        return uvs;
    }
}
```

The magic here is mostly in `Vector2 localPos = transform.InverseTransformPoint(worldPos) * ppu;`.  Here we convert to the sprites coordinate system--which ensures that we match correct coordinates regardless of scale, rotation--and then scale to pixels;

![Eyedropper rotation + scale demo.](/assets/2019_scale_demo.png)

As an example of usage, here's a basic `Eyedropper` script:
```cs
[RequireComponent(typeof(SpriteRenderer))]
public class Eyedropper : MonoBehaviour {
    public CoordinateMap mapper;

    private SpriteRenderer spriteRenderer;
    private Sprite spriteToEyedrop;

    private void Start() {
        spriteRenderer = GetComponent<SpriteRenderer>();
        spriteToEyedrop = mapper.GetComponent<SpriteRenderer>().sprite;
    }

    private void Update() {
        if (Input.GetMouseButton(0)) {
            // NOTE: if your objects aren't at zPos = 0, you'll have to adjust for that.
            Vector2 mouseCoord = Input.mousePosition;
            Vector2 worldPos = Camera.main.ScreenToWorldPoint(mouseCoord);

            Vector2 coords = mapper.TextureSpaceCoord(worldPos);
            //Vector2 coords = mapper.TextureSpaceUV(worldPos);

            Color pixel = spriteToEyedrop.texture.GetPixel((int)coords.x, (int)coords.y);
            //Color pixel = sprite.texture.GetPixelBilinear(coords.x, coords.y);

            spriteRenderer.color = pixel;
        }
    }
}
```

# Notes and Potential Alternatives
Another solution I thought of was to use a 3D raycast from the camera to the sprite, which *should* return correct a correct UV coordinate from the sprite mesh.  This solution doesn't even work (`raycastHit.textureCoord` always returns `(0.0, 0.0)`), and you need to attach a 3D collider to your sprite renderer, which is all kinds of wrong.  You could probably get this to work by replacing your sprite renderers with quads, but you of course lose all associated advantages.

&nbsp;
[Most](https://stackoverflow.com/questions/44143733/unity-correct-sprite-texture-width-height) [solutions](https://gamedev.stackexchange.com/questions/117139/how-to-get-a-pixel-color-from-a-specific-sprite-on-touch-unity) I [found](https://forum.unity.com/threads/uv-texture-coordinates-bounds-using-sprite-packer.400592/) calculate UVs from sprite rect calculations.  Unless you absolutely *never* want to rotate, scale, or flip your sprites, **do not do this**: the calculation will bork.

> Full code and sample scene available at: https://github.com/Toqozz/blog-code/tree/master/sprite_coordinates



































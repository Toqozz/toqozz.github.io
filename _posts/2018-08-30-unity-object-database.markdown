---
layout: post
title: "An object database pattern for Unity3D"
date: 2018-08-30
categories:
---

# An object database pattern for Unity3D
Assigning object references through the Unity inspector is a great tool.  Unfortunately though, it tends to really get in the way of doing code-based object instantiation; in particular, there's no clean Unity-endorsed solution to making simple static classes which utilize game objects.

&nbsp;
When I really need to solve this problem, I've been using a scriptable object database type solution.  I'll tie this post into my [inventory post](https://toqoz.svbtle.com/a-unity-inventory-system-that-actually-works), where we needed a simple and tangle-free way to associate item names with game objects, because raw jsonified object references would change between builds.

# What we want
What we really want here is pretty basic.  We just want a way to do something like `ItemDatabase.GetActual("ruby");` and get back an asset -- be it a prefab, scriptable object, sprite, whatever.  A dictionary of sorts, basically.  It also needs to be able to be accessed without first instantiating something in the scene -- no `MonoBehaviour`.

&nbsp;
The only way to achieve this is of course using `Resources.Load()` in some way.

# Solution
Let's start with a really simple pseudo-dictionary implementation.  I'm just using `switch` here because it's much easier to demonstrate the concept with.  If you're using this kind of pattern for something serious, consider using a [`SerializableDictionary` type](https://github.com/azixMcAze/Unity-SerializableDictionary) (just copy in the `Assets/SerializableDictionary` folder into your project), which will make insertion / deletion far easier.

>Check out [`SoundDatabase.cs`](https://github.com/Toqozz/blog-code/blob/master/database/SoundDatabase.cs) in the [github repo](https://github.com/Toqozz/blog-code/blob/master/database) of this post for an example of a serializable dictionary being used in a solution like this, though I don't recommend actually using it in production.

```cs
// ItemDatabase.cs

[CreateAssetMenu(menuName = "Items/Database", fileName = "ItemDatabase.asset")]
public class ItemDatabase : ScriptableObject {
	// Item objects.
	public Item Ruby;
	public Item Sapphire;
	public Item Emerald;
	public Item Amethyst;
	
    public Item GetActual(string name) {
	    if (string.IsNullOrEmpty(name)) {
		    //Debug.Log("GetActual(): name is null or empty.  You're either checking an empty slot or using this function incorrectly.");
		    return null;
	    }
	    
		switch (name.ToLower()) {
			case "ruby": return Ruby;
			case "sapphire": return Sapphire;
			case "emerald": return Emerald;
			case "amethyst": return Amethyst;
			
            default: 
	            Debug.Log("Could not find an Item for key \"" + name + "\", is it typed correctly?");
	            return null;
		}
	}
}
```

And create the corresponding scriptable object asset in your `Resources` folder:

![Creating the item database](https://imgur.com/F62i2EZ.gif)

Now we can resolve object references from a "pure code" class if we need to.  Going back to our inventory example, we can get reliable item references without needing any "living" game objects:
```cs
// Item.cs

...

[System.Serializable]
public class ItemInstance {
    public string ItemIdentifier;
    public int Quantity = 1;
    public Quality.QualityGrade Quality;
    public bool IsNew;

    private Item _item = null;
    public Item item {
        get {
            if (_item == null) {
                _item = ((ItemDatabase) Resources.Load("ItemDatabase")).GetActual(ItemIdentifier);
            }

            return _item;
       }
    }

    public ItemInstance(string itemName, int quantity, Quality.QualityGrade quality, bool isNew) {
        this.ItemIdentifier = itemName;
        this.Quantity = quantity;
        this.Quality = quality;
        this.IsNew = isNew;
    }
}
```

# Take Note
We now have a tangle free way to retrieve game object references.  `Resources.Load()` is an excellent tool to know about, but make no mistake, assigning objects to `MonoBehaviour` variables is definitely the preferred solution *most of the time*.  Doing so allows Unity to perform some space-wise optimizations and not include any unused assets in your builds.  When we abandon this system, Unity is forced to include all assets in the  `Resources/` folder.  *Note:* this includes objects that your database links to.

&nbsp;
Some people really resent any use of the `Resources/` folder.  I feel as though this resentment is a little misguided sometimes.  To my knowledge, the worst thing about using the `Resources/` folder is that it potentially makes your build a bit bigger.  This seems like a really insignificant trade-off when compared to the benefits that using it can bring--`MonoBehaviour`-only type problem solving can seriously get in the way of elegant code-based solutions.  For these scenarios, I think it's well worthwhile to make good use of `Resources.Load()`.

&nbsp;
To avoid any confusion, when I say `MonoBehaviour`-only type problem solving, I mean avoiding all use of the `Resources/` folder and strictly assigning objects in the inspector (drag and drop).


&nbsp;
&nbsp;
A further point to note is that you probably shouldn't use something like this for huge databases.  Loading all those assets into memory can take time.  A solution might be to load the database once using an Init() method when the game starts.



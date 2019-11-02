---
layout: post
title: "Object pooling in Unity"
date: 2019-11-02
categories:
---

# Object pooling in Unity
![Demo gif](/assets/2019_projectile_spam.gif)

[`Instantiate()`](https://docs.unity3d.com/ScriptReference/Object.Instantiate.html) is expensive.  If possible, you should never use it at runtime.  For one-offs this is usually done by spawning whatever objects you need in `Awake()`, and then calling `SetActive(true)` when you need them.  For more than one-offs, you probably want to use a pool.
A pool is a group of objects that you instantiate at some convenient time (probably on scene load), and then later on, when you want to spawn an object, you grab it from the pool instead of making it fresh.  This technique is used extensively throughout just about any well-programmed game.

> A good rule is to never call `Instantiate()` when the player has control.

# A dead simple pool
Here's a simple, performant pool you can use:

```cs
public class Pool : MonoBehaviour {
    [Serializable] // Let this appear in the inspector.
    public class ObjectPool {
        public int amount;
        public PooledObject objectToPool;
    }

    // Singleton boilerplate.
    private static Pool _instance;
    public static Pool Instance {
        get {
            if (!_instance) {
                _instance = FindObjectOfType<Pool>();
            }

            return _instance;
        }
    }

    // List of objects to pool (only used for instantiation)
    public List<ObjectPool> objectPools;

    // Pool of objects, indexed by instance ID.
    // A queue works pretty naturally here.
    private Dictionary<int, Queue<PooledObject>> pool;

    private void Awake() {
        // Spawn all objects in provided pools.
        pool = new Dictionary<int, Queue<PooledObject>>();
        foreach (ObjectPool objPool in objectPools) {
            int amount = objPool.amount;
            PooledObject obj = objPool.objectToPool;

            // Saved prefabs have an instance id, which we can use to talk about the same prefab from other scripts.
            int id = obj.GetInstanceID();
            Queue<PooledObject> queue = new Queue<PooledObject>(amount);
            for (int i = 0; i < amount; i++) {
                var clone = Instantiate(obj, transform);
                clone.id = id;
                clone.Finished += ReQueue;      // When `Finish()` is called, we put our object back in the queue.
                clone.gameObject.SetActive(false);
                queue.Enqueue(clone);
            }

            pool.Add(id, queue);
        }
    }

    private PooledObject GetNextObject(PooledObject obj) {
        // @NOTE: create a new queue if none exists, for pooling "unplanned" objects?
        var queue = pool[obj.GetInstanceID()];
        PooledObject clone = null;
        // If queue is empty (has been exhausted -- the pool size was too small), extend the queue by instantiating a new object,
        // and add it to the future queue.
        if (queue.Count == 0) {
            Debug.LogWarning("Object Pool queue was empty; wasn't able to get a new pooled object, so one will be instatiated.");
            clone = Instantiate(obj, transform);
            clone.id = obj.GetInstanceID();
            clone.Finished += ReQueue;
            clone.gameObject.SetActive(false);
        } else {
            clone = queue.Dequeue();
        }

        return clone;
    }

    // Gets an object from the pool and returns it after setting position, rotation, and active.
    public PooledObject Spawn(PooledObject obj, Vector3 position, Quaternion rotation) {
        var clone = GetNextObject(obj);
        clone.transform.position = position;
        clone.transform.rotation = rotation;
        clone.gameObject.SetActive(true);
        return clone;
    }

    private void ReQueue(PooledObject obj) {
        // Hide object and insert back in queue for reuse.
        obj.gameObject.SetActive(false);
        var queue = pool[obj.id];
        queue.Enqueue(obj);
    }
}
```

`PooledObject` is teeny tiny:

```cs
public class PooledObject : MonoBehaviour {
    [HideInInspector]
    public int id;
    public Action<PooledObject> Finished;

    // A component reference for fast access -- avoids calls to GetComponent<>().
    public Component behaviour;

    public T As<T>() where T: Component {
        return behaviour as T;
    }

    public void Finish() {
        if (Finished != null) {
            Finished(this);
        }
    }

    // Convenience method to call finish when particles finish.
    // Needs ParticleSystem stop action to be set to "Callback".
    private void OnParticleSystemStopped() {
        Finish();
    }
}
```

## Usage
Usage is as straightforward as possible.  Create a new Game Object in your scene and attach the `Pool` script.  Then, attach the `PooledObject` script to any prefabs that you want to be pooled, and drag them into the object pools list, with a best-guess for how many will be used concurrently:

> If you exceed the amount in the object pool, new ones will be spawned with `Instantiate()` rather than failing.

![Setup process demonstration](/assets/2019_create_pool.gif)


Then, instead of calling `Instatiate()` and `Destroy()`, we call `Pool.Spawn()` and `PooledObject.Finish()`:
```cs
public class ProjectileSpawner : MonoBehaviour {
    public float spawnRate = 0.1f;
++  public PooledObject projectile;
--  public Projectile projectile;

    private float timer = 0f;

    private void Update() {
        timer += Time.deltaTime;
        if (timer > spawnRate) {
            timer -= spawnRate;

            // Spawn object with random 2D rotation.
++          PooledObject instance =
++              Pool.Instance.Spawn(projectile, transform.position, Quaternion.Euler(0f, 0f, Random.Range(0f, 360f)));
            // We can avoid GetComponent<>() for a frequently accessed component, which is nice.
++          instance.As<Projectile>().speed = Range.Range(.5f, 1f);

--          Projectile instance =
--              Instantiate(projectile, transform.position, Quaternion.Euler(0f, 0f, Random.Range(0f, 360f)));
--          instance.speed = Random.Range(.5f, 1f);
        }
    }
}
```

```cs
public class Projectile : MonoBehaviour {
    ...
    private void OnTriggerEnter2D(Collider2D other) {
        ...

++      GetComponent<PooledObject>().Finish();
--      Destroy(gameObject);
    }
}
```

> It would be better to cache the `PooledObject` component.  `GetComponent<>()` has been used here for simplicity.

![Projectile spawner with pool](/assets/2019_projectile_spawner.gif)

# `Instantiate()` doesn't care, but `SetActive()` does
The really nice thing about using `Instatiate()` and `Destroy()` is that you don't have to worry about any kind of previous state on the object -- everything is new and fresh.  If you're disabling and re-enabling objects (like in a pool), you **do** have to pay attention to object state; particle progress, animations, and any variables changed on components can all trip you up.  There are ways you could reset components to fresh (serialization), but if you want to keep your performance intact, this is just something that you're going to have to eat, sorry.

---

# The numbers
What good is an optimization without running the numbers?

I'm testing this with a modified version of the above `ProjectileSpawner` code.  To avoid variance, I've removed the randomness and some other stuff;
```cs
public class BenchmarkProjectileSpawner: MonoBehaviour {
    public float delay = 5f;
    public int amount = 5000;

    // Pool variant.
    public PooledObject projectile;
    -------------------------------
    // Instantiate variant.
    public GameObject projectile;

    private float timer = 0f;

    // These probably allocate, so cache them for benchmarking.
    private Vector3 position = Vector3.zero;
    private Quaternion rotation = Quaternion.identity;

    // Update is called once per frame
    private void Update() {
        timer += Time.deltaTime;
        if (timer > delay) {
            // Only fire once.
            timer = -Mathf.Infinity;
            for (int i = 0; i < amount; i++) {
                // Pool variant.
                Pool.Instance.Spawn(projectile, position, rotation);
                -----------------------------------------------------------
                // Instantiate variant.
                Instantiate(projectile, position, rotation);
            }
        }
    }
}
```

```cs
public class BenchmarkProjectile : MonoBehaviour {
    public float speed = 1f;

    private PooledObject pooledObject;

    private float timer = 0f;
    private Vector2 up = Vector2.up;

    private void Awake() {
        pooledObject = GetComponent<PooledObject>();
    }

    private void Update() {
        transform.Translate(up * speed * Time.deltaTime);

        timer += Time.deltaTime;
        if (timer > 2f) {
            timer = 0f;
            // Pool variant.
            pooledObject.Finish();
            ------------------------
            // Instantiate variant.
            Destroy(gameObject);
        }
    }
}
```

The test is pretty basic.  We spawn 5,000 projectiles with either `Instantiate()` or `Pool.Spawn()`, and observe the results through the built-in profiler.  We'll also have a look at differences in destruction times between the two approaches.

These results aren't on-the-nose accurate -- I'm running in editor, and I'm only eyeballing the variance.  They're more than enough, however, to get the point across;

![Profiler when running with Instantiate](/assets/2019_instantiate_5k_profiler.gif)
*`Instatiate()`/`Destroy()` profiler*

![Profiler when running with object pools](/assets/2019_pool_5k_profiler.gif)
*Object pool profiler*

The results are in!

|             | Instantiate | Pool     |
| ----------- | ----------- | -------- |
| **Spawn**   | ~206.37ms   | ~33.72ms |
| **Despawn** | ~44.61ms    | ~17.69ms |

Spawning 5,000 projectiles was enough to make us drop a couple of frames even with our object pool.  With `Instantiate()`, it's closer to a freeze.  When spawning lots of objects in one place, you probably *always* want to split the operation over multiple frames, regardless of how optimized your architecture is.  Even a small amount of stutter in games is horrible and really goes against "game feel" -- even if players can't point out why, they'll feel uncomfortable playing your game.

If you're thinking: "Why would I ever want to spawn thousands of objects at once?  `Instantiate()` is good enough.", I have a few things to say:
First of all, you might not spawn that many objects from any *one place* at the same time, but as your game gets bigger you can end up spawning a significant number of objects at the same time either by coincidence, or because an in-game event triggers a bunch of things at once; e.g. AI enemies shooting at a player.
Secondly, the objects I've used in this benchmark are pretty much as simple as they come -- only a few (basic) components and no children.  Add a couple `Rigidbody`s, character animations, and child objects, and the number of things you can spawn without players feeling anything drops **hard**.
Lastly, keep in mind that PC (where this benchmark was run) is probably the fastest platform you'll be running your game on.  Spawning even a couple of fat objects on mobile can be enough to cause a stutter.

An object pool significantly increases your headroom for spawning lots of objects at runtime, and is almost as convenient as instantiation.  If you're calling `Instantiate()`, chances are that it can be replaced with an object pool.

> Source code and Unity project available at [https://github.com/Toqozz/blog-code/tree/master/object_pool](https://github.com/Toqozz/blog-code/tree/master/object_pool).

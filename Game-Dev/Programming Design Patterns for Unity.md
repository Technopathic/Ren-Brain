## The Observer Pattern

The observer pattern's goal is to de-couple tightly coupled methods. For example, when you have a situation where a "Rampage" status of a character is active, this needs to update the Rampage UI (to show a red screen for example), affect the enemy AI somehow, and may affect the character's HP bar as well. 

There are two general approaches with this, 1) To have a Rampage class which calls each of these other classes to update them with the Rampage active boolean, or 2) to have each of these classes check the Rampage status of the active character on every frame. The first method is highly coupled while the second method is costly on resources. 

We instead use the observer pattern which uses a sub-pub pattern. A character registers an observer, can unregister an observer, and notify an observer. After registration, the character can then notify the observer when rampage is active. The observer has an update function which then fires to the "ConcreteObservers" which are the classes that need to be updated (enemy AI, UI, HP Bar, etc). When the rampage character is no longer in existence, the unregisterObserver function should be called.

## Observers with UnityEvents

You can add a SerializedField with a type of UnityEvent eventName, which does allow you to use an invoke function to trigger when the UnityEvent eventName should be called. Then in the Unity Editor, since it is a SerializedField, you can assign a callback (another function to be called), whenever eventName is triggered. 

An easy to understand example of using this would be resetting a character's health stat whenever their character levels up. Whenever a character levels up, then we invoke the UnityEvent onLevelUp, which then has resetHealth as a callback.

## Delegates, Actions, and Events

One of the issues with using UnityEvent is that it exposes a lot of functions which shouldn't be accessible unless needed. For example, when using `GetComponent<Level>().onLevelUp.(...)` you're provided with many events that are dangerous to have exposed outside of the Component. 

Instead, we can use an event with type Action such as `public event Action onLevelUpAction` which can be called like a method, `onLevelUpAction()` . This will give an error unless there is a listener, so be sure to wrap this in an if statement which checks for non-null `if (onLevelUpAction != null) {...}` 

In the class, which will be subscribing to our onLevelUpAction, we will use the `OnEnable()` and `OnDisable()` methods from Unity (because we may not want the function invoked / removed if the class is not active). We then add / remove a function as a subscribed function.  Be sure to not use the `()` for the subscribing function, otherwise it will call the function immediately.
`GetComponent<Level>().onLevelUpAction += ResetHealth;`
`GetComponent<Level>().onLevelUpAction -= ResetHealth;`

Using a type Action is a simple way of doing a delegate type. Using a delegate type looks like:
```
public delegate void CallbackType();

public event CallbackType onLevelUpAction;
```

The effect is exactly the same. A note to make is that the level of accessibility (public, private, protected) needs to be the same. A delegate could have parameters (such as `float level`) where the subscribing function (`resetHealth()` in this case) also takes a parameter. 

Using Action is just fine instead of using delegates. If you do want to pass an argument with Action, you can use a generic format `Action<float>` for example, where the `resetHealth()` AND the `onLevelUpAction()` would need a paramter.

## The Singleton (Anti?) Pattern

When you have some sort of object that you only want it to load once. Such as an audio player, since there's only one audio instance / speaker. You create a static instance of the audio player itself within the class. A private static method instance needs to be used where it points to itself. 
```
public class AudioPlayer {
    private static AudioPLayer instance = null;
    public static AudioPlayer GetInstance() {
        if(instance == null) {
            instance = new AudioPlayer();
        }
        return instance;
    }
}
```

In order so that the nobody else is able to create another instance of an audio player, outside of this class, we have the default constructor as private 
```
private AudioPlayer() {}
```

Singleton doesn't pollute the namespace, but statics don't either. 
They provide encapsulation, but so do statics. 
Allows for lazy initialization, which prevents initialization until we needs it. 

It's only slightly better than statics, and is still global state, which is why most people consider it an anti-pattern.
You can access it from anywhere. It violates rule three of doing more than one things. This class not only does it's usual job, it also ensures itself only is initialized once. 

But in Unity, we often want to do something like this because want code that has a large scope than the scene. Everything is in monobehaviours, and we and want those monobehaviours to travel between scenes with us.

If you would like to use singletons with monobehaviours, the way to do so would be:
```
public class AudioPlayerBehaviour : MonoBehaviour {
    private static AudioPlayerBehaviour instance = null;
    
    private void Awake() {
      if(instance == null) {
        instance = this;
        DontDestroyOnLoad(gameObject);
      } else if (instance != this) {
        Destroy(gameObject);
      }
}
```

It's recommended not to use since it violates most of the best coding practices, but sometimes it is unavoidable.

## Better than a Singleton

We need a way to make an object persistent across scenes. To do this, we can introduce a "Persistent Object Spawner" which has a single boolean, saying whether we have spawned or not. This class spawns a prefab of our choosing, unless that prefab has already spawned. 
In scene, we will create an empty called "ObjectSpawner", we then drag the persistentObjectSpawner class to the ObjectSpawner emtpy. We then create another empty prefab called "PersistentObjects" and put our Singleton object into it. Then we drag the PersistentObjects into our asset browser and delete the original PersistentObjects prefab from the scene. We then add the PersistentObjects as the persistent prefab to the ObjectSpawner prefab. 

Here is an example of a PersistentObjectSpawner class:
```
using UnityEngine;

public class PersistentObjectSpawner : MonoBehaviour
{    
    [Tooltip("This prefab will only be spawned once and persisted between scenes.")]
    [SerializedField] GameObject persistentObjectPrefab = null;

    static bool hasSpawned = false;

    private void Awake()
    {
        if (hasSpawned) return;

        SpawnPersistentObjects();
        hasSpawned = true;
    }

    private void SpawnPersistentObjects()
    {
        GameObject persistentObject = Instantiate(persistentObjectPrefab);
        DontDestroyOnLoad(persistentObject);
    }
}
```

## Finite State Machines

An enumeration of various states (instances of existence?) of being in the scene. For example a character can be in a crouching, standing, or jumping state. States can be transitioned between each other using functions (actions). 
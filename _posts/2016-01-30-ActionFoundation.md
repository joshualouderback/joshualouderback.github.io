---
layout: post
title: Foundation of Actions! (Part 1)
excerpt: "Actions and Coroutines!"
modified: 2016-01-30
tags: [Unity, Action, Actions, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

###What are _Actions_?

An action can be thought of as an objective. For example say you want to move an asteroid from a to b, that simply is an action. What if that position b is the player's ship and we want to warn the players a "Warning Dialogue"? Now say we want the lights in the room the player is in to fade to black and once the room goes dark follow it with flashing red lights and alarm sounds. All of these things can simply be represented with actions to make scripting events like these simplier and faster than ever before. And I am going to show you how, so remember this scripted event because I will be referencing it throughout.

>

_*Note\: that this entire article is based on using C# and Unity 5, but the same techniques can be applied to other engines with some work._

---

### Base Action

An action is too generic to describe, but all actions will have a lot of the same under the hood. These are things like\:

* Update-able, but they will most likely update differently.
* Can be paused and resumed.
* Or they can even be outright canceled.
* They manage themselves, so they need to keep track of themselves.
* Actions are hierarchical, so they look to their parents for info.

So in that case we need an abstract class that our all our different actions can build from. I also wanted to give you some context to what an action is comprised of, since everything builds from here on out. 

>

_*Note\: The code blocks are formmated to condense minor sections and reduce page length._

{% highlight c# %}
public abstract class Action
{
  protected bool paused_ = false;       // Actions can be paused
  protected bool running_ = false;      // Need to know when an action is running
  protected bool completed_ = false;    // Need to keep track of when the action is done
  protected Action parent_ = null;      // We need to know our parent for extra info
  protected Coroutine routine_ = null;  // They keep track of their own routine
  protected Actions target_ = null;     // Actions are tied to gameObjects

  // Constructor
  public Action() {}

  /////// Public Methods /////// 
  // Since actions manage themselves they look to their parent for most things
  public void SetParent(Action parent)
  {
    // Set our parent and our target to our parents target
    parent_ = parent;
    target_ = parent_.target_;
  }
  // Since actions manage themselves they start themselves
  public Coroutine StartAction()
  {
    // Set our routine, because we manage ourselves
    routine_ = target_.StartCoroutine(Update());
    // Return the routine, just in case others want to yield on us
    return routine_;
  }

  ///////  Settors /////// 
  public void Pause() { parent_.paused_ = true; }
  public void Resume() { parent_.paused_ = false; }

  /////// Gettors /////// 
  public virtual bool IsPaused() { return parent_.IsPaused(); }
  public bool IsCompleted() { return completed_; }
  public bool IsRunning() { return running_; }

  /////// Virtual methods /////// 
  // Every action updates differently
  public virtual IEnumerator Update() { yield break; }
  // Some actions may cancel differently
  public virtual void Cancel()
  {
    if(routine_ != null)
      target_.StopCoroutine(routine_);
  }
}
{% endhighlight %} 


---

###Coroutines\:

Before we can dive into how each type of action works, we are going to need to understand coroutines. Unity defines it as a function that can suspend its execution, this is known as yielding, until the given YieldInstruction is finished. Since a coroutine can suspend its execution, it means you can have that function run over multiple frames. Coroutines are the foundation of actions. Let us look look at our example of an asteroid traveling from a to b, normally we would write the component like this:

{% highlight c# %}
public class AsteroidMover : MonoBehaviour {
  public Transform startMarker;
  public Transform endMarker;
  public float duration = 5.0F;
  private float currTime = 0.0F;

  // Every frame let us try to lerp
  void Update() {
    // Calculate our time step
    float t = currTime / duration;
    if(t <= 1.0F) // Only interpolate until we reach the end position
      transform.position = Vector3.Lerp(startMarker.position, endMarker.position, t);
    currTime += Time.deltaTime;        
  }
}
{% endhighlight %}  

There are a few annoyances that come up\:

* Once this is done lerping we are wasting an update call, unless we destroyed this object or component.
* How will we tell this object to start interpolating if it isn't suppose to start right away?
* If we want something to happen next, we need some way of communicating this is done and that it only tells it once.

Now trying to do this simple thing ends up creating little problems that we are going to have to handle. Well this is why actions are awesome because they will remove these little problems and let you do these simple things without having to think about all those little problems. The first step in solving these problems are by utilizing coroutines. Coroutines solve our first problem for free due to their nature of yielding until they stop being told to yield. Now let us look how we can modify our class with coroutine to solve this problem:


{% highlight c# %}
public class AsteroidMover : MonoBehaviour {
  public Transform startMarker;
  public Transform endMarker;
  public float duration = 5.0F;
  private float currTime = 0.0F;

  // We can change our update function into a coroutine
  IEnumerator Move() {
    // Everytime we hit the yield statement, on the next frame 
    // we will start at top of while loop and check it again
    while(curr < duration)
    {
      // Our lerp code stays the same
      float t = currTime / duration;
      transform.position = Vector3.Lerp(startMarker.position, endMarker.position, t);
      currTime += Time.deltaTime;
      // Now we want to yiaction workeld until the next frame
      yield return null;  
    }
    // Once we pass the while loop, this function won't be called anymore
  }
}
{% endhighlight %} 

Look at that! We barely had to rewrite our code to fix that problem. Now the only major roadblock left is how do we tell this coroutine to start and how do we tell the following action to start after this one? If you think about our problem all we are trying to do is create an order of actions. We are building a _sequence_!

>

[Continue to part 2.](http://joshualouderback.com/ActionManagement/)

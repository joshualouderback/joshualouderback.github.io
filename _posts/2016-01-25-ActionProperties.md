---
layout: post
title: Action Properties (Part 5)
excerpt: "The Journey To Action Properities"
modified: 2016-01-30
tags: [Unity, Action, Actions, Action Property, ActionProperty, Action Properties, ActionProperties, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### What Are Action Properties? ###

Do you remember in the very first article when I talked about having an asteroid move from a to b? Well this is where ActionProperties come into play! You may be thinking since we already have Action Routines we can do that easily, so then why would we need another system for that? Well the simple answer is that we use interpolation a lot in game programming, so why not have a really nice system wrap it? Think about it, we interpolate color changes, positions of objects, rotations of objects, animations, lights and so much more! The name Action Property comes from the fact that we are manipulating a specific property of an object through an Action. 

### The Struggle ###

At the heart of Action Properties are two things:

1. Interpolation
2. Property Of Object

As simple as these two things are, they sure did make my life hard. In order to accomplish an Action Property, I needed to make an Action that can interpolate over a number of frames to any property given. Coming from C++, my instinctual reaction is to just make interpolate a generic function. But first let us just quickly look at the math behind interpolation, so you can fully understand the problem.

#### Interpolation: ####

Interpolation is simply just obtaining a series of "points" in between two values. "Points" for example are: all the points along a line between (0,0) and (1,1), or colors values from red to blue. There are many more examples, but what all of these representations boil down to is a value between 0 (start value) and 1 (end value). Interpolation can be written in many different forms, but the one we are going to use is:

>

```  
General Form:
Start + (End - Start) * t
```

>

Now, in this equation, t is the value between 0 and 1. You can see if we plug in 0, we get just the start value and if we plugged in 1, we would be left with just the end value. To get any values in between start and end, we simply create a number between 0 and 1. There are plenty of different ways you can interpolate between two values and our ActionProperties will support that, but for now, we are just going to look at linear interpolation. 

>

```
Linear Interpolation:
t = currentTime / duration
```

>

#### Back To It: ####

Since we got that out of the way, we can look at our problem. In order to interpolate a property it needs to be able to add, subtract and scale. In C++ we would just make this a generic function and the compiler will error if the user passes a type that doesn't support the +, - and * operators. Well the problem is, C# doesn't allow us to use operators on generics and there goes our simplest and most intuitive solution... well almost! When researching subtracting generic types, I found about constraints. Constraints restrict generics to specific types. Someone suggested building an interface that requires the implementation of Add, Subtract and Scale functions and restricting the generics in Action Properties to those. It basicaly gives us the same thing as C++, but rather than the type having to overload those operators they just need to fill out the interface. 

>

Now that we have a new solution, I went about trying to implement it and not to my surprise I hit another roadblock. The problem is that in order to use an interface the class has to inherit from it, which means we can't use any of the types in Unity (colors, vectors, matrices, and quaternions). So again, back to the drawing board. I mean I could of tried to wrap the classes, but that probably would of been ugly. I could of even rewritten all of them, but then you (reading this) would have to go do that as well. Both of those options did not sound very appealing to me, so yeah, back to the drawing board...

>

Then, I came across extension methods and thought maybe I could utilize them. Soon I came to realize these weren't going to help either. Since I can't constrain these classes to anything, at least to my knowledge, to ensure that these extension methods exist, I am out of luck. I even tried seeing if you could extend a class by an interface and that doesn't even exist. Why doesn't that exist (If you know you should let me know)? It would be such a nice feature! Once again stopped dead in my tracks and all I could say is why C#! Why?!!!!!!! 

#### Finish Line: ####

After trying so many other things that I can't even remember, I came across another idea... What about template specialization? I could write a generic ActionProperty class and just specialize each of the types I want. When specializing, I could write functions to subtract, add and scale each of those types and it would solve all my problems. Well, guess what?! C# doesn't have template specialization either! When trying to figure out how to do template specialization and finding out that it doesn't exist, I came across one thing: "C# doesn't want you to work with specialization except polymorphism". That is when I finally hit the idea, what if I just write the base generic ActionProperty class and inherit a specialized type of that class. Like so:

{% highlight c# %} 
// Here is the base generic class, marked as abstract because it needs specialization
public abstract class ActionProperty<T> : Action { ... }

// Inheriting a specialized type of that class
public class ActionPropertyVec4 : ActionProperty<Vector4> { ... }
{% endhighlight %} 

And wah lah! We got it! It works! Woo! And so, I thought I made it, but there still was one last problem... references. Ideally, with an ActionProperty, we would just love to pass in a reference to the value we want to modify and store it. You might be wondering, how did you have a problem with references? Yeah, it was a surprise to me too. Apparently, Unity does not allow you to pass references to the internals of their types. The prime example is moving that object from a to b, because all we need is the position of the object. Ideally, we would just pass a reference to the objects position (gameObjects.transform.position), but Unity does not allow that. There were two approaches I saw from here. The first, is to just take in the reference to the transform and build support of knowing what internal I should grab in the transform. For example, letting the user pass in an enumeration value that I defer to, in order to select the correct internal. The second, was building a callback function that allows the user to define how they want to set their values. I went with the second approach, because I felt like it would be more extendable to the user. 

### Base Class\: ###

Here it is, in all its glory! I would of loved to make this not as redudant, so if anyone comes across any better solutions, let me know. I would love to hear them! 

{% highlight c# %} 
public abstract class ActionProperty<T> : Action
{
  public delegate void Settor(T newValue);              // Format of our settor delegate
  protected EasesAndCurves.EaseFunction easeFunction_;	// Reference to the ease function for type of interpolation
  protected float duration_;	                          // How long the user wants to interpolate over
  protected T startValue_;                              // The start value
  protected T change_;                                  // The change we interpolate with
  protected Settor settor_;                             // Reference to the settor to call
  
  // Our abstract functions that need to be overrided to support generic update code
  public abstract T Add(T interpolated, T start);
  public abstract T Scale(float scalar, T difference);

  // Since all properties update the same, why make users override this function in each
  // implementation when they will just be copying and pasting code
  public override IEnumerator Update ()
  {
    // Set the current time to duration and set as running
    float currentTime = 0;
    running_ = true;

    // While we have time, update once per frame
    while(currentTime < duration_)
    {
      // If not paused, update
      if(!IsPaused())
      {
        // Calculate the ease interpolation factor
        var easeFactor = easeFunction_(currentTime, duration_);

        // Interpolate with ease factor and use callback to set it
        settor_(Add(Scale(easeFactor, change_), startValue_)); 

        // Update frame time, yield till next frame
        currentTime += Time.deltaTime;
      }
      yield return null;
    }

    // Set the position as exactly the last position, just in case of floating point error
    settor_(Add(change_, startValue_));
    // Mark as completed, then break yielding
    completed_ = true;
    yield break;
  }
}
{% endhighlight %} 

### Extension Example\: ###

Here is an example of one of the ActionProperty extension classes. To make any other one it is just copy and paste, then just change the types. The only work you may have to do is subtraction, addition and scaling if your class doesn't already support those operators.

{% highlight c# %} 
public class ActionPropertyVec4 : ActionProperty<Vector4>
{
  public ActionPropertyVec4(ActionManager manager, Vector4 startValue, Vector4 endValue, float duration, EasesAndCurves.Eases ease, 
                            Settor settor)
  {
    // Set the starting value
    startValue_ = startValue;
    // Since the change never changes... we can calculate it once rather than every frame
    change_ = endValue - startValue;
    // Set the duration of the interpolation
    duration_ = duration;
    // Set the reference to the settor
    settor_ = settor;
    // Grab the ease function from our ease class
    easeFunction_ = new EasesAndCurves().GetEase(ease);
    // Set our parent and add ourselves to them
    SetParent(manager);
    AddAction(this);
  }

  // Override two simple functions, rather than copy paste code
  public override Vector4 Add(Vector4 interpolated, Vector4 start)
  {
    return interpolated + start;
  }
  public override Vector4 Scale(float scalar, Vector4 difference)
  {
    return scalar * difference;
  }
}
{% endhighlight %} 

### Ease Class\: ###

My ease class is much larger than the one below, but I don't want to balloon this article too much.

{% highlight c# %} 
public class EasesAndCurves 
{
  //All the various ease types.
  public enum Eases
  {
    Linear,
    QuadIn,
    QuadInOut,
    QuadOut,
    // ... much more ease types
  };

  public delegate float EaseFunction(float currentTime, float duration);

  public EaseFunction GetEase(Eases easeType)
  {
    // Grab the function for caculating the ease
    switch(easeType)
    {
    case Eases.Linear:
      return Linear;
    case Eases.QuadIn:
      return QuadIn;
    case Eases.QuadInOut:
      return QuadInOut;
    case Eases.QuadOut:
      return QuadOut;

    // ... so many other types

    default:
    return Linear;
    }
  }

  public float Linear(float currentTime, float duration)
  {
    return currentTime / duration;
  }

  public float QuadIn(float currentTime, float duration)
  {
    currentTime /= duration;

    return currentTime * currentTime;
  }

  public float QuadOut(float currentTime, float duration)
  {
    currentTime /= duration;
    return -1 * currentTime * (currentTime - 2);
  }

  public float QuadInOut(float currentTime, float duration)
  {
    currentTime /= duration / 2;

    if (currentTime < 1)
    {
      return (0.5f * currentTime * currentTime);
    }

    currentTime -= 1;
    return -0.5f * (currentTime * (currentTime - 2) - 1);
  }

  // ... again ease functions here
}
{% endhighlight %} 

### Sample\: ###

Once I get the time, I will put up a full project on github with the ActionSystem and the example I stated at the beginning of the article. For now, here is a little sample, so you can see general syntax of using it.

{% highlight c# %} 
public class ActionTest : MonoBehaviour {
  // Store seq, so we can pause it later
  ActionSequence seq;

  // Callback for settors
  void SetPosition(Vector3 set) { this.transform.position = set; }
  void SetColor(Color set) { mat.color = set; }

  // We want to create action in start, because we don't want to create this every frame
  // Example of general setup of Actions
  void Start () {
    // Get material and set it to blue, not checking for null but should
    mat = this.gameObject.GetComponent<MeshRenderer>().material;
    mat.color = Color.blue;
    // Create the sequence and store it, so we can add to it
    seq = new ActionSequence(this.gameObject);
    // Create group and store it, so we can add to it
    ActionGroup grp = new ActionGroup(seq);
    // Move object up while changing its color
    new ActionPropertyVec3(grp, this.transform.position, 
                           this.transform.position + new Vector3(0, 10, 0), 2.0f,
                           EasesAndCurves.Eases.Linear, this.SetPosition);
    new ActionPropertyColor(grp, mat.color, Color.red, 2.0f, 
                            EasesAndCurves.Eases.Linear, this.SetColor);
  }
  
  // Example of pausing and cancelling
  void Update () {
    // If press space we pause or resume it
    if(Input.GetKeyDown(KeyCode.Space)) 
    {
      if(seq.IsPaused())
        seq.Resume();
      else
        seq.Pause();
    }
    // Otherwise if press e, cancel entire action
    else if(Input.GetKeyDown(KeyCode.E))
    {
      seq.Cancel();
    }
}
{% endhighlight %} 

### Conclusion\: ###

Writing this Action system was really fun and helped expand my knowledge of C# even further. I highly recommend implementing an Action system in your engine, because developing content is faster and less error prone. It is really simple to use and to teach. Getting any new members up to speed with this system will take no time at all! Writing this article actually helped me find bugs in the system and build an even deeper understanding of C#. So I highly recommend spending some time writing about your own work, if you don't do so already. It was a great learning experience. Also during the process of writing this article I wanted to be as full disclosure as possible, so that you come out with a very strong foundation to look back to. If you come across any unclarity, let me know so I can update the article (if needed) or answer your questions. Since this is such a great tool, I don't mind if people just grab it and go. All I ask is for you credit me and my work. If you find any bugs, or new solutions let me know! I am always striving to improve the system and my understanding. Lastly, send me the cool stuff you do with this! I'd love to see how you use it and hope to learn techniques I haven't even thought of.


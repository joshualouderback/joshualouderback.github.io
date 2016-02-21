---
layout: post
title: Action Delays, Calls, And Routines (Part 4)
excerpt: "How To Add Action Calls, Delays and Routines!"
modified: 2016-01-30
tags: [Unity, Action, Actions, Action Call, ActionCall, Action Delay, ActionDelay, Action Routine, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### All The Actions ###

Now that we have everything we need to manage flow of our Actions, we need the Actions that actually let us do actions! The toolkit of Actions are really simple and that is what so great about Actions. We have Action Delays, which simply just waits for a specified time. Next is Action Calls, and they just call a single frame function. Then there are Action Routines, just in case we want a multi-frame function and why wouldn't we?! Coroutines are awesome! Now let's get on with it!

### Action Delay\: ###

Action Delaying is really simple. All we want is a simple timer that waits until the set time is up. The only thing important to note here is that why didn't we just utilize WaitForSeconds(duration)? If you don't know what WaitForSeconds is, it is a YieldInstruction you can return that tells a Coroutine to yield for that duration. So again, why not? It encapsulates everything we want, and we don't have to manage our own timer every frame... Well, what happens when we pause? If we use WaitForSeconds, nothing, because it won't know it is paused until after it returns from waiting the full duration we specified. That on its own competely destroys the reliablity we are trying to build with Actions. We want to be able to just code our logic without having to worry about edge cases, and using WaitForSeconds would cause users to have to be aware of that edge case. Since the goal is making life easier, we use our own timers rather than utilizing Coroutine functionality. 

{% highlight c# %} 
public class ActionDelay : Action
{
  private float timeLeft_; // Time left to wait

  // Constructor to add delay to manager and set timer
  public ActionDelay(ActionManager manager, float duration) 
  { 
    timeLeft_ = duration;
    SetParent(manager);
    AddAction(this);
  }

  // Coroutine for updating delays
  public override IEnumerator Update ()
  {
    // We have started running
    running_ = true;

    // Delay until there is no time left to wait
    while(timeLeft_ > 0)
    {
      // If we are not paused, decrement time
      if(!IsPaused())
        timeLeft_ -= Time.deltaTime;
      // yield until timer stops
      yield return null; 
    }

    // Mark action as completed, and break yields
    completed_ = true;
    yield break;
  }	
}
{% endhighlight %} 

### ActionCall\: ###

Action Calls are one of the biggest workers of the Action System. Here is where you will be writing the logic in the next step in your sequence, or logic that is meant to run simultaneously within its group. They can be simple things, for example calling a sound effect, or it can be the next step within some complex logic. Just remember that Action Calls are single frame functions. What I mean by this is that we call this function in our current frame; it runs the code and then leaves the function all in the same frame. This is important because our Action system expects to just call this function and move on, but if we have logic that is running off into the wild we have no idea what could happen! That is why we have Action Routines, for those multi-frame functions. 

{% highlight c# %} 
public class ActionCall : Action
{
  public delegate void Function(); // Delegates for ActionCall must be of this type
  private Function function_;      // Store the delegate the user attaches to this action

  // Constructor, provide function for action call
  public ActionCall(ActionManager manager, Function function)
  {
    function_ = function;
    SetParent(manager);
    AddAction(this);
  }

  // Coroutine that updates action calls
  public override IEnumerator Update () 
  {
    // We have started running
    running_ = true;

    // While we are paused, wait
    while(IsPaused()) 
    {
      yield return null;
    }

    // Activate single frame function 
    function_();
    // Mark as complete, and break yieldings
    completed_ = true;
    yield break;
  }	
}
{% endhighlight %} 

One thing I would like to point out is that our ActionCall functions take nothing and return nothing. Why we have no return type is quite obvious, since after the function is run it comes back to the Action System, and then what is it going to do with it? The Action System is generic and shouldn't ever care about what happened in your function. Now I've only been using C# for about 4 months when I wrote this system and could not find any good ways to support a varadic number of parameters. From my understanding there are no variadic functions in C#, well the only option is to use params object[] and I don't like that. I didn't want to require the user to have to remember how they laid out their parameters, which is error prone. Now, if you are in C++ you can use the lovely std::function<void()> that allows you to pass in a functor with any number of parameters! Sadly, there are also no functors within C# either. But, I've seen some sources of people writing their own and will probably go back and do that when I am able. If anyone has any better solutions, I would love to know!

### Action Routine\: ###

The only things different here from ActionCall is that we require a Coroutine instead and we need to override Cancel. Since we are creating another Coroutine, we need the ActionRoutine to manage it. So, if we are cancelled, it can end the routine it started as well as itself.

{% highlight c# %} 
public class ActionRoutine : Action
{
  public delegate IEnumerator Function();
  private Function function_;
  protected Coroutine actionRoutine_ = null;

  public ActionRoutine(ActionManager manager, Function function)
  {
    function_ = function;
    SetParent(manager);
    AddAction(this);
  }

  public override void Cancel ()
  {
    // Cancel the routine we started
    if(parent_ != null && actionRoutine_ != null)
      parent_.GetTarget().StopCoroutine(actionRoutine_);

    // Then call our base class to cancel ourselves
    base.Cancel();
  }

  // Update is called once per frame
  public override IEnumerator Update () 
  {
    // We have started running
    running_ = true;

    // While we are paused, wait
    while(IsPaused()) 
    {
      yield return null;
    }

    // Wait until we return from the function
    actionRoutine_ = parent_.GetTarget().StartCoroutine(function_());
    yield return actionRoutine_;
    completed_ = true;

    // We are complete, break yielding
    yield break;
  }	
}
{% endhighlight %} 

Okay, I lied about this article including all the Actions! The only Action we have left are ActionProperties, and they were the bane of my existence. I wanted to seperate them into their own article, so I could talk about how I ended up implementing them the way I did. It definitely was a journey! 

>

[Continue to part 5. (Action Properties)](http://joshualouderback.com/ActionProperties/)

---
layout: post
title: Action Pausing And Cancelling (Part 3)
excerpt: "How To Pause And Cancel Actions!"
modified: 2016-01-30
tags: [Unity, Action, Actions, Sequences, Groups, Action Pausing, Action Cancelling, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### Pausing\: ###

By how we already setup our actions pausing is really easy to finish off. If we look back at our base Action class:

{% highlight c# %} 
///////  Settors /////// 
public void Pause() { parent_.paused_ = true; }
public void Resume() { parent_.paused_ = false; }

/////// Gettors /////// 
public virtual bool IsPaused() { return parent_.IsPaused(); }
{% endhighlight %} 

You can see that pausing rest upons our parents, and this is true due to the fact that actions are hierarchical. If any piece of the "engine" stops, so will the rest of it. Now there is only one problem in the "engine" that we have to fix. You may have been asking yourself why is IsPaused() marked virtual? Well if you think about it, if we always ask our parent, then the parent at the top of the hierarchy would just keep asking themselves. Remember in the constructor of a sequence we did this:

{% highlight c# %} 
  public ActionSequence(GameObject target) {
    // Sequence's parents should always start as themselves
    parent_ = this;
    ...
  }
{% endhighlight %} 

The main sequence will be the top of the hierarchy, so their parent will be left as themselves. Now when it is time to ask if they are paused, they will go spiral into an infinite loop of asking themselves if they are paused. Obviously we can't let the head hancho be a complete fool! This is where the override of the IsPaused() method comes in:

{% highlight c# %} 
public class ActionSequence : Action 
{
  ...

  // Override base isPaused implementation because we could be our own parent
  public override bool IsPaused()
  {
    // We may not be our parent if we are a sequence in a sequence, so check
    if(parent_ == this)
      return paused_;
    else
      return parent_.IsPaused();
  }
}
{% endhighlight %} 

And there you have it! We can pause and unpause any of our actions! Now, we have another fish to fry... 

---

### Cancelling\: ###

Cancelling was actually the last thing I approached when implementing actions, and it caused a bit of a re-write. Luckily for you, all the implementation I have showed you is ready to support cancelling without any need for rewrites. In my original implementation, I was mistaken by the functionality of StopCoroutine(). I thought that since I was able to StartCoroutine by supplying the coroutine function, that I could also end a coroutine by supplying it with the same coroutine function like so:

{% highlight c# %} 
// Get the current action, so we can update it
Action action = sequence.Peek();

// Start the coroutine supply the coroutine update function
StartCoroutine(action.Update());

// Then stop it by supplying the same coroutine function
StopCoroutine(action.Update());

{% endhighlight %} 


Since this wasn't the case, I was concerned that having a self-managed action system wouldn't be possible. Somewhere I needed to keep track of the coroutines, so that I could cancel them when needed. And that is when it hit me! I can just store the coroutines within the actions themselves. Now if we look back to the base action class we can see why there was a Coroutine member, and why we had StartAction and Cancel methods!

{% highlight c# %} 
public abstract class Action {
  protected Coroutine routine_ = null;
  ...

  // Originally I had Groups and Sequences calling StartCoroutine, but since we needed 
  // the actions to manage themselves I just wrapped it inside a function.
  // So the action itself can get the coroutine and the caller can get it too.
  public Coroutine StartAction() {
    // Set our routine, because me manage ourselves
    routine_ = target_.StartCoroutine(Update());
    // Return the routine, just in case others want to yield on us
    return routine_;
  }

  // I also had Groups and Sequences calling StopCoroutine, but that was incorrect.
  // Now the action itself has to call StopCoroutine since it holds the coroutine.
  public virtual void Cancel() {
    if(routine_ != null)
      target_.StopCoroutine(routine_);
  }
}
{% endhighlight %} 

"Wait a minute!", you may be saying. You've seen my tricks and you may have noticed that I marked Cancel() as virtual and there is someone that needs to override it. You got me! And I bet you guessed that it would be the parents who have to override it! Well you are close, unless you were thinking about the two "managers", groups and sequences. Since group and sequences hold the actions, they are the ones who have to tell all of their children that they are being cancelled. Lets see what we need to add to handle that:

{% highlight c# %} 
public class ActionSequence : Action {
  ...

  // Now since sequences are usually at the top not only do we have to handle 
  // telling it to cancel its children, but it also needs to be able to cancel itself.
  public override void Cancel()
  {
    // As long as we have objects
    if(sequence.Count > 0)
    {
      // Grab the current action
      Action action = sequence.Peek();
      // Cancel the action
      action.Cancel();
      // Remove the rest
      sequence.Clear();
    }

    // Make sure we have a routine, before cancelling blindly
    if(routine_ != null)
    {
      // Since we do, stop it and null it out
      target_.StopCoroutine(routine_);
      routine_ = null;
    }
  }
{% endhighlight %} 

Groups are basically the same thing:

{% highlight c# %} 
public class ActionGroup : Action {
  ...
  
  // If we need to cancel all 
  public override void Cancel()
  {
    // Loop through every action and stop it
    foreach(var action in list)
    {
      // Tell the action to cancel itself
      action.Cancel();
    }

    // Clear it
    list.Clear();

    // Make sure we have a routine, before cancelling blindly
    if(routine_ != null)
    {
      // Stop it
      target_.StopCoroutine(routine_);
      routine_ = null;
    }
  }
}
{% endhighlight %} 

There we go! Now that we have all of the management functionality out of the way, we can move onto the actions that actually do actions!

>

[Continue to part 4. (All The Actions)](http://joshualouderback.com/AllTheActions/)

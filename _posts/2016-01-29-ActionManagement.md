---
layout: post
title: Action Management (Part 2)
excerpt: "Sequences and Groups."
modified: 2016-01-30
tags: [Unity, Action, Actions, Sequences, Groups, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### Management\: ###

When I approached implementing actions one of the biggest things I wanted was self-management. I didn't want some complex system managing a simple system, especially when there is no need. If a player creates an action and they care about it, they will store it within whoever cares about. This way if they care to pause or cancel it they don't need to go looking for it. Also since actions are bound to objects, if you don't want an action to persist attach to it to an object that will eventually be destroyed. You can use this to your advantage in some cases when binding logic with actions, for example a boss attack sequence will automatically end when the boss is destroyed. Lastly, with the benefit of using coroutines we don't need to add an extra system to manage it, since the engine is already managing it for us. The only thing we need to manage are the coroutines themselves, but that is easily done by making the action itself manage it. And due to the nature of how actions are structured, which we will get into below, the main "managers" of actions are the top most actions in the hierarchy. Any part of the hierarchy will affect the rest, so having them manage themselves and each other is such a logical direction to go. Now onto the first and major "manager" of actions, sequences!

### Sequences\: ###

Sequences are the main "manager" of actions, because every action belongs within some sequence, even if it is just one action. We can easily represent a sequence as queue since it has all the properties of a sequence: 

* First in first out, which maintains our order.
* Front access only, therefore we can't update next action until front is done and removed.

With something to manage the order of our actions and when to notify the next action, we have basically stripped out all of the annoyances of writing what we used to call "simple code". We can really call it "simple code" now that we only have to focus on what the logic is.

{% highlight c# %}
public class ActionSequence : Action
{
  // The sequence of actions, placed in queue to maintain FIFO
  private Queue<Action> sequence = new Queue<Action>();

  // Constructor binding the sequence to a game object
  public ActionSequence(GameObject target)
  {
    // Sequence's parents should always start as themselves
    parent_ = this;

    // If target is specified
    if(target != null)
    {
      // Get action component
      var actions = target.GetComponent<Actions>();
      // If the has it attach our target to it
      if(actions != null)
        target_ = actions;
      else // Otherwise add it ourselves and attach
        target_ = target.AddComponent<Actions>();
    }
    else // Otherwise use singleton
    {
      target_ = Singleton<Actions>.Instance;
    }
  }

  // Add function to allow users to add any action type to queue	
  public void AddAction(Action action)
  {
    // Parent the action to this sequence
    action.SetParent(this);

    // Place action in queue
    sequence.Enqueue(action);

    // If this is our first action, activate sequence coroutine
    if(sequence.Count == 1)
    {
      routine_ = target_.StartCoroutine(this.Update());
    }
  }

  // Coroutine for updating sequences
  public override IEnumerator Update()
  {
    // We have started running
    running_ = true;

    // While there are actions to update, update them
    while(sequence.Count > 0)
    {
      // Get the current action, so we can update it
      Action action = sequence.Peek();

      // If we are paused yield until unpaused
      while(IsPaused())
        yield return null;

      // Call the action once, then wait until it is completed
      while(!action.IsCompleted())
      {	
        // If action isn't already running, start it
        if(!action.IsRunning())
          // Yield until the action we started finishes
          yield return action.StartAction();
      }

      // Remove the current action
      sequence.Dequeue();
    }

    // Complete, break yielding
    yield break;
  }
}
{% endhighlight %} 

In the update of a sequence is really when we start utilizing all the capabilites of coroutine to our advantage. A coroutine will maintain all of the data that was on the stack before we yielded, this way when we return all of the data we had before is still here. The while loop wrapped around our action.StartAction() is purely there to take advantage of this. After the started action is completed we will return to while(!action.IsCompleted()) and we will immediately break out of the loop, and remove our action. We technically bypass two unneccessary operations getting the action and checking if it is paused. As you hopefully can start to see you can really start extending this capability to far more complex logic and gain even more of an optimization.


### Groups ###

Coming soon...

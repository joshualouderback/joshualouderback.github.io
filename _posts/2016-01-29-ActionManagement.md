---
layout: post
title: Action Management (Part 2)
excerpt: "Sequences and Groups!"
modified: 2016-01-30
tags: [Unity, Action, Actions, Sequences, Groups, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### Management\: ###

When I approached implementing Actions, one of the biggest things I wanted was self-management. I didn't want some complex system managing a simple system, especially when there is no need. If a player creates an action and they care about it, they will store it within whatever object cares about it. This way if they care to pause or cancel it they don't need to go looking for it. Also, since Actions are bound to objects, if you don't want an action to persist, then attach to an object that will eventually be destroyed. You can use this to your advantage in some cases when binding logic with Actions. For example, a boss's attack sequence will automatically end when the boss is destroyed. Lastly, one the benefits of using coroutines is that we don't need to add an extra system to manage it, since the engine is already managing it for us. The only thing we need to manage are the coroutines themselves, but that is easily done by making the action manage itself. And due to the nature of how Actions are structured, which we will get into below, the main "managers" of Actions are the top most Actions in the hierarchy. Any part of the hierarchy will affect the rest, so having them manage themselves and each other is a logical direction to go.  

---

### Action Managers\: ###

In the previous article we laid out the base class of all Actions, but they referenced something known as an ActionManager as their parent. Action Managers are an extension of the Action class to reduce memory footprint, support the interface we want and maintain performance. Since Actions are hierarchical, we want to ensure that they all of their children have the same target as their parent, pause and unpause with their parent and cancel with their parent. Those are what the Action Manager class offers us and now let us see how we support that with the ActionManager implementation:

{% highlight c# %}
// The "Managers" of Actions (groups and sequences)
public abstract class ActionManager : Action
{
  // Since every action relies on these, but always look to their parent,
  // then we abstracted them out to the managers to reduce memory footprint
  protected Actions target_ = null;
  protected bool paused_ = false;

  // Children need to get the parents target to know who to ask to start
  // and stop their coroutines
  public Actions GetTarget() { return target_; }

  // Since the managers control pausing and resume we need to override the request
  // virtual functions of the children Actions
  public override void Pause() {
    // We may not be our own parent
    if(parent_ == this)
      paused_ = true;
    else
      return parent_.Pause();
  }
  public override void Resume() {     
    // We may not be our own parent
    if(parent_ == this)
      paused_ = false;
    else
      return parent_.Resume();
  }
}
{% endhighlight %}

>

Now that we got the ground work of ActionManagers out of the way, we can move onto the first and major "manager" of Actions, Sequences!

---

### Sequences\: ###

Sequences are the main "manager" of Actions, because every action belongs within some sequence, even if it is just one action. We can easily represent a sequence as a queue, since it has all the properties of a sequence: 

* First in first out, which maintains our order.
* Front access only, therefore we can't update the next action until the front is done and removed.

With something to manage the order of our Actions and when to notify the next action, we have basically stripped out all of the annoyances of writing what we used to call "simple code". We can really call it "simple code" now that we only have to focus on what the logic is.

{% highlight c# %}
public class ActionSequence : Action
{
  // The sequence of Actions, placed in queue to maintain FIFO
  private Queue<Action> sequence = new Queue<Action>();

  // In case we want to add a sequence to a sequence or a sequence to a group
  public ActionSequence(ActionManager manager)
  {
    // Set our parent to them
    SetParent(manager);
    // And set our target as theirs
    target_ = manager.GetTarget();
    // Add ourselves to them
    AddAction(this);
  }

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
      // If the target has our component, attach a reference
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

  // Restricts users to only be able to add Actions through constructors
  protected override void AddAction(Action action)
  {
    // Place action in queue
    sequence.Enqueue(action);
    
    // If this is our first action, and we are the parent
    // then activate sequence coroutine
    if(sequence.Count == 1 && parent_ == this)
    {
      routine_ = target_.StartCoroutine(this.Update());
    }
  }

  // Coroutine for updating sequences
  public override IEnumerator Update()
  {
    // We have started running
    running_ = true;

    // While there are Actions to update, update them
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

In the update of a sequence is when we really start utilizing all the capabilites of coroutines to our advantage. A coroutine will maintain all of the data that was on the stack before we yielded, this way when we return, all of the data we had before is still here. The while loop wrapped around our action.StartAction() is purely there to take advantage of this. After the started action is completed we will return to while(!action.IsCompleted()) and we will immediately break out of the loop, and remove our action. We technically bypass two unneccessary operations, getting the action, and checking if it is paused. As you hopefully can start to see, you can really start extending this capability to far more complex logic gaining even more of an optimization.

---

### Groups ###

Now that we have sequences, it would be helpful if we had an action that could run Actions in parallel! If you remember our example from last time, we wanted an asteroid to rotate as it was travelling towards the player's position. The only way we would be able to do this is if we could move the asteroid with one action and combine it with another action that rotates it. Remember that all Actions are contained in a sequence, and a sequence can only update one action at a time? Therefore, we will want to create an action that consists of a _group_ of Actions. This is where action groups come in, the next "manager" of Actions. Lets take a look at how the action group is laid out:

{% highlight c# %}
public class ActionGroup : Action
{
  // We use a List for the Actions in the group, since we want to be able to 
  // access each action without having to remove any.
  private List<Action> list = new List<Action>();

  // Default Constructor
  public ActionGroup()
  {
  }

  // Restricts users to only be able to add Actions through constructors
  protected override void AddAction(Action action)
  {
    // Add the action to the group list
    list.Add(action);
  }

  // Coroutine for updating Action Groups
  public override IEnumerator Update()
  {
    // We have started running
    running_ = true;

    // If we are started and empty, wait
    while(list.Count == 0)
    {
      // This just allows users to add the group into sequence before adding
      // Actions into the group
      yield return null;
    }

    // While we are paused, wait
    while(IsPaused()) 
    {
      yield return null;
    }

    // Loop through every action and start it
    foreach(var action in list)
    {
      action.StartAction();
    }

    // Now that all have started, yield until they all are completed
    for(int i = 0; i < list.Count; )
    {
      if(!list[i].IsCompleted())
        yield return null;
      else 
        ++i;
    }

    // Clear the group and mark as completed
    completed_ = true;
    list.Clear();	
    // Now we are done, break yielding
    yield break;
  }
}
{% endhighlight %} 

If you recall in base implementation of the Action class, I mentioned that I would get back to why AddAction is written the way it is. All of that is explained in the next section of the article, found by clicking below!

>

[Continue to part 2.5. (Adding Actions)](http://joshualouderback.com/AddingActions/) 

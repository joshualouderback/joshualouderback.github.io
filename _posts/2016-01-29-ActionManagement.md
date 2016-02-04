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

When I approached implementing actions, one of the biggest things I wanted was self-management. I didn't want some complex system managing a simple system, especially when there is no need. If a player creates an action and they care about it, they will store it within whatever object cares about it. This way if they care to pause or cancel it they don't need to go looking for it. Also, since actions are bound to objects, if you don't want an action to persist, then attach to an object that will eventually be destroyed. You can use this to your advantage in some cases when binding logic with actions. For example, a boss's attack sequence will automatically end when the boss is destroyed. Lastly, one the benefits of using coroutines is that we don't need to add an extra system to manage it, since the engine is already managing it for us. The only thing we need to manage are the coroutines themselves, but that is easily done by making the action manage itself. And due to the nature of how actions are structured, which we will get into below, the main "managers" of actions are the top most actions in the hierarchy. Any part of the hierarchy will affect the rest, so having them manage themselves and each other is a logical direction to go. Now onto the first and major "manager" of actions, sequences!

---

### Sequences\: ###

Sequences are the main "manager" of actions, because every action belongs within some sequence, even if it is just one action. We can easily represent a sequence as a queue, since it has all the properties of a sequence: 

* First in first out, which maintains our order.
* Front access only, therefore we can't update the next action until the front is done and removed.

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

In the update of a sequence is when we really start utilizing all the capabilites of coroutines to our advantage. A coroutine will maintain all of the data that was on the stack before we yielded, this way when we return, all of the data we had before is still here. The while loop wrapped around our action.StartAction() is purely there to take advantage of this. After the started action is completed we will return to while(!action.IsCompleted()) and we will immediately break out of the loop, and remove our action. We technically bypass two unneccessary operations, getting the action, and checking if it is paused. As you hopefully can start to see, you can really start extending this capability to far more complex logic gaining even more of an optimization.

---

### Groups ###

Now that we have sequences, it would be helpful if we had an action that could run actions in parallel! If you remember our example from last time, we wanted an asteroid to rotate as it was travelling towards the player's position. The only way we would be able to do this is if we could move the asteroid with one action and combine it with another action that rotates it. Remember that all actions are contained in a sequence, and a sequence can only update one action at a time? Therefore, we will want to create an action that consists of a _group_ of actions. This is where action groups come in, the next "manager" of actions. Lets take a look at how the action group is laid out:

{% highlight c# %}
public class ActionGroup : Action
{
  // We use a List for the actions in the group, since we want to be able to 
  // access each action without having to remove any.
  private List<Action> list = new List<Action>();

  // Default Constructor
  public ActionGroup()
  {
  }

  // Add function to push actions into group
  public void AddAction(Action action)
  {
    // Parent the action to this group
    action.SetParent(this);
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
      // actions into the group
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

One major note about this implementation is that we are supporting all ways a user can add a group to a sequence. Users can create the group, followed by either adding all of the actions to it, then adding it to the sequence like so: 

{% highlight c# %} 

  void Start()  {
    ActionSequence seq = new ActionSequence(this.gameObject);
    ActionGroup grp = new ActionGroup();
    // Add the actions
    grp.AddAction(new Action());
    ...
    grp.AddAction(new Action());
    // Then add the group to the sequence
    seq.AddAction(grp);
  }

{% endhighlight %} 

Or by adding it the sequence first, then adding all of the actions like in the example below:

{% highlight c# %} 

  void Start()  {  
    ActionSequence seq = new ActionSequence(this.gameObject);
    ActionGroup grp = new ActionGroup();
    // Add the group to the sequence
    seq.AddAction(grp);
    // Then add the actions
    grp.AddAction(new Action());
    ...
    grp.AddAction(new Action());
  }

{% endhighlight %} 


Since we have no way to enforce one or the other (unless by throwing errors), I thought it best to be able to support both ways. This is why the action group needs to overload SetParent(), because if we added all of our actions to the group before adding the group to the sequence, then all of the targets for the group's children will be wrong.  

{% highlight c# %} 
public class ActionGroup : Action {
  ...

  // Overriding since we may not of set our parent to the sequence we add ourselves
  // to yet, so we will want to garauntee all of children get to set to it here
  public override void SetParent (Action parent)
  {
    // Call the base implementation to set our parent and target
    base.SetParent(parent);

    // We only need to update our children if we have any
    if(list.Count > 0)
    {
      // Loop through all the actions in our list and set their parents,
      // effectively updating their target correctly
      foreach(Action action in list)
      {
        // Otherwise set it
        action.SetParent(this);
      }
    }
  }
}
{% endhighlight %} 

Notice that we have only covered adding and updating actions within groups and sequences? What about pausing and cancelling? These two things also rely on our "managers" of actions and are covered in the next part of the article.

>

[Continue to part 3. (Pausing And Cancelling)](http://joshualouderback.com/PausingAndCancelling/) 

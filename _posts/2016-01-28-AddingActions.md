---
layout: post
title: Adding Actions (Part 2.5)
excerpt: "Why Do We Add Actions The Way We Do?"
modified: 2016-01-30
tags: [Unity, Action, Actions, Sequences, Groups, Programming, Tips]
comments: true
#image:
#  feature: sample-image-5.jpg
---

### Why protected AddAction? ###

As promised, I am going to explain why AddAction was set to be a virtual protected function. Note, what is below does not relate to the current implementation and should not be replicated, it is purely to explain the evolution of the Action System. In previous iterations, AddAction was the main interface to adding Actions to groups and sequences. There were two really big problems with this: 

>

**1) Supporting both ways to add to groups and sequences.**

>

Users used to be able to create the group, followed by either adding all of the Actions to it, then adding it to the sequence like so:

{% highlight c# %}
void Start()  {
  ActionSequence seq = new ActionSequence(this.gameObject);
  ActionGroup grp = new ActionGroup();
  // Add the Actions
  grp.AddAction(new Action());
  ...
  grp.AddAction(new Action());
  // Then add the group to the sequence
  seq.AddAction(grp);
}
{% endhighlight %} 

Or by adding it the sequence first, then adding all of the Actions like in the example below:

{% highlight c# %}
void Start()  {  
  ActionSequence seq = new ActionSequence(this.gameObject);
  ActionGroup grp = new ActionGroup();
  // Add the group to the sequence
  seq.AddAction(grp);
  // Then add the Actions
  grp.AddAction(new Action());
  ...
  grp.AddAction(new Action());
}
{% endhighlight %} 

Since we have no way to enforce one or the other (unless by throwing errors), I thought it best to be able to support both ways. This is why the old implementation needed to overload SetParent(), because if we added all of our Actions to the group before adding the group to the sequence, then all of the targets for the group’s children will be wrong.

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
      // Loop through all the Actions in our list and set their parents,
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

**2) Keeping sequences from firing off, even though their parents didn't trigger them.**

{% highlight c# %}
void Start()  {
  // Create the parent sequence  
  ActionSequence seq = new ActionSequence(this.gameObject);
  // Give some action to do first, before starting the second sequence
  seq.AddAction(new Action());
  // Create the second sequence
  ActionSequence seq2 = new ActionSequence(this.gameObject);
  // Add the Actions
  seq2.AddAction(new Action());
  ...
  seq2.AddAction(new Action());
  // Then add the second sequence to the first sequence
  seq.AddAction(seq2);
}
{% endhighlight %} 

Recall whenever an ActionSequence gains an Action and it is the parent, then it can start triggering the sequence. As you can see in the example above, when we create that second sequence and start adding Actions to it, that sequence will trigger. The second sequence is it's own parent right up to the AddAction call to add the second sequence to the first. This fundamentally destroys the point of Actions, which is control the order and flow of Actions. If we have a sequence start before it should, it will cause game breaking features. This is why AddAction is now a protected virtual function. Within the base class of Action we implemented AddAction like so:

{% highlight c# %}
protected virtual void AddAction(Action action) { parent_.AddAction(action); }
{% endhighlight %} 

This is effectively asking our parent to add ourselves to them. Now we have to override this function in our ActionManager's, so when the child asks to be added they know how to add them. Hence where the virtual keyword came into play. Finally, it is marked protected so that the only way to call AddAction is within the class itself. This way users outside the implementation cannot AddActions without following our desired interface, because if we let them call AddAction they could repeat the 2nd problem we listed above. 

>

Notice that we have only covered adding and updating Actions within groups and sequences? What about pausing and cancelling? These two things also rely on our "managers" of Actions and are covered in the next part of the article.

>

[Continue to part 3. (Pausing And Cancelling)](http://joshualouderback.com/PausingAndCancelling/) 

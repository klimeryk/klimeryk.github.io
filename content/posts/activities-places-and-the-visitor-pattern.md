---
title: 'Activities, places and the visitor pattern'
date: Mon, 22 Sep 2014 17:01:00 +0000
draft: false
url: /2014/09/22/activities-places-and-the-visitor-pattern/
tags: ['Design patterns', 'Google Web Toolkit', 'GWT', 'Programming']
---

If you’re writing a GWT application you are most likely following the MVP pattern. With GWT 2.1 the [Activities and Places API](http://www.gwtproject.org/doc/latest/DevGuideMvpActivitiesAndPlaces.html) was introduced and while it’s not strictly an MVP framework, it’s a good base for the architecture of your application.

A core component of Activities and Places is the [`ActivityMapper`](http://www.gwtproject.org/javadoc/latest/com/google/gwt/activity/shared/ActivityMapper.html) interface. Its role is simple - you get a `Place` and based on it an appropriate `Activity` should be returned. There’s one problem - you most likely will want to choose the `Activity` based on the type of the provided `Place`. The usual approach (mentioned even in the official documentation) is a long chain of `if (place instanceof CustomPlace) else if (place instanceof CustomPlace2)`… etc. You might have a similar `if/else` chain somewhere else in your application where you want to react to a `PlaceChangedEvent`. If you add a new `Place`, you have to remember to add (another…) `if else` in all those places.

```java
public class AppActivityMapper implements ActivityMapper {
    @Override
    public Activity getActivity(Place place) {
        activity = null;
        if (place instanceof StartPlace) {
            return new StartActivity();
        } else if (place instanceof SearchPlace) {
            return new SearchActivity();
        } else if (place instanceof CustomerPlace) {
            return new CustomerActivity();
        }
        return null;
    }
}
```

There’s gotta a be a better way!

And there is: [the visitor pattern](http://en.wikipedia.org/wiki/Visitor_pattern).

> A way of separating an algorithm from an object structure on which it operates. A practical result of this separation is the ability to add new operations to existing object structures without modifying those structures.

Sounds exactly like something we need! If the details are unclear at this point, don’t worry. It’s best to see it in action and how it fits with our needs.

Let’s say we have three custom `Place`s in our application: `StartPlace`, `SearchPlace` and `CustomerPlace`. They can be with parameters/tokens or not - it doesn’t matter.

Now, we’ll define the interface of our visitor. This visitor will be able to “visit” every place that “accepts” it and invoke some place-specific method.

```java
public interface PlaceVisitor {
    void visitStartPlace(StartPlace place);
    void visitSearchPlace(SearchPlace place);
    void visitCustomerPlace(CustomerPlace place);
}
```

As you can see, we have a method for each of our custom places. Every method takes a place as a parameter and returns nothing. We could return an `Activity` but as we’ll see further on, it limits the usage of this interface and is actually against the visitor pattern.

Now, let’s add a new base class for all of our custom places. It could be an interface, but since we always have to extend the `Place` class, we might as well extend it and add our method:

```java
public abstract class AppPlace extends Place {
    public abstract void accept(PlaceVisitor visitor);
}
```

We only need to add one method - one that “accepts” the visitor. We leave the implementation to the specific place - that’s the whole point. We can now modify our `Place` to extend our `AppPlace` instead of just `Place`:

```java
public class StartPlace extends AppPlace {
    @Override
    public void accept(PlaceVisitor visitor) {
        visitor.visitStartPlace(this);
    }
}

public class SearchPlace extends AppPlace {
    @Override
    public void accept(PlaceVisitor visitor) {
        visitor.visitSearchPlace(this);
    }
}

public class CustomerPlace extends AppPlace {
    @Override
    public void accept(PlaceVisitor visitor) {
        visitor.visitCustomerPlace(this);
    }
}
```

The pieces should start falling into place now - we can see above that it’s the place that decides to invoke a specific method on the visitor. This way, the visitor doesn’t know what place is accepting it, but the place can tell it: “Hey, you’re visiting a `SearchPlace`, act accordingly”.

We can now replace our long `ActivityMapper` implementation with a much simpler one:

```java
public class AppActivityMapper implements ActivityMapper, PlaceVisitor {
    private Activity activity;

    @Override
    public Activity getActivity(Place place) {
        activity = null;
        if (place instanceof AppPlace) {
            ((AppPlace) place).accept(this);
        }
        return activity;
    }

    @Override
    public void visitStartPlace(StartPlace place) {
        activity = injector.getStartActivity();
    }
    
    @Override
    public void visitSearchPlace(SearchPlace place) {
        activity = injector.getSearchActivity();
    }
    
    @Override
    public void visitCustomerPlace(CustomerPlace place) {
        activity = injector.getCustomerActivity();
    }
}
```

Now, whenever we add a new `Place`, we make it extend `AppPlace`, this makes us provide the implementation for the `accept` method which in turn causes us to add a new method to the `PlaceVisitor` interface. Now, say we have a `PlaceChangedEvent`’s handler somewhere we can re-use the `PlaceVisitor`:

```java
public class RootPresenter ... implements PlaceVisitor {

    public RootPresenter(...) {
        eventBus.addHandler(PlaceChangeEvent.TYPE, new PlaceChangeEvent.Handler() {
            @Override
            public void onPlaceChange(PlaceChangeEvent event) {
                Place place = event.getNewPlace();
                if (place instanceof AppPlace) {
                    ((AppPlace) place).accept(RootPresenter.this);
                }
            }
        });
    }
    
    @Override
    public void visitStartPlace(StartPlace place) {
        view.getStartButton.setActive(true);
        // Or whatever else
    }
    
    // etc...
}
```

Whenever we add a new `Place` we have to add a new method to the `PlaceVisitor` which in turn flags any class that implements it with an error - now it’s impossible to miss handling a new `Place` somewhere in your code.

One more point worth mentioning - in the last snippet you can see that making the `visit*` methods return an `Activity` directly wouldn’t be a good idea - while it would be useful in the `ActivityManager`’s implementation, it would be useless in `RootPresenter`.

I highly recommend reading the appropriate chapter from the great [Design Patterns](http://en.wikipedia.org/wiki/Design_Patterns) book from [the Gang of Four](http://c2.com/cgi/wiki?GangOfFour). The visitor pattern is a very useful one and the chapter gives examples when it’s good to use and when a different approach is better.

Happy coding!

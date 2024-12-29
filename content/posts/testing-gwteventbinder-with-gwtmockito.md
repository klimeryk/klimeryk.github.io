---
title: 'Testing gwteventbinder with gwtmockito'
date: Wed, 27 Aug 2014 21:46:00 +0000
draft: false
tags: ['Google Web Toolkit', 'GWT', 'Programming', 'Testing']
---

[gwteventbinder](https://github.com/google/gwteventbinder) and [gwtmockito](https://github.com/google/gwtmockito) are great projects that are esential if you’re writing applications in Google Web Toolkit/GWT. So it was a surprise to me that they don’t play together well “out of the box”.

See, the problem is that gwtmockito injects it’s own (safe, not using any code that requires running a browser) mocks when it encounters `GWT.create` in your code. That’s very cool, but when using gwteventbinder, you need to call the `bindEventHandlers` method to, well, bind the event handler. And the mock, obviously, doesn’t do that.

However, there’s a solution - `FakeProvider`s. By registering one for the `EventBinder` interface, it will be used for providing instances of `EventBinder` _and_ its subclasses. Unfortunately, `EventBinder` uses a GWT Generator for it’s implementation, and to “emulate” that during our tests, we have to resort to Java Reflection.

The following implementation is inspired by comments and suggestions from [an issue](https://github.com/google/gwteventbinder/issues/17) raised on gwteventbinder’s tracker. You can find alternative solutions there, too.

```java
public class FakeEventBinderProvider implements FakeProvider<EventBinder<?>> {
    @Override
    public EventBinder<?> getFake(Class<?> type) {
        return (EventBinder<?>) Proxy.newProxyInstance(FakeEventBinderProvider.class.getClassLoader(), new Class<?>\[\] { type }, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, final Object\[\] args) throws Throwable {
                String methodName = method.getName();
                assert methodName.equals("bindEventHandlers");

                final List<HandlerRegistration> registrations = new LinkedList<HandlerRegistration>();
                EventBus eventBus = (EventBus) args\[1\];

                List<Method> presenterMethods = getAllMethods(args\[0\].getClass());
                for (final Method presenterMethod : presenterMethods) {
                    if (presenterMethod.isAnnotationPresent(EventHandler.class)) {
                        @SuppressWarnings("unchecked") // Should always be ok, since the Generator for EventBinder should do all the safe-checking 
                        Class<? extends GenericEvent> eventType = (Class<? extends GenericEvent>) (presenterMethod.getParameterTypes())\[0\];
                        registrations.add(eventBus.addHandler(GenericEventType.getTypeOf(eventType), new GenericEventHandler() {
                            @Override
                            public void handleEvent(GenericEvent event) {
                                try {
                                    presenterMethod.setAccessible(true);
                                    presenterMethod.invoke(args\[0\], event);
                                } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
                                    throw new RuntimeException(e);
                                }
                            }
                        }));
                    }
                }

                return new HandlerRegistration() {
                    @Override
                    public void removeHandler() {
                        for (HandlerRegistration registration : registrations) {
                            registration.removeHandler();
                        }
                        registrations.clear();
                    }
                };
            }
        });
    }

    private List<Method> getAllMethods(Class<?> type) {
        List<Method> methods = new LinkedList<Method>();
        methods.addAll(Arrays.asList(type.getDeclaredMethods()));
        if (type.getSuperclass() != null) {
            methods.addAll(getAllMethods(type.getSuperclass()));
        }
        return methods;
    }
}
```

It’s not as bad as it seems. When you get rid of the Java Reflection cruft, it boils down to:

1.  Find all the methods on the presenter that are annotated with `EventHandler`.
2.  Register a new handler that invokes the callback method for the event specified as it’s first argument.
3.  Return a `HandlerRegistration` instance that on call to `removeHandler` removes all the handlers added in the previous point (this behavior is copied from the actual implementation of gwteventbinder).

You might note that I don’t do any safe-checks… Well, assuming that you’ve run your code before, using (Super)DevMode, the EventBinder’s generator already checked that your code is sane and within the requirements of gwteventbinder.

Now, all that’s left is to register this provider, for example in your `@Before` method:

```java
@Before
public void setUpEventBindery() {
    GwtMockito.useProviderForType(EventBinder.class, new FakeEventBinderProvider());
}
```

Like I’ve already mentioned, you only need to do this for the base interface, `EventBinder`, since as the documentation for `GwtMockito.useProviderForType` says:

> (..) the given provider should be used to GWT.create instances of the given type and **its subclasses**.

Happy testing!

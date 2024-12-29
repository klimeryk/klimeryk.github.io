---
title: 'Testing asynchronous GWT-RPC services'
date: Thu, 28 Aug 2014 17:01:00 +0000
draft: false
url: /2014/08/28/testing-asynchronous-gwt-rpc-services/
tags: ['Google Web Toolkit', 'GWT', 'Programming', 'Testing']
---

Continuing the theme of testing with gwtmockito, I’d like to show you a neat class that’s bundled with gwtmockito that allows for easy mocking of GWT-RPC asynchronous services.

The class I’m talking about is [AsyncAnswers](http://google.github.io/gwtmockito/javadoc/com/google/gwtmockito/AsyncAnswers.html). It’s meant to be used with the [doAnswer](http://docs.mockito.googlecode.com/hg/org/mockito/Mockito.html#doAnswer(org.mockito.stubbing.Answer)) stubber.

Supposing we have a `LoginServiceAsync` class:

```java
public interface LoginServiceAsync {
    void login(String login, String password, AsyncCallback<SomePojo> callback);
}
```

Assuming `service` is a mock for `LoginServiceAsync`, we could simulate a successful trip to the server with:

```java
doAnswer(AsyncAnswers.returnSuccess(somePojoInstance)).when(service).login(eq(login), eq(password), Matchers.<AsyncCallback<SomePojo>>.any());
```

Similarly, we can simulate that a failure occurred:

```java
doAnswer(AsyncAnswers.returnFailure(new LoginException("Unknown user"))).when(service).login(eq(login), eq(password), Matchers.<AsyncCallback<SomePojo>>.any());
```

Everything should be working, but that line is a bit too long. For starters, we can `import static` `AsyncAnswers`. But the biggest sore in the eye is the `any()` call. We need to explicitly set the type parameters, and for that to happen we can’t rely on static imports - we have to use the `Matchers` class directly. Let’s fix that by introducing a new `ArgumentMatcher`:

```java
public class AnyAsyncCallback<T> extends ArgumentMatcher<AsyncCallback<T>> {
    public AnyAsyncCallback() {
    }
    
    @Override
    public boolean matches(Object argument) {
        return argument instanceof AsyncCallback;
    }
}
```

Now, to use it like `any` or `anyCollection`, put this method somewhere, like in your base test class:

```java
protected static <T> AsyncCallback<T> anyAsyncCallback(Class<T> clazz) {
    return Matchers.argThat(new AnyAsyncCallback<T>());
}
```

We pass around that `<T>` and `Class<T>` for type safety - no need for casts or `@SuppressWarning`s. Now it looks less verbose and more to the point:

```java
doAnswer(returnSuccess(somePojoInstance)).when(service).login(eq(login), eq(password), anyAsyncCallback(SomPojo.class));
```

Happy testing!

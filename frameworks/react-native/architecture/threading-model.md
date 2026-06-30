# Threading Model

## UI/main Thread

This is the primary thread where all native UI components are created and manipulated. It handles user interactions, renders UI components, and manages device screen updates.

Every React Native UI update happens on this thread. Therefore, if you’re manipulating your state frequently, this thread can become busy and cause performance issues.

This thread’s primary role is to keep the interface smooth and responsive. An example is animating a member using the Animated API.

```typescript
Animated.timing(this.state.fadeAnim, {
  // this executes on the UI thread
  toValue: 1,
  duration: 2000,
}).start();
```

In the above code snippet, `Animated.timing` updates the component’s opacity over two seconds. This animation occurs on the Main Thread to ensure smooth UI updates.

## JavaScript Thread

React Native applications execute JavaScript code in a separate JavaScript engine, which happens on the JavaScript thread. This includes API calls, handling touch events, and executing JavaScript code.

This is the thread where your actual React and JavaScript code gets executed. An example would be setting the state after fetching data from an API.

```typescript
fetchData = async () => {
  const response = await fetch("https://api.example.com/data");
  const json = await response.json();
  this.setState({ data: json }); // this line is executed in the JS thread
};
```

This entire operation runs on the JavaScript Thread.

## Native Modules Thread

React Native allows you to write code in native languages (like Java for Android and Objective-C or Swift for iOS) when performing tasks without JavaScript. That is known as a native module, and the execution of this native code happens in the Native Modules thread.

If you’re using native code in your React Native app, it gets executed here. The native modules thread can also offload heavy computations from the JavaScript thread to keep your application responsive.

A simple example would be creating a Toast module in Android.

```java
// This is Java code that will run on the Native Modules Thread
@ReactMethod
public void show(String message, int duration) {
  Toast.makeText(getReactApplicationContext(), message, duration).show();
}
```

In this example, the show method will be invoked from JavaScript but run on the Native Modules thread.
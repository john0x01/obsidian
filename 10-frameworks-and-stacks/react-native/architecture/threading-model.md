# Threading Model

React Native distributes work across several threads so that the user interface stays responsive while JavaScript and native code execute. The three threads a developer interacts with most are the UI (main) thread, the JavaScript thread, and the native modules thread.

## UI / Main Thread

The **UI thread** (also called the main thread) is the primary thread on which all native UI components are created and manipulated. It handles user interactions, renders UI components, and manages updates to the device screen.

Every React Native UI update happens on this thread. Consequently, frequently manipulating state can keep the thread busy and cause performance problems. Its primary role is to keep the interface smooth and responsive — for example, animating a value with the `Animated` API:

```typescript
Animated.timing(this.state.fadeAnim, {
  // this executes on the UI thread
  toValue: 1,
  duration: 2000,
}).start();
```

Here, `Animated.timing` updates the component's opacity over two seconds. The animation runs on the main thread to ensure smooth UI updates.

## JavaScript Thread

React Native runs the application's JavaScript in a separate engine on the **JavaScript thread**. This thread executes the React and JavaScript code, including API calls and touch-event handling.

A typical example is setting state after fetching data from an API:

```typescript
fetchData = async () => {
  const response = await fetch("https://api.example.com/data");
  const json = await response.json();
  this.setState({ data: json }); // this line is executed in the JS thread
};
```

The entire operation runs on the JavaScript thread.

## Native Modules Thread

React Native lets you write code in native languages (Java for Android, Objective-C or Swift for iOS) to perform tasks that JavaScript cannot. Such code is packaged as a **native module**, and its execution happens on the **native modules thread**.

When an app uses native code, it runs here. The native modules thread can also offload heavy computation from the JavaScript thread to keep the application responsive. A simple example is a Toast module on Android:

```java
// This is Java code that will run on the Native Modules Thread
@ReactMethod
public void show(String message, int duration) {
  Toast.makeText(getReactApplicationContext(), message, duration).show();
}
```

The `show` method is invoked from JavaScript but runs on the native modules thread.

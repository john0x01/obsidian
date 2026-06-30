# Rendering

## Fabric

React Native executes the same React framework code as React for the web. However, it renders to native platform views (**host views**) rather than to DOM nodes, which are the web's host views. This is made possible by the **Fabric Renderer**, which lets React talk to each platform and manage its host view instances. The Fabric Renderer exists in JavaScript and targets interfaces exposed by C++ code.

### Shadow Tree

The Fabric Renderer creates a **React Shadow Tree** composed of **React Shadow Nodes**. A React Shadow Node is an object that represents a React Host Component to be mounted; it holds the props that originate from JavaScript along with layout information (`x`, `y`, `width`, `height`). In Fabric, React Shadow Node objects exist in C++. Before Fabric, they lived in the mobile runtime heap (for example, the Android JVM).

## Render, Commit and Mount

## Cross-Platform Implementation

## View Flattening

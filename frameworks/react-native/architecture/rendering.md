# Rendering

## Fabric

React Native executes the same React framework code as React for the web. However, React Native renders to general platform views (host views) instead of DOM nodes (which can be considered web’s host views). Rendering to host views is made possible by the Fabric Renderer. Fabric lets React talk to each platform and manage its host view instances. The Fabric Renderer exists in JavaScript and targets interfaces made available by C++ code.

### Shadow Tree

A React Shadow Tree is created by the Fabric Renderer and consists of React Shadow Nodes. A React Shadow Node is an object that represents a React Host Component to be mounted, and contains props that originate from JavaScript. They also contain layout information (x, y, width, height). In Fabric, React Shadow Node objects exist in C++. Before Fabric, these existed in the mobile runtime heap (e.g. Android JVM).

## Render, Commit and Mount

## Cross Platform Implementation

## View Flattening
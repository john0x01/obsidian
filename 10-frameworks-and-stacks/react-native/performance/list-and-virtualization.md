# List and Virtualization

## Why Virtualization Matters

**Virtualization** is the technique of mounting only the rows currently visible plus a small buffer, while recycling or unmounting the rest. It exists because rendering every row in a `ScrollView` forces React to reconcile every item, Yoga to measure every node, and the platform to build a native view tree for content that may never be on screen. On low-end Android this OOMs around a few thousand rows; on iOS the app feels sluggish well before then, because the main thread has to commit the entire hierarchy. Virtualization amortizes this cost — at the price of complexity, since scroll position, state preservation, and layout measurements all become harder.

## VirtualizedList, FlatList, and SectionList Hierarchy

**`VirtualizedList`** is the base component; `FlatList` and `SectionList` inherit from it with opinions about data shape. The base accepts a generic `getItem`/`getItemCount` API, which matters when your data isn't already an array — cursors, reactive stores, or paginated chunks without a concrete list. `SectionList` is `FlatList` plus section headers and footers; sections are flattened internally, so every section boundary contributes a separate render pass, and `stickySectionHeadersEnabled` adds another layer of work.

## The Render Window: windowSize

**`windowSize`** bounds what stays mounted. It is expressed in multiples of visible length, not items or pixels. The default of `21` means 10 screens above + 1 visible + 10 below — too generous for heavy item content on low-end devices. Drop it to `5`–`7` for media-rich cards; raise it above `11` for smoother scroll with tiny items. Items outside the window are either unmounted (`FlatList`) or hidden (`removeClippedSubviews`).

## initialNumToRender

**`initialNumToRender`** controls how many items render synchronously on mount before virtualization kicks in. Too low and the viewport shows blank space briefly; too high and mount time explodes. Tune it by measuring how many items fit the viewport plus a small buffer — for a feed with ~600pt-tall cards on a standard phone, `4`–`6` is usually right. Do not set it equal to the full list size "to be safe"; that defeats virtualization entirely.

## maxToRenderPerBatch and updateCellsBatchingPeriod

These throttle how aggressively the list renders new items during scroll. **`maxToRenderPerBatch`** is the count of items per batch; **`updateCellsBatchingPeriod`** is the delay in milliseconds between batches. Raising both fills the window faster but blocks the JS thread and causes dropped frames. Lower them on low-end Android; raise them for tiny items on flagship devices. Measure with the Perf Monitor before assuming the defaults are wrong.

## removeClippedSubviews

**`removeClippedSubviews`** is a native optimization that detaches off-screen subviews from the native view hierarchy without unmounting the React component. On Android, detaching views from `ViewGroup`s is cheap and this helps; on iOS it has historically been less reliable and can leave visible items blank after fast scrolls. Test on both platforms before enabling globally, and never combine it with libraries that assume stable native handles.

## keyExtractor Stability

**`keyExtractor`** must return a stable, unique identifier per item. Returning the array index is a common bug: on insert or delete, every subsequent key "changes" and forces a remount of every row, destroying list performance. Use entity IDs from your data, not positional ones. If entities don't have IDs, synthesize them once at ingestion and carry them through.

## getItemLayout

When provided, **`getItemLayout`** lets `FlatList` skip the measurement pass and compute offsets arithmetically. It is required for `scrollToIndex` to work reliably on not-yet-rendered items, and it substantially smooths scrolling for fixed-height rows. It is only applicable when every item is the same height — for variable heights, use FlashList or `onScrollToIndexFailed` with a manual fallback.

## FlashList Recycling Model

**FlashList** (Shopify) replaces virtualization with *recycling*: instead of unmounting off-screen items, it keeps a small pool of cell instances and re-renders them with new data. This eliminates the mount/unmount cost that dominates `FlatList` on long scrolling sessions. The price is that items must be structurally similar (the same tree shape per "type"), and component state must tolerate being re-bound to different data.

### estimatedItemSize and overrideItemLayout

**`estimatedItemSize`** seeds FlashList's recycler with how many cells to keep warm: too small and it over-allocates on the first layout pass; too large and you see blank spots while it catches up. **`overrideItemLayout`** lets you give FlashList the exact dimensions per item when you know them — critical for mixed-height content (social feeds with images, text, and reactions), where a single estimate is wrong for most rows.

### State Leakage in Recycled Cells

Because FlashList reuses mounted components, local component state that doesn't match the item shape bleeds between rows. Avoid `useState` for item-specific state without resetting it when data changes; prefer lifting state to the list level or keying with `key={item.id}`. In-flight animations are especially dangerous — a cell that started a fade-in at index 3 can be reused for index 17 mid-animation unless you cancel on data change.

## Avoiding Inline Functions in renderItem

Every inline arrow in `renderItem` is a new reference per parent render, defeating `React.memo` and forcing memoized children to re-render. Hoist `renderItem` to module scope, or wrap it in `useCallback` with stable dependencies. For per-item handlers, pass `item.id` to a stable handler rather than closing over each item.

## Item Memoization

Wrap item components in `React.memo` with the default comparator when item props are flat and handlers are stable. For complex item props, a custom comparator is rarely worth it — the comparator itself runs on every render, and an expensive comparison loses the win. Measure before adding custom comparators.

## Separator and Sticky Header Cost

**`ItemSeparatorComponent`** renders between every pair of items — multiply its render cost by the row count; this is easy to forget. **`stickyHeaderIndices`** adds a secondary render path that keeps the header mounted at the top; on Android this is implemented JS-side and can cost more than it saves for simple headers. Measure before adding either.

## Nested Virtualized Lists

RN warns against putting a `VirtualizedList` inside a `ScrollView`, because the outer `ScrollView` has infinite height — the inner list never virtualizes and renders everything. Use `FlatList` with `ListHeaderComponent`/`ListFooterComponent` instead, or reach for FlashList, which has better support for nested virtualization. Ignoring the warning is a common cause of memory blowups.

### Horizontal Inside Vertical

Carousels as rows inside a feed are common, but each horizontal list is its own virtualization boundary, and many of them compound mount cost. Keep per-row horizontal items few and lightweight, or use FlashList at both levels so cell instances can be pooled across the outer list.

## maintainVisibleContentPosition

**`maintainVisibleContentPosition`** keeps visible content anchored when items are prepended above it — essential for chat and "load more above" patterns, where naive rendering scrolls the user away from what they were reading. iOS has had native support for years; Android support landed more recently and has edge cases around fast prepends, so test with realistic insert rates.

## Inverted Lists for Chat

**`inverted`** rotates the list 180°, making index 0 the bottom. Render cost is identical, but it aligns naturally with chat UIs that add messages at the bottom. Accessibility and some gesture handlers behave unexpectedly because touch coordinates are mirrored — verify VoiceOver/TalkBack paths before shipping.

## onEndReached Pagination and Thrash

**`onEndReached`** fires near the bottom; **`onEndReachedThreshold`** is a fraction of visible length, not a pixel count. A common bug is firing the callback repeatedly while a fetch is in flight — always guard with a ref to the request state. Also note that lists with many small items trigger `onEndReached` earlier than expected, so tune the threshold relative to item density.

## Infinite Scroll Memory Pressure

Virtualization reduces *rendered* rows, not backing data. An infinitely growing array accumulates indefinitely. For very long sessions, either implement windowed data (drop items more than N screens from visible) or paginate with a visible cap. Image references inside items also keep decoded bitmaps alive — clear caches aggressively or use a disk-first loader like expo-image.

## Debugging List Performance

Enable the **Perf Monitor** (dev menu → Show Perf Monitor) to see JS and UI FPS separately — list jank is usually a JS-thread problem, visible as low JS FPS during scroll. The React DevTools Profiler shows render times per item; "Record why each component rendered" identifies the culprit prop. On Android, `adb shell dumpsys gfxinfo <pkg> framestats` gives frame-level data and Choreographer stats.

## FlatList vs FlashList Decision

Use **`FlatList`** when items have extremely variable trees, you need maximum compatibility, or you can't add a dependency. Use **FlashList** when the list is long (>50 items of similar structure), you care about scroll smoothness on low-end Android, or mount-cost spikes show up in profiling. For chat and feeds, FlashList almost always wins.

### Problem
* [The profile of one frame](https://github.com/JerryShih/doc/blob/master/off-main-painting/Profiler_DisplayList_DisplayItem_RenderLayer.png)
* Try to reduce the RenderLayer() cost at main thread.

---

### Status
* Bug 1235018 - IPC deferring mode
  * f+
* Bug 1166173 - off-main-painting
  * f?

---

### Original Paint Flow in Gecko
    PresShell::Paint()
      ClientLayerManager::BeginTransaction()
      nsLayoutUtils::PaintFrame()
        nsDisplayList::PaintRoot()
          mozilla::FrameLayerBuilder::BuildContainerLayerFor()
            ContainerState::ProcessDisplayItems()
            ...
      ClientLayerManager::EndTransaction()
        Root->RenderLayer()     // draw calls
        ClientLayerManager::ForwardTransaction()  // send layer data to parent side
        ... // clear some layers and TextureClient which are not used after transaction
        if (RepeatTransaction) {    // APZ repeated transaction
          ClientLayerManager::BeginTransaction()
          ClientLayerManager::EndTransaction()    // a recursive call here
        }

----

### Original Paint Flow in Gecko
* [Repeated transaction](https://docs.google.com/presentation/d/1cq1WtypJ6Ff8m2fr3vwlLMkS8mZIq1WZlgIlClAI9-8/edit#slide=id.p)

---

### IPC deferring mode
* Prevent the actor __delete__ call during off-main painting.
* Make sure all messages are still in order during off-main painting(e.g. MakeSnapshot()).
* [IPC deferring mode diagram](https://github.com/JerryShih/doc/blob/master/off-main-painting/ipc.md)

---

### Proposed Off-main Painting Flow
    ClientLayerManager::EndTransaction()
      if (!RepeatTransaction) {     // Try to handle APZ repeated transaction.
        WaitAllOffMainTransaction() // Only wait previous transactions for frame boundary
      }
      GetDrawTargetAsyncManager()->BeginTransaction() // Record all draw commands
      Root->RenderLayer()                             // during a off-main painting
      GetDrawTargetAsyncManager()->EndTransaction()   // transaction
      ClientLayerManager::ForwardTransaction()
        GetDrawTargetAsyncManager()->PostApplyLastTransaction() // post to thread
      ... // clear some layers and TextureClient which are not used after transaction
      if (RepeatTransaction) {
        ClientLayerManager::BeginTransaction()
        ClientLayerManager::EndTransaction()
      }

----

### The Painting Thread
    ApplyLastTransaction()
      ApplyPendingDrawCommand()
      SignalWaitableObject()

---

### AsyncPaintData
* [How to preserve DrawTarget/TextureClient for off-main painting?](https://github.com/JerryShih/doc/blob/master/off-main-painting/asyncPaintData.md)
* [The DrawTargetAsync](https://github.com/JerryShih/doc/blob/master/off-main-painting/drawTargetAsync.md)
* [Record a draw call with DrawTargetAsync](https://github.com/JerryShih/gecko-dev/blob/d803e9247ed2f9796eb6ab9c6e171903bb456f72/gfx/2d/DrawTargetAsync.cpp#L199)

---

### DrawTargetAsync from TextureClient
* [Creaet DrawTargetAsync in BorrowDrawTarget](https://github.com/JerryShih/gecko-dev/blob/c995ee9f284cc0a093d17153b12a1ba0f2ff920a/gfx/layers/client/TextureClient.cpp#L696)

---

### Discussion
* The memcopy of WrappingDataSourceSurface during recording
* There are some flush-draw-command calls at main thread
  * Snapshot(), GetNativeSurface(), Borrow[Cairo|CG]ContextFromDrawTarget() and ReadbackSink in TextureClient

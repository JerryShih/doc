### Status
* bug 1166173 off-main-painting
  * handle off-main-painting layer transaction
    * https://github.com/JerryShih/gecko-dev/commits/defer-layer-transaction-at-message-channel-parent-side
  * do the actual painting task at another thread
    * working...
* bug 1083101 multi-threaded DrawTargetTiled

----

### Impl. for Layer Transaction IPC
* Hack in MessageChannel.
* Start/EndPendingMessage()
* All messages between Start/EndPendingMessage() will be deferred and be processed until EndPendingMessage() call

``` c
/////////// Main thread
// Start message deferring
layerTransactionParent->GetIPCChannel()->StartPendingMessage();

layerTransactionParent->SendUpdateNoSwap();
// .....
// other ipc message

/////////// Painting thread
ActualDrawCommand();
// Notify the parent side that all draw calls are done.
// Start to process all pending message.
layerTransactionParent->GetIPCChannel()->
  EndPendingMessage();
    
```
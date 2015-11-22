<!--Meta author:'Jerry' theme:'night' title:IPC 101-->
<!--Meta width:1280 height:800-->
<!--Meta margin:0-->
<!--Meta minScale:0.5-->

<!--sec1-->
## IPC 101

### Jerry

<!--sec2-->
### IPC

* inter-process communication
* share data across multiple processes
* Chrome <=> Content
	* some actions requires chrome privilege

<br/><hr/>

### IPDL

* the language to generate c++ code for message passing
* IPDL file => ipdl.py => c++ ipc code

<!--sec3-->
### IPDL Code Generator
```c
                                  PTestParent.h:                            
                                  class PTestParent                         
                                  {                                         
                                  protected:                                
                                    virtual bool RecvChildToParent() = 0;   
PTest.ipdl:                         virtual bool RecvBothDirection() = 0;   
protocol PTest                    public:                                   
{                                   bool SendParentToChild();               
child:                              bool SendBothDirection();               
  ParentToChild();                };                                        
parent:                              PTestChild.h:                          
  ChildToParent();                   class PTestChlid                       
both:                                {                                      
  BothDirection();                   protected:                             
};                                     virtual bool RecvParentToChild() = 0;
                                       virtual bool RecvBothDirection() = 0;
                                     public:                                
                                       bool SendChildToParent();            
                                       bool SendBothDirection();            
                                     };                                                                  
```
1. what's actor
	* IPDL will generate two end point for message passing. This two end point are called actor.
2. message direction

<!--sec4-->
### Actor Impl

```c
TestChild.h:
#include <PTestChild.h>
class TestChild : public PTestChild
{
  ....
  virtual bool RecvParentToChild() MOZ_OVERRIDE;
  irtual bool RecvBothDirection() MOZ_OVERRIDE;
};
```
```python
moz.build:
include('/ipc/chromium/chromium-config.mozbuild')

IPDL_SOURCES = [
    'ipc/PTest.ipdl',
]
```
<!--sec5.1-->
### Message Data Type

* c++ primitive type or ipc builtin type(check ipc/ipdl/ipdl/builtin.py)
```
'int32_t'
'uint32_t'
'int64_t'
'nsresult'
'nsString'
'nsCString'
```
* struct
```c
IPDL:
struct Pos
{
  int x;
  int y;
};
protocol PTest
{
child:
  StructData(Pos pos);
};
```
```c
c++:
class PTestChild
{
  virtual bool RecvStructData(const Pos& pos) = 0;
};
```
* union

<!--sec5.2-->
### Message Data Type(Cont'd)

* array
```c
protocol PTest
{
child:
	ArrayData(int32_t[] array);
};
```
```c
c++:
class PTestChild
{ 
	virtual bool RecvArrayData(nsTArray<int32_t>&& array) = 0;
};
```
* actor
	* send an actor parent will convert it to an actor child on the other side
* file descriptor
	* currently, only supprt for posix system
	* linux: SOL_SOCKET and SCM_RIGHTS flag
	* please check ipc_channel_posix.cc
* shared memory
	* please check shared_memory_posix.cc and shared_memory_win.cc
* !!! The parameter is a temporary object, user should make a copy if needs. !!!

<!--sec6-->
### sync/async Semantic

```c
                                PTestParent.h:                                                                   
PTest.ipdl:                     class PTestParent                                                                
sync protocol PTest             {                                                                                
{                               protected:                                                                       
child:                            virtual bool RecvChildToParent(const int32_t& num, int* ans, bool* result) = 0;
  async ParentToChild();        };                                                                               
parent:                              PTestChild.h:                                                               
  sync ChildToParent(int32_t num)    class PTestChlid                                                            
    returns (int ans, bool result);  {                                                                           
both:                                public:                                                                     
  async BothDirection();               bool SendChildToParent(const int32_t& num, bool* result);                 
};                                   };                                                                          
```

* Async
	* sender is not blocked
	* IPDL use async by default
* Sync
	* sender blocks until the receiver receives the message and sends back a reply if needs
		* onlys sync call may have return values
	* sync only can used from child to parent.

<!--sec7-->
### intr Semantic

* intr represent a sync and re-entrant messages

```c
intr protocol PTest
{
child:
  intr IntrParentToChild();
parent:
  intr IntrChildToParent();
};
```
```c
eg:
CallIntrChildToParent()    ->    AnswerIntrChildToParent()
                                       |
                                       |
AnswerIntrParentToChild()  <-    CallIntrParentToChild()
```

* !!! They're basically used for plugins. Just pretend that they don't exist. !!!

<!--sec8-->
### priority Semantic

* prio(normal), prio(high) and prio(urgent)
* check MessageChannel.cpp for detail
```c
prio(normal upto high) protocol PTest
{
child:
  prio(high) async AsyncHighParentToChild();
parent:
  sync SyncNormalChildToParent();
};
```
```c
eg:
SyncNormalChildToParent()     ->    RecvNormalChildToParent()
                                           |
                                           |
RecvAsyncHighParentToChild()  <-    SendAsyncHighParentToChild()
```

* !!! This is incredibly dangerous. You should make sure the sync-calling code is completely unrelated to the incoming high priority message. !!!

<!--sec9-->
### compress Semantic

* remove the redundant message
* could be used with async message

```c
PBrowser.ipdl
prio(normal upto urgent) intr protocol PBrowser
{
  UpdateDimensions(...) compress;
};
```

* please check MessageChannel::OnMessageReceivedFromLink and Bug 1076820

<!--sec10.1-->
### Subprotocol

```c
PTest.ipdl                                                               
include protocol PTestSubtree;            PTestSubtree.ipdl              
sync protocol PTest                       include protocol PTest;        
{                                         protocol PTestSubtree          
  manages PTestSubtree;                   {                              
child:                                      manager PTest;               
  async ParentToChild();                  child:                         
parent:                                     //delete from parent to child
  //create from child to parent.            __delete__();                
  sync PTestSubtree(int32_t data)           ParentToChild();             
    returns (bool result);                };                             
};                                                                       
```
```c
PTestParent.h
class PTestParent
{
  virtual bool RecvPTestSubtreeConstructor(PTestSubtreeParent* actor, const int32_t& data, bool* result);
  virtual void ActorDestroy(ActorDestroyReason why) MOZ_OVERRIDE;
  virtual PTestSubtreeParent* AllocPTestSubtreeParent(const int32_t& data, bool* result) = 0;
  virtual bool DeallocPTestSubtreeParent(PTestSubtreeParent* aActor) = 0;
};
```
```c
PTestChild.h
class PTestChild
{
	PTestSubtreeChild* SendPTestSubtreeConstructor(onst int32_t& data, bool* result);
	virtual void ActorDestroy(ActorDestroyReason why) = 0;
  virtual PTestSubtreeChild* AllocPTestSubtreeChild(const int32_t& data, bool* result) = 0;
  virtual bool DeallocPTestSubtreeChild(PTestSubtreeChild* aActor) = 0;
};
```

<!--sec10.2-->
### Subprotocol(Cont'd)

```c
PTest.ipdl                                                               
include protocol PTestSubtree;            PTestSubtree.ipdl              
sync protocol PTest                       include protocol PTest;        
{                                         protocol PTestSubtree          
  manages PTestSubtree;                   {                              
child:                                      manager PTest;               
  async ParentToChild();                  child:                         
parent:                                     //delete from parent to child
  //create from child to parent.            __delete__();                
  sync PTestSubtree(int32_t data)           ParentToChild();             
    returns (bool result);                };                             
};                                                                       
```
```c
PTestSubtreeChild.h
class PTestSubtreeChild
{
    virtual bool Recv__delete__();
    virtual void ActorDestroy(ActorDestroyReason why) = 0;
};
```
```c
PTestSubtreeParent.h
class PTestSubtreeParent
{
	static bool Send__delete__(PTestSubtreeParent* actor);
	virtual void ActorDestroy(ActorDestroyReason why) = 0;
};
```

<!--sec11.1-->
### Actor Lifetime

* create actor
	* top-level protocol PTest
		* call PTestParent::Open()
	* subprotocol PTestSubtree
  	* call PTestChild::SendPTestSubtreeConstructor()
* the corresponding AllocPTest* will be called

<!--sec11.2-->
### Actor Lifetime(Cont'd)

* destroy actor
	* Top-level protocol PTest
	* call PTestParent::Close() to close the channel.
  	* PTestParent::DestroySubtree()
  		* PTestParent::ActorDestroy()
  	* PTestParent::DeallocSubtree()
	* subprotocol PTestSubtree
  	* parent:
  		* PTestSubtreeParent::Send\_\_delete\_\_()
  	* child:
    	* PTestSubtreeChild::Recv\_\_delete\_\_()
      	* PTestSubtreeChild::DestroySubtree()
      		* PTestSubtreeChild::ActorDestroy()
      	* unregister itself from its manager

<!--sec12-->
### Thread Model

```c
       Chrome                                              Content                  
                                          +                                         
                                          |                                         
                                          |                                         
+---+--+----+         +-----------+       |       +-----------+        +---+--+----+
|Top-level  |         |    IO     |       |       |    IO     |        |Top-level  |
|Protocol   | <-----> |MessageLoop| <-----+-----> |MessageLoop| <----> |Protocol   |
|MessageLoop|         +-----------+     unix      +-----------+        |MessageLoop|
+-----------+                          domain                          +-----------+
                         ^             socket             ^                         
+---+--+----+            |                +               |            +---+--+----+
|Top-Level  |            |                |               |            |Top-level  |
|Protocol   | <----------+                +               +----------> |Protocol   |
|MessageLoop|                                                          |MessageLoop|
+-----------+                                                          +-----------+
```

* top-level protocol and its subprotocol will run on the thread which call Open()
* each top-level protocol has its own IPC channel.
* please check MessageChannel::Open()

<!--sec13-->
### Debug IPC

* build debug version
* export environment variable "MOZ_IPC_MESSAGE_LOG=1" or "MOZ_IPC_MESSAGE_LOG=$TOP_LEVEL_PROTOCOL_NAME"

<!--sec14-->
### QA???
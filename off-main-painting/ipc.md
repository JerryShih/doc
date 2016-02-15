### IPC async message
                      start        push M1        push M2                 end pending,
                      pending      to queue       to queue                start to process M1 and M2
    compositor +---------+---------+--------------+------------------------+------------------------------->
    thread               ^         ^              ^                        ^
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
                         |         |              |                        |
    painting   +-----------------------------------------+-----------------+------------------------------->
    thread               |         |              |      ^               Channel->EndPendingMessage()
                         |         |              |      |
                         |         |              |      | FlushDrawCommand
                         |         |              |      |
                         |         |              |      |
    main       +--+----------------+--------------+------+-----+------------------------------------------->
    thread        ^      ^   protocol.SendM1()                 ^
                  |      |                 protocol.SendM2()   |
                  |      |                                     |
                  +      +                                     +
             BeginFrame Channel->StartPendingMessage()       EndFrame

----

### IPC sync message
                      start                         push M1      end pending,
                      pending                       to queue     start to process M1
    compositor +---------+--------------------------+-------------------+---------------------+------------>
    thread               ^                          ^                   ^                     |
                         |                          |                   |                     |
                         |                          |                   |                     |Reply M1
                         |                          |                   |                     |
                         |                          |                   |                     |
                         |                          |                   |                     |
                         |                          |                   |                     |
                         |                          |                   |                     |
                         |                          |                   |                     |
    painting   +---------------+----------------------------------------+---------------------------------->
    thread               |     ^                    |          Channel->EndPendingMessage()   |
                         |     |                    |Wait for M1                              |
                         |     |FlushDrawCommand    |                                         |
                         |     |                    |     +-----------------------------------+
                         |     |                    |     |
    main       +--+------------+--------------------+-----v-------------------------------------+---------->
    thread        ^      ^              protocol.SendSyncM1()                                   ^
                  |      |                                                                      |
                  |      |                                                                      |
                  +      +                                                                      +
             BeginFrame Channel->StartPendingMessage()                                        EndFrame
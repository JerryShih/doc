### AsyncPaintData and DrawTargetAsyncManager

                             Register
                     +---------------------+
                     |                     |
                     |                     |
                     |                     |
                     +                     |                DrawTargetAsync
               AsyncPaintData              |                       |
                     ^                     v                       |
                     |             DrawTargetAsyncManager          |
           +---------+                                             |
           |         |                                       AsyncPaintData
           |         +----------+
           |                    |
DrawTargetAsyncPaintData        |
           |                    |
           |       TextureClientAsyncPaintData
           |                    |
      DrawTarget                |
                                |
                           TextureClient

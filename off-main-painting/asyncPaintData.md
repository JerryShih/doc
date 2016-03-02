### AsyncPaintData and DrawTargetAsyncManager
                                Register
                         +---------------------+
                         |                     |
                         |                     |
                         |                     |
                         +                     |
                   AsyncPaintData              |
                         ^                     v
                         |             DrawTargetAsyncManager
               +---------+
               |         |
               |         +----------+
               |                    |
    DrawTargetAsyncPaintData        |
               |                    |
               |       TextureClientAsyncPaintData
               |                    |
          DrawTarget                |
                                    |
                               TextureClient

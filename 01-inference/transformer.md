                  Request
                     │
                     ▼
                  Tokenize
                     │
                     ▼
                ┌─────────┐
                │ Prefill │
                └────┬────┘
                     │
                     ▼
                KV Cache
                     │
                     ▼
                ┌─────────┐
                │ Decode  │
                └────┬────┘
                     │
             Token 1 │
             Token 2 │
             Token 3 │
                  ...│
                     ▼
                  Response



                 Requests
                     │
                     ▼
          ┌──────────────────┐
          │ Continuous Batch │
          └────────┬─────────┘
                   │
                   ▼
             Inference Engine
                   │
             ┌─────┴─────┐
             ▼           ▼
          Prefill      Decode
             │           │
             └─────┬─────┘
                   ▼
               KV Cache

---
epic: board
refinement: pristine
slices:
  - name: local
    status: delivered
    plan: ./local/index.md
  - name: remote
    status: delivered
    plan: ./remote/index.md
  - name: chunk-detail
    status: delivered
    plan: ./chunk-detail/index.md
  - name: mobile
    status: delivered
---

# Plan — `epic:board`

The board's slice plans, each frozen as the slice shipped, and the mockups they were drawn from. The mobile slice landed
without a plan of its own; [delivered.md](../../delivered.md) records every slice.

| Where                                    | Read when                                                                                  |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| [local/](./local/index.md)               | Looking up what the board's first slice — the hub-served web app — was conceived to be     |
| [remote/](./remote/index.md)             | Looking up the board's provider login, roles, and runner single sign-on                    |
| [chunk-detail/](./chunk-detail/index.md) | Looking up the routed chunk detail page and its shared artifact renderer                   |
| [artifacts/](./artifacts/)               | Seeing the mockups behind any slice — the founding concept round and the chunk detail page |

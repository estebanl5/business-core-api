# Query Handler Prompt
Generate a new **MediatR Query + Handler** for {Entity}.
- Use EF Core with AsNoTracking and proper filtering.
- Map to DTOs in Application layer.
- Add xUnit tests with an in-memory DbContext (Testcontainers or EF Core InMemory).

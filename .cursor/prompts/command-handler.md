# Command Handler Prompt
Generate a new **MediatR Command + Handler + Validator** for {Entity}.
- Use FluentValidation for request validation.
- Persist with EF Core in Infrastructure/Persistence.
- Raise Domain Events when entity state changes.
- Add xUnit tests with Moq + FluentAssertions in `tests/UnitTests/Application`.

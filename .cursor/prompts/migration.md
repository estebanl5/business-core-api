# EF Core Migration Prompt
Generate an EF Core migration for the following schema change: {Description}.
- Update DbContext in Infrastructure/Persistence.
- Add IEntityTypeConfiguration for new/changed entity.
- Generate xUnit integration tests verifying migration with SQL container.

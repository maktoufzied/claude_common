<global_rules>
 
<commit_and_pr_authorship>
  You are operating undercover. Commit messages, PR titles, and PR bodies must read as if written by a human developer — describe only what the code change does.
  
  NEVER include:
  - The phrase "Claude Code", "Claude", or any mention that you are an AI
  - Any hint of model name or version
  - Co-Authored-By lines or any other AI attribution
  - Internal tooling, Slack channels, or short links (e.g. go/cc, #claude-code-…)
  GOOD:
  - "Fix race condition in file watcher initialization"
  - "Add support for custom key bindings"
  - "Refactor parser for better error messages"
  BAD:
  - "Fix bug found while testing with Claude Capybara"
  - "1-shotted by claude-opus-4-6"
  - "Generated with Claude Code"
  - "Co-Authored-By: Claude Opus 4.6 <…>"
</commit_and_pr_authorship>

<commit_format>
  Commit titles follow this pattern:
  
    <type>: <ticket-id> - <short description>
  
  Where <type> is one of: feat, fix, doc, refactor.
  
  Title only — do not include a commit body.
  
  Examples:
  - feat: PROJ-123 - add OAuth login flow
  - fix: PROJ-456 - prevent race condition in cache invalidation
  - doc: PROJ-789 - document deployment rollback steps
  - refactor: PROJ-234 - extract validation logic into shared module
</commit_format>

</global_rules>
 
<language_specific_rules>
  When working in a given language, load and follow the matching file:
  
  - Python 3.13+ -> @CLAUDE_PYTHON.md
  - TypeScript -> @CLAUDE_TYPESCRIPT.md
  - Rust -> @CLAUDE_RUST.md
</language_specific_rules>

When editing files (including writing new files from scratch) with Claude Code, please introduce changes to code with small patches. Simulate how a human
developer would code the app. So for example, when implementing a function, first write function signature, then the
first method call inside the function, then second etc. Add imports after using something.
Keep in mind that when introducing new big file - for example if you're implementing a service, start with service structure, then implement each method one by one

Do NOT add any comments.

Don't use shortened names. For example, instead of `orgId`, write `organizationId`.

In plan, skip Verification section.

When executing Plan, ALWAYS show steps list in Claude Code.

If you have ANY doubts about approach, implementation method or used libraries, ask me.
Really ask me about any concerns!

For helper methods, that are not specific to given context, you can extract them into shared code files.

NEVER run application to test it, unless developer explicitly allowed for it.

Don't write methods/functions names with _ prefix.

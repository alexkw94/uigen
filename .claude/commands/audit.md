Here's a practical example of a custom command that audits project dependencies for vulnerabilities:

This audit command does three things:

1. Runs npm audit to find vulnerable installed packages
2. Runs npm audit fix to apply updates
3. Runs tests to verify the updates didn't break anything
After creating your command file, you must restart Claude Code for it to recognize the new command.

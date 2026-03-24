# claude-setup
A snapshot of my Claude Code setup. 

Some skills are specific to my projects; however, the following are the highlights that are worth copying.

Some notes:
- Skills are composable, so you'll notice that some of the larger skills like /send or /worktree actually call the smaller skills like address PR feedback or lint within them.
- This is nice because you can just use the smaller skills as you need, or Claude will use them as it needs them ad hoc. When it's called in a larger skill, it's just like little pieces of a recipe.
- In general, these were not made all in one shot, and I don't actually recommend you copying all of these in one shot. I think it's best to just add little bits and pieces as you go, where you're like, "Oh, I do this manual, repetitive thing over and over" just ask Claude to make a skill. I would say like 50% of my PRs have a small tweak or a new skill added to the .claude folder. I don't try to make separate PRs just for claude stuff. Just roll it in as you go.

## /address-pr-feedback
Pulls in all feedback from a PR, summarises them and addresses each one, tells you if it's been addressed or not.

## /lint
Claude uses this to run my pre-commit file after changes have been made so that it finds linting issues before actually committing. So this would run Ruff and ty on my Python backend and then biome on my TypeScript frontend.

## /rebase
With many parallel work trees, it's nice to be able to rebase the current one onto the latest changes in main. This skill is great because it automatically rebases, figures out merge conflicts, and resolves them.

## /review-pr
This is a skill that's used with the GitHub workflows. It was just nice to have the prompt for the PR review in a single spot rather than in the GitHub workflow file.

## /send
I use this to literally "send" small issues or bugs, where it will figure out what happened, do all the work, commit, push, open a PR, wait for the Claude code workflow to review it, address the PR feedback, and then push that. When I go look at the PR, it's already gone through a review loop.

## /worktree
This is my most used skill. I'll open up a new terminal tab and then use this skill to launch a worktree and kick off the /send skill. I'll either Just put the linear ticket number I want to execute or I'll describe the bug or small feature I wanted to implement and add a screenshot. It'll go read the linear ticket and just implement it. Or I'll describe a bug or put a screenshot and it'll spin up a worktree and send it.

## /title
I run Claude code in a CLI in a terminal called [cmux](https://cmux.com/). I just use this to name the tabs within Cmux. That way I can just keep track of all my work trees. Also recommend [ghostty](https://ghostty.org/). 
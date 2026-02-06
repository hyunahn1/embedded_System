# 6.1 Version Control & Collaboration

## Topics Covered

### Git Flow / Trunk-Based Development

#### Git Basics Review
```bash
# Initialize repository
git init

# Add files
git add file.c
git add .

# Commit changes
git commit -m "Add feature X"

# View history
git log --oneline --graph --all

# Create branch
git branch feature/new-sensor

# Switch branch
git checkout feature/new-sensor
# or
git switch feature/new-sensor

# Merge branch
git checkout main
git merge feature/new-sensor

# Push to remote
git push origin main

# Pull from remote
git pull origin main
```

#### Git Flow

##### Branch Structure
```
main (production-ready)
├── develop (integration branch)
│   ├── feature/sensor-driver
│   ├── feature/uart-protocol
│   └── feature/power-management
├── release/v1.0
│   └── hotfix/critical-bug
└── hotfix/security-patch
```

##### Branch Types
- **main**: Production-ready code
- **develop**: Integration branch for features
- **feature/**: New features (branch from develop)
- **release/**: Release preparation (branch from develop)
- **hotfix/**: Urgent fixes (branch from main)

##### Workflow
```bash
# Start new feature
git checkout develop
git checkout -b feature/sensor-driver

# Work on feature
git add sensor.c
git commit -m "Implement sensor driver"

# Finish feature
git checkout develop
git merge --no-ff feature/sensor-driver
git branch -d feature/sensor-driver
git push origin develop

# Create release
git checkout -b release/v1.0 develop
# ... version bump, final testing ...
git checkout main
git merge --no-ff release/v1.0
git tag -a v1.0 -m "Release version 1.0"
git push --tags

# Hotfix
git checkout -b hotfix/critical-bug main
# ... fix bug ...
git checkout main
git merge --no-ff hotfix/critical-bug
git checkout develop
git merge --no-ff hotfix/critical-bug
```

#### Trunk-Based Development

##### Concept
- **Single Main Branch**: All work on main (or master)
- **Short-Lived Branches**: Feature branches live <1 day
- **Frequent Integration**: Multiple commits per day
- **Feature Flags**: Hide incomplete features

##### Workflow
```bash
# Create short-lived feature branch
git checkout -b add-sensor-support main

# Make small commits
git add sensor.c
git commit -m "Add sensor initialization"

# Rebase on main (keep history clean)
git fetch origin
git rebase origin/main

# Push and create PR
git push origin add-sensor-support

# After review, merge (squash or merge commit)
# Delete branch immediately
git branch -d add-sensor-support
```

##### Advantages
- Simpler workflow
- Faster integration
- Fewer merge conflicts
- Better for CI/CD

#### Git Best Practices for Embedded

##### Commit Messages
```
[Component] Short summary (50 chars or less)

More detailed explanation if needed. Wrap at 72 characters.

- Bullet points for multiple changes
- Reference issue tracker: Fixes #123

Signed-off-by: Your Name <your.email@example.com>
```

**Good Examples:**
```
[HAL] Add SPI driver for sensor communication

Implement SPI master mode with DMA support for efficient
sensor data acquisition. Includes:
- SPI initialization and configuration
- DMA setup for continuous data transfer
- Error handling and timeout mechanism

Fixes #456
```

##### .gitignore for Embedded
```gitignore
# Build artifacts
*.o
*.obj
*.elf
*.bin
*.hex
*.map
*.lst
*.d

# Directories
build/
output/
Debug/
Release/

# IDE files
.vscode/
.idea/
*.code-workspace

# OS files
.DS_Store
Thumbs.db

# But keep linker scripts and config
!*.ld
!sdkconfig
```

##### Binary Files
```bash
# Use Git LFS for large files
git lfs install
git lfs track "*.bin"
git lfs track "*.hex"
git add .gitattributes
```

### Gerrit (Code Review)

#### Overview
- **Purpose**: Web-based code review tool
- **Integration**: Works with Git
- **Features**: Line-by-line comments, voting, continuous integration
- **Workflow**: Submit change → Review → Approve → Merge

#### Architecture
```
Developer → Git Push → Gerrit → Code Review → CI Tests → Merge
```

#### Change-ID
- **Unique Identifier**: Each commit has a Change-ID
- **Amendment**: Can update same change with new patch sets

```bash
# Install commit-msg hook (adds Change-ID automatically)
curl -Lo .git/hooks/commit-msg \
    http://gerrit-server/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg
```

#### Workflow
```bash
# Configure Gerrit remote
git remote add gerrit ssh://user@gerrit-server:29418/project

# Make changes
git checkout -b feature/new-sensor
git add sensor.c
git commit -m "[Sensor] Add temperature sensor driver"

# Push to Gerrit (refs/for/main creates a review)
git push gerrit HEAD:refs/for/main

# After review comments, amend commit
git add sensor.c
git commit --amend
git push gerrit HEAD:refs/for/main  # Same Change-ID, new patch set
```

#### Review Process
1. **Submit Change**: Push to Gerrit
2. **Automated Checks**: CI builds and tests
3. **Code Review**: Reviewers comment
4. **Voting**:
   - Code-Review: -2 to +2 (-2 blocks)
   - Verified: -1 to +1 (CI result)
5. **Submit**: Approved changes merged

#### Review Commands
```bash
# Fetch change from Gerrit
git fetch gerrit refs/changes/34/1234/2  # Change 1234, patch set 2
git checkout FETCH_HEAD

# Cherry-pick change
git fetch gerrit refs/changes/34/1234/2
git cherry-pick FETCH_HEAD

# Rebase change
git fetch origin main
git rebase origin/main
git push gerrit HEAD:refs/for/main
```

#### Gerrit Best Practices

##### One Logical Change per Commit
```
Bad:  Fix bug + add feature + refactor
Good: Three separate commits
```

##### Respond to Comments
- Address all review comments
- Reply to clarify or explain
- Mark comments as "Done" after addressing

##### Keep Patch Sets Clean
```bash
# Rebase before submitting new patch set
git fetch origin
git rebase origin/main
git push gerrit HEAD:refs/for/main
```

### Branching and Merging Strategies

#### Merge vs Rebase vs Squash

##### Merge (--no-ff)
```bash
git merge --no-ff feature/sensor
```
- **Result**: Merge commit preserves branch history
- **Pros**: Complete history
- **Cons**: Cluttered history with many branches

##### Rebase
```bash
git rebase main
```
- **Result**: Linear history, no merge commit
- **Pros**: Clean, linear history
- **Cons**: Rewrites history (don't rebase published commits)

##### Squash
```bash
git merge --squash feature/sensor
git commit -m "Add sensor driver"
```
- **Result**: All feature commits → single commit
- **Pros**: Clean history, one commit per feature
- **Cons**: Loses intermediate commit details

#### Merge Conflicts
```bash
# During merge/rebase
git status  # See conflicted files

# Edit files to resolve conflicts
# Look for conflict markers:
<<<<<<< HEAD
current code
=======
incoming code
>>>>>>> feature-branch

# After resolving
git add resolved_file.c
git commit  # or git rebase --continue
```

#### Advanced Git Techniques

##### Interactive Rebase
```bash
# Rewrite last 3 commits
git rebase -i HEAD~3

# Options in editor:
# pick   = use commit
# reword = use commit, edit message
# edit   = use commit, stop for amending
# squash = combine with previous
# fixup  = like squash, discard message
# drop   = remove commit
```

##### Stash
```bash
# Save work in progress
git stash

# List stashes
git stash list

# Apply stash
git stash apply
git stash pop  # Apply and remove

# Stash with message
git stash save "WIP: sensor driver"
```

##### Bisect (Find Bug Introduction)
```bash
# Start bisect
git bisect start
git bisect bad          # Current commit is bad
git bisect good v1.0    # v1.0 was good

# Git checks out middle commit
# Test the commit
git bisect good  # or git bisect bad

# Repeat until found
git bisect reset  # End bisect
```

### Collaboration Best Practices

#### Pull Request (PR) / Merge Request (MR)

##### PR Description Template
```markdown
## Summary
Brief description of changes

## Motivation
Why is this change necessary?

## Changes
- Bullet point 1
- Bullet point 2

## Testing
How was this tested?

## Checklist
- [ ] Code builds without errors
- [ ] Unit tests pass
- [ ] Code follows style guide
- [ ] Documentation updated
- [ ] Tested on hardware
```

##### Code Review Guidelines

**As Author:**
- Keep PRs small (<400 lines)
- Write clear descriptions
- Respond promptly to feedback
- Don't take criticism personally

**As Reviewer:**
- Review promptly (within 24 hours)
- Be constructive and respectful
- Focus on code, not person
- Ask questions, don't demand
- Approve when ready

##### Review Checklist
```
□ Correctness: Does code work as intended?
□ Design: Is architecture sound?
□ Complexity: Is code simple and readable?
□ Tests: Are there adequate tests?
□ Naming: Are names clear and consistent?
□ Comments: Are comments helpful?
□ Style: Does code follow style guide?
□ Documentation: Is documentation updated?
```

#### Git Hooks

##### Pre-commit Hook
```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run static analysis
cppcheck --enable=all --error-exitcode=1 src/

# Run code formatter check
clang-format --dry-run --Werror src/*.c

# Run tests
make test
```

##### Commit-msg Hook
```bash
#!/bin/sh
# .git/hooks/commit-msg

# Check commit message format
commit_msg=$(cat "$1")

if ! echo "$commit_msg" | grep -qE '^\[[A-Z]+\]'; then
    echo "Error: Commit message must start with [COMPONENT]"
    exit 1
fi
```

## Key Concepts

- **Atomic Commits**: One logical change per commit
- **Linear History**: Easier to understand and bisect
- **Code Review**: Essential for quality
- **Branch Protection**: Require reviews before merging

## Practical Exercises

1. Set up Git Flow for a project
2. Practice rebasing feature branch on main
3. Resolve merge conflicts
4. Perform interactive rebase to clean history
5. Submit change to Gerrit for review
6. Write comprehensive PR description
7. Create Git hooks for quality checks

## Tools

- **Git GUI**: GitKraken, SourceTree, Git Extensions
- **GitHub/GitLab/Bitbucket**: Hosting and collaboration
- **Gerrit**: Enterprise code review
- **Pre-commit**: Framework for git hooks

## Resources

- Pro Git book (free online)
- Atlassian Git tutorials
- GitHub Flow guide
- Gerrit documentation

# GitHub Configuration Guide

This directory contains GitHub-specific configuration files for the romanchaa997 ecosystem.

## Files Overview

### CODEOWNERS
**Purpose**: Automatic code owner assignment for pull requests  
**Features**:
- Specifies @romanchaa997 as the default owner for all changes
- Requires review for critical files: security policies, documentation, workflows
- Automatically requests reviews based on file changes

**Usage**: GitHub automatically notifies code owners when PRs affect their files

### dependabot.yml
**Purpose**: Automated dependency management and security updates  
**Configuration**:
- **npm**: Weekly updates on Mondays at 03:00 UTC
- **pip**: Weekly updates on Mondays at 03:00 UTC  
- **github-actions**: Weekly updates on Mondays at 03:00 UTC

**Features**:
- Automatic vulnerability scanning
- Creates pull requests with dependency updates
- Assigns to @romanchaa997 for review
- Labels updates for easy filtering
- Uses conventional commit messages

**Important**: Ensure you have Dependabot enabled in repository Settings > Security > Dependabot

### pull_request_template.md
**Purpose**: Standardize pull request submissions  
**Sections**:
1. **Description**: Summary of changes
2. **Type of Change**: Bug fix, feature, security improvement, etc.
3. **Related Issues**: Link to issue tracker
4. **Testing**: How changes were tested
5. **Checklist**: Verification steps including:
   - GPG commit signing
   - Test coverage
   - Documentation updates
   - Security review
   - Performance impact

**Usage**: Template auto-populates when creating new PRs

## GitHub Settings Applied

### Branch Protection (main branch)
✅ Require pull request before merging  
✅ Require status checks to pass  
✅ Require conversation resolution  
✅ Require signed commits  
✅ Require linear history  
✅ Prevent forced pushes

### Repository Settings
✅ Require contributors to sign off on web commits  
✅ Release immutability enabled  
✅ Public visibility with security policies  

## Recommended Next Steps

### 1. Enable GitHub Actions
```yaml
# Create workflow files in .github/workflows/
# Example: ci.yml, security.yml, deploy.yml
```

### 2. Configure Dependabot Dashboard
- Go to Security tab
- Review and merge Dependabot PRs regularly
- Configure PR auto-merge if desired

### 3. Set Up Status Checks
- Add required CI/CD checks from GitHub Actions
- Configure code quality gates
- Set up automated testing

### 4. Enable Code Scanning
- GitHub Advanced Security features
- SNYK integration
- CodeQL analysis

## Commit Signing

All commits require GPG signatures. Setup:

```bash
# Generate GPG key
gpg --full-generate-key

# Configure Git
git config user.signingkey <KEY_ID>
git config commit.gpgsign true

# Sign commits
git commit -S -m "feat: add feature"
```

## Pull Request Workflow

1. **Create Feature Branch**
   ```bash
   git checkout -b feature/your-feature
   ```

2. **Make Changes & Sign Commits**
   ```bash
   git commit -S -m "type: description"
   ```

3. **Push & Create PR**
   ```bash
   git push origin feature/your-feature
   ```

4. **Complete PR Template**
   - Fill all sections
   - Link related issues
   - Request reviews

5. **Address Feedback**
   - Code reviews from @romanchaa997
   - Resolve conversations
   - Update based on comments

6. **Merge**
   - All checks must pass
   - Linear history maintained
   - Signed commits required

## Labels & Automation

Auto-applied labels from Dependabot:
- `dependencies` - Dependency update
- `javascript` / `python` - Ecosystem type
- `github-actions` - Workflow updates

Manual labels for organization:
- `bug` - Bug report or fix
- `enhancement` - New feature
- `security` - Security-related
- `documentation` - Docs update

## Troubleshooting

### Dependabot PRs not appearing
- Enable Dependabot in Settings > Security
- Check organization permissions
- Verify repository visibility

### Signed commits failing
- Ensure GPG key is configured
- Check git config settings
- Verify GPG is installed

### Branch protection violations
- All commits must be signed
- Status checks must pass
- Conversations must be resolved

## Resources

- [GitHub CODEOWNERS Docs](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
- [GPG Commit Signing](https://docs.github.com/en/authentication/managing-commit-signature-verification)
- [Branch Protection Rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches)
